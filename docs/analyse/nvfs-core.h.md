# nvfs-core.c/h 文件解读

## 1. 文件功能描述

`nvfs-core.c/h` 是 NVIDIA GPUDirect Storage (GDS) 内核模块的 **核心实现文件**，定义了驱动的主要 IOCTL 接口、数据结构、I/O 操作和模块初始化逻辑。该文件是整个 GDS 驱动的核心组件，实现了用户空间与内核空间的通信接口以及 GPU 直接存储访问的核心逻辑。

### 主要功能：
- **IOCTL 接口**：定义用户空间与内核空间通信的命令和数据结构
- **设备管理**：字符设备注册、设备创建和销毁
- **I/O 操作**：同步/异步读写操作、批量 I/O 支持
- **GPU 内存管理**：GPU 页面固定、DMA 映射、回调处理
- **模块生命周期**：模块初始化、卸载和参数管理

---

## 2. 核心数据结构说明

### 2.1 `struct nvfs_ioctl_map_s` - GPU 缓冲区映射请求

```c
struct nvfs_ioctl_map_s {
    s64    size;              // GPU 缓冲区大小
    u64    pdevinfo;          // PCI 设备信息（domain/bus/device/function）
    u64    cpuvaddr;          // Shadow Buffer 地址
    u64    gpuvaddr;          // GPU 缓冲区虚拟地址
    u64    end_fence_addr;    // 完成栅栏地址（用于异步 I/O 通知）
    u32    sbuf_block;        // Shadow Buffer 块数量（4KB 块）
    u16    is_bounce_buffer;  // 是否为弹跳缓冲区标志
    u8     padding[2];        // 填充字节
} __attribute__((packed, aligned(8)));
```

**字段说明：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `size` | `s64` | GPU 缓冲区大小（字节），必须大于 0 |
| `pdevinfo` | `u64` | PCI 设备信息，编码格式：`[domain:32][bus:8][device:5][function:3]` |
| `cpuvaddr` | `u64` | Shadow Buffer 的 CPU 虚拟地址 |
| `gpuvaddr` | `u64` | GPU 缓冲区的虚拟地址 |
| `end_fence_addr` | `u64` | 完成栅栏地址，GPU 通过写入该地址通知 CPU I/O 完成 |
| `sbuf_block` | `u32` | Shadow Buffer 的 4KB 块数量 |
| `is_bounce_buffer` | `u16` | 标记是否为弹跳缓冲区（用于兼容性回退） |

---

### 2.2 `struct nvfs_ioctl_ioargs` - I/O 操作参数

```c
struct nvfs_ioctl_ioargs {
    u64 cpuvaddr;             // Shadow Buffer 地址
    loff_t offset;            // 文件偏移量
    u64 size;                 // I/O 大小
    u64 end_fence_value;      // 栅栏完成值
    s64  ioctl_return;        // 系统调用返回值
    nvfs_file_args_t file_args;  // 文件参数
    int fd;                   // 文件描述符
    uint8_t sync:1;           // 同步 I/O 标志
    uint8_t hipri:1;          // 高优先级 I/O 标志
    uint8_t allowreads:1;     // 允许读操作标志
    uint8_t use_rkeys:1;      // 使用 RDMA 远程密钥标志
    uint8_t optype:3;         // 操作类型（READ:0 | WRITE:1）
    uint8_t reserved:1;       // 保留位
    u8 padding[3];            // 填充字节
} __attribute__((packed, aligned(8)));
```

**字段说明：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `cpuvaddr` | `u64` | 已映射的 Shadow Buffer 地址 |
| `offset` | `loff_t` | 文件读写偏移量 |
| `size` | `u64` | I/O 操作的数据大小 |
| `end_fence_value` | `u64` | 栅栏完成值，异步 I/O 时 GPU 写入该值表示完成 |
| `ioctl_return` | `s64` | 系统调用返回值，负值表示错误 |
| `file_args` | `nvfs_file_args_t` | 文件参数（inode 号、设备号等） |
| `fd` | `int` | 文件描述符 |
| `sync` | `bitfield` | 同步 I/O 标志，1 表示阻塞直到完成 |
| `hipri` | `bitfield` | 高优先级 I/O 标志 |
| `allowreads` | `bitfield` | 允许在 O_WRONLY 模式下读取 |
| `use_rkeys` | `bitfield` | 使用 RDMA 远程密钥标志 |
| `optype` | `bitfield` | 操作类型：READ=0, WRITE=1 |

---

### 2.3 `struct nvfs_file_args` - 文件参数

```c
struct nvfs_file_args {
    ino_t inum;               // inode 号
    u32 generation;           // inode 版本号（用于缓存验证）
    u32 majdev;               // 设备主设备号
    u32 mindev;               // 设备次设备号
    u64 devptroff;            // 设备缓冲区偏移量
} __attribute__((packed, aligned(8)));
```

**字段说明：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `inum` | `ino_t` | 文件的 inode 号 |
| `generation` | `u32` | inode 版本号，用于检测文件是否被替换 |
| `majdev` | `u32` | 设备主设备号 |
| `mindev` | `u32` | 设备次设备号 |
| `devptroff` | `u64` | 设备缓冲区偏移量 |

---

### 2.4 `struct nvfs_ioctl_batch_ioargs` - 批量 I/O 参数

```c
#ifdef NVFS_BATCH_SUPPORT
struct nvfs_ioctl_batch_ioargs {
    uint64_t ctx_id;          // 上下文标识符
    uint64_t nents;           // I/O 条目数量
    nvfs_ioctl_ioargs_t *io_entries;  // I/O 条目数组指针
} __attribute__((packed, aligned(8)));
#endif
```

---

### 2.5 `struct nvfs_ioctl_metapage` - 元数据页

```c
struct nvfs_ioctl_metapage {
    volatile u64 end_fence_val;         // 完成栅栏值
    volatile u64 result;                // I/O 结果
    volatile nvfs_metastate_enum state; // I/O 状态
    struct nvfs_io_sparse_data sparse_data;  // 稀疏文件数据
};
```

**状态枚举值：**

| 状态 | 说明 |
|------|------|
| `NVFS_IO_META_CLEAN` | I/O 状态正常 |
| `NVFS_IO_META_SPARSE` | 遇到稀疏文件空洞 |
| `NVFS_IO_META_DIED` | I/O 被终止 |

---

## 3. IOCTL 命令定义

| 命令 | 值 | 说明 |
|------|-----|------|
| `NVFS_IOCTL_REMOVE` | `_IOW('t', 1, int)` | 移除映射 |
| `NVFS_IOCTL_READ` | `_IOW('t', 2, int)` | 执行读操作 |
| `NVFS_IOCTL_MAP` | `_IOW('t', 3, int)` | 映射 GPU 缓冲区 |
| `NVFS_IOCTL_WRITE` | `_IOW('t', 4, int)` | 执行写操作 |
| `NVFS_IOCTL_SET_RDMA_REG_INFO` | `_IOW('t', 5, int)` | 设置 RDMA 注册信息 |
| `NVFS_IOCTL_GET_RDMA_REG_INFO` | `_IOW('t', 6, int)` | 获取 RDMA 注册信息 |
| `NVFS_IOCTL_CLEAR_RDMA_REG_INFO` | `_IOW('t', 7, int)` | 清除 RDMA 注册信息 |
| `NVFS_IOCTL_BATCH_IO` | `_IOW('t', 8, int)` | 批量 I/O 操作 |

---

## 4. 核心函数解析

### 4.1 `nvfs_init()` - 模块初始化

**函数原型：**

```c
static int __init nvfs_init(void)
```

**功能**：初始化 GDS 内核模块

**返回值：**
- `0`：成功
- 负值：错误码

**实现逻辑：**

```c
static int __init nvfs_init(void)
{
    // 1. 检测 NVIDIA 驱动版本
    int version = get_nvidia_driver_version();
    
    // 2. 根据驱动版本选择 P2P 分配方式
    if (version >= NVIDIA_MIN_DRIVER_FOR_VGPU) {
        nvfs_use_legacy_p2p_allocation = 0;  // 使用 persistent_p2p API
    }
    
    // 3. 注册字符设备
    major_number = register_chrdev(0, DEVICE_NAME, &nvfs_dev_fops);
    
    // 4. 创建设备类
    nvfs_class = class_create(CLASS_NAME);
    
    // 5. 创建多个设备节点（/dev/nvidia-fs0, /dev/nvidia-fs1, ...）
    for (i = 0; i < nvfs_curr_devices; i++) {
        nvfs_device[i] = device_create(nvfs_class, NULL,
                MKDEV(major_number, i), NULL, DEVICE_NAME"%d", i);
    }
    
    // 6. 初始化子系统
    nvfs_mgroup_init();       // 初始化内存组管理
    nvfs_proc_init();         // 初始化 procfs 接口
    nvfs_stat_init();         // 初始化统计子系统
    nvfs_fill_gpu2peer_distance_table_once();  // 计算 GPU-存储距离
    
    return 0;
}
```

---

### 4.2 `nvfs_open()` - 设备打开

**函数原型：**

```c
static int nvfs_open(struct inode *inode, struct file *file)
```

**功能**：打开 GDS 设备文件

**返回值：**
- `0`：成功
- 负值：错误码

**实现逻辑：**

```c
static int nvfs_open(struct inode *inode, struct file *file)
{
    // 1. 检查模块是否正在关闭
    if (atomic_read(&nvfs_shutdown) == 1)
        return -EINVAL;
    
    // 2. 获取操作计数
    mutex_lock(&nvfs_module_mutex);
    nvfs_get_ops();
    
    // 3. 注册 DMA 操作（探测存储供应商模块）
    ret = nvfs_blk_register_dma_ops();
    
    mutex_unlock(&nvfs_module_mutex);
    return ret;
}
```

---

### 4.3 `nvfs_ioctl()` - IOCTL 处理入口

**函数原型：**

```c
static long nvfs_ioctl(struct file *file, unsigned int ioctl_num,
                       unsigned long ioctl_param)
```

**功能**：处理用户空间的 IOCTL 请求

**参数：**
- `file`：文件指针
- `ioctl_num`：IOCTL 命令号
- `ioctl_param`：用户空间参数地址

**返回值：**
- `0`：成功
- 负值：错误码

**支持的 IOCTL 命令：**

| 命令 | 处理函数 | 说明 |
|------|----------|------|
| `NVFS_IOCTL_MAP` | `nvfs_map()` | 映射 GPU 缓冲区 |
| `NVFS_IOCTL_READ` | `nvfs_io_start_op()` | 读操作 |
| `NVFS_IOCTL_WRITE` | `nvfs_io_start_op()` | 写操作 |
| `NVFS_IOCTL_BATCH_IO` | `nvfs_io_batch_submit()` | 批量 I/O |
| `NVFS_IOCTL_SET_RDMA_REG_INFO` | `nvfs_set_rdma_reg_info_to_mgroup()` | 设置 RDMA 信息 |
| `NVFS_IOCTL_GET_RDMA_REG_INFO` | `nvfs_get_rdma_reg_info_from_mgroup()` | 获取 RDMA 信息 |
| `NVFS_IOCTL_CLEAR_RDMA_REG_INFO` | `nvfs_clear_rdma_reg_info_in_mgroup()` | 清除 RDMA 信息 |

---

### 4.4 `nvfs_map()` - GPU 缓冲区映射

**函数原型：**

```c
static int nvfs_map(nvfs_ioctl_map_t *input_param)
```

**功能**：将 GPU 缓冲区映射到 GDS 驱动

**参数：**
- `input_param`：映射参数

**返回值：**
- `0`：成功
- 负值：错误码

**实现流程：**

```
nvfs_map()
├── 固定 Shadow Buffer 页面
├── 获取 GPU 信息结构
├── 设置 PCI 设备信息
├── 映射 GPU 信息
│   ├── 获取 end_fence 页面
│   └── 固定 GPU 页面
│       ├── 计算 GPU 虚拟地址范围
│       ├── 调用 nvidia_p2p_get_pages()
│       └── 验证页表版本和页大小
└── 更新 I/O 状态为 IO_READY
```

---

### 4.5 `nvfs_io_init()` - I/O 操作初始化

**函数原型：**

```c
struct nvfs_io* nvfs_io_init(int op, nvfs_ioctl_ioargs_t *ioargs)
```

**功能**：初始化 I/O 操作结构

**参数：**
- `op`：操作类型（READ 或 WRITE）
- `ioargs`：I/O 参数

**返回值：**
- 成功：指向 `nvfs_io` 的指针
- 失败：`ERR_PTR(-EINVAL)` 或其他错误码

**验证逻辑：**

```c
// 1. 验证偏移量和大小
if (ioargs->offset < 0) return ERR_PTR(-EINVAL);
if (ioargs->offset % NVFS_BLOCK_SIZE || ioargs->size % NVFS_BLOCK_SIZE)
    return ERR_PTR(-EINVAL);

// 2. 验证文件描述符
fd = fdget(ioargs->fd);
if (!file) return ERR_PTR(-EINVAL);

// 3. 验证文件权限
ret = nvfs_check_file_permissions(op, file, ioargs->allowreads);

// 4. 验证文件参数（inode、设备号等）
if (file_args->inum != inode->i_ino) return ERR_PTR(-ESTALE);

// 5. 获取内存组
nvfs_mgroup = nvfs_get_mgroup_from_vaddr(ioargs->cpuvaddr);

// 6. 更新 I/O 状态
nvfs_transit_state(gpu_info, sync, IO_READY, IO_IN_PROGRESS);
```

---

### 4.6 `nvfs_io_start_op()` - 启动 I/O 操作

**函数原型：**

```c
long nvfs_io_start_op(nvfs_io_t* nvfsio)
```

**功能**：启动 I/O 操作

**参数：**
- `nvfsio`：I/O 操作结构指针

**返回值：**
- 非负值：成功传输的字节数
- 负值：错误码

**实现逻辑：**

```c
long nvfs_io_start_op(nvfs_io_t* nvfsio)
{
    // 1. 写操作预处理
    if (op == WRITE) {
        // 检查是否需要 fallocate
        if (!file_is_bdev && nvfs_need_fallocate(inode)) {
            if (i_size_read(inode) == 0) {
                ret = f->f_op->fallocate(f, 0, 0, 1);
            }
        }
        // 刷新脏页
        ret = flush_dirty_pages(f, fd_offset, bytes_left, nvfsio);
    }
    
    // 2. 循环处理 I/O
    while (bytes_left) {
        // 检查终止请求
        if (atomic_read(&gpu_info->io_state) != IO_IN_PROGRESS) {
            ret = -EIO;
            goto failed;
        }
        
        // 计算本次处理的字节数
        bytes_issued = min(bytes_left, shadow_buf_size - rdma_seg_offset);
        
        // 填充元数据页
        ret = nvfs_mgroup_fill_mpages(nvfs_mgroup, nr_blocks);
        
        // 检查稀疏文件
        if (op == READ && nvfs_is_sparse(f)) {
            nvfsio->check_sparse = true;
        }
        
        // 执行直接 I/O
        ret = nvfs_direct_io(op, f, nvfsio->cpuvaddr, bytes_issued, fd_offset, nvfsio);
        
        // 更新进度
        bytes_done += ret;
        bytes_left -= ret;
        fd_offset += ret;
    }
    
    return bytes_done;
}
```

---

### 4.7 `nvfs_io_complete()` - I/O 完成回调

**函数原型：**

```c
static void nvfs_io_complete(struct kiocb *kiocb, long res)
```

**功能**：异步 I/O 完成回调（从中断上下文调用）

**参数：**
- `kiocb`：内核 I/O 控制块
- `res`：I/O 结果

**实现逻辑：**

```c
static void nvfs_io_complete(struct kiocb *kiocb, long res)
{
    nvfs_io_t* nvfsio = container_of(kiocb, struct nvfs_io, common);
    
    // 1. 更新统计信息
    if (nvfsio->common.ki_flags & IOCB_WRITE) {
        file_end_write(file);
        if (res >= 0 && nvfsio->rw_stats_enabled) {
            nvfs_stat64_add(res, &nvfs_n_write_bytes);
            nvfs_update_write_throughput(res, &nvfs_write_bytes_per_sec);
        }
    } else {
        if (res >= 0 && nvfsio->rw_stats_enabled) {
            nvfs_stat64_add(res, &nvfs_n_read_bytes);
        }
    }
    
    // 2. 异步 I/O：释放资源
    if (!nvfsio->sync)
        nvfs_io_free(nvfsio, res);
    
    nvfs_put_ops();
}
```

---

### 4.8 `nvfs_get_pages_free_callback()` - GPU 页面释放回调

**函数原型：**

```c
static void nvfs_get_pages_free_callback(void *data)
```

**功能**：NVIDIA 驱动调用此回调释放 GPU 页面

**触发条件：**
1. 用户空间显式释放 GPU 内存
2. 进程异常退出

**实现逻辑：**

```c
static void nvfs_get_pages_free_callback(void *data)
{
    nvfs_mgroup_ptr_t nvfs_mgroup = data;
    struct nvfs_gpu_args *gpu_info = &nvfs_mgroup->gpu_info;
    
    // 1. 终止正在进行的 I/O
    nvfs_io_terminate(gpu_info, 1);
    
    // 2. 清理 DMA 映射哈希表
    hash_for_each_safe(gpu_info->buckets, bkt, tmp, pci_dev_mapping, hentry) {
        nvfs_nvidia_p2p_free_dma_mapping(pci_dev_mapping->dma_mapping);
        hash_del(&pci_dev_mapping->hentry);
        kfree(pci_dev_mapping);
    }
    
    // 3. 释放页表
    page_table = xchg(&gpu_info->page_table, NULL);
    if (page_table) {
        nvfs_nvidia_p2p_free_page_table(page_table);
    }
    
    // 4. 标记元数据页状态为 DIED
    nvfs_ioctl_mpage_ptr->state = NVFS_IO_META_DIED;
    
    // 5. 释放引用
    nvfs_mgroup_put(nvfs_mgroup);
}
```

---

## 5. 模块参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `max_devices` | `uint` | 16 | 字符设备数量 |
| `dbg_enabled` | `uint` | 0 | 启用调试日志 |
| `info_enabled` | `uint` | 1 | 启用信息日志 |
| `peer_stats_enabled` | `uint` | 0 | 启用对等设备统计 |
| `rw_stats_enabled` | `uint` | 0 | 启用读写统计 |
| `use_legacy_p2p_allocation` | `uint` | 1 | 使用传统 P2P 分配方式 |

---

## 6. 常量定义

```c
#define DEVICE_NAME "nvidia-fs"
#define CLASS_NAME "nvidia-fs-class"
#define MAX_NVFS_DEVICES 16U

#define GPU_PAGE_SHIFT   16
#define GPU_PAGE_SIZE    ((u64)1 << GPU_PAGE_SHIFT)  // 64KB
#define GPU_PAGE_MASK    (~(GPU_PAGE_SIZE - 1))

#define NVFS_BLOCK_SIZE  4096  // 4KB
#define MAX_IO_RETRY     5

#define NVFS_P2P_MAX_CONTIG_GPU_PAGES 65535  // 最大连续 GPU 页数

// 文件系统魔数
#define LUSTRE_SUPER_MAGIC  0x0bd00bd0U
#define BEEGFS_SUPER_MAGIC  0x19830326U
#define SCATEFS_SUPER_MAGIC 0x53544653
```

---

## 7. I/O 状态机

```
IO_FREE ──► IO_INIT ──► IO_READY ──► IO_IN_PROGRESS ──► IO_READY
   │            │            │              │               │
   │            │            │              ▼               │
   │            │            │     IO_TERMINATE_REQ         │
   │            │            │              │               │
   │            │            ▼              ▼               │
   │            │      IO_CALLBACK_REQ ◄────┘               │
   │            │            │                              │
   │            ▼            ▼                              │
   └──────► IO_TERMINATED ◄─────────────────────────────────┘
```

---

## 8. 使用示例

### 8.1 映射 GPU 缓冲区

```c
#include <sys/ioctl.h>
#include <fcntl.h>

int fd = open("/dev/nvidia-fs0", O_RDWR);

struct nvfs_ioctl_map_s map_args = {
    .size = 1024 * 1024 * 1024,  // 1GB
    .pdevinfo = 0x0000000001000000,
    .cpuvaddr = (u64)shadow_buffer,
    .gpuvaddr = (u64)gpu_buffer,
    .end_fence_addr = (u64)fence_addr,
    .sbuf_block = 262144,  // 1GB / 4KB
    .is_bounce_buffer = 0
};

int ret = ioctl(fd, NVFS_IOCTL_MAP, &map_args);
```

### 8.2 同步读操作

```c
struct nvfs_ioctl_ioargs io_args = {
    .cpuvaddr = map_args.cpuvaddr,
    .offset = 0,
    .size = 4096,
    .fd = file_fd,
    .sync = 1,
    .hipri = 0,
    .allowreads = 0,
    .use_rkeys = 0,
    .optype = 0  // READ
};

int ret = ioctl(fd, NVFS_IOCTL_READ, &io_args);
```

### 8.3 异步写操作

```c
struct nvfs_ioctl_ioargs io_args = {
    .cpuvaddr = map_args.cpuvaddr,
    .offset = 0,
    .size = 4096,
    .end_fence_value = 1,  // 完成时写入此值
    .fd = file_fd,
    .sync = 0,  // 异步
    .optype = 1  // WRITE
};

int ret = ioctl(fd, NVFS_IOCTL_WRITE, &io_args);

// 轮询完成状态
while (*(volatile u64*)fence_addr != 1) {
    // 等待完成
}
```

---

## 9. 相关文件

| 文件 | 说明 |
|------|------|
| `nvfs-mmap.c/h` | 内存映射管理 |
| `nvfs-dma.c/h` | DMA 操作接口 |
| `nvfs-pci.c/h` | PCI 拓扑管理 |
| `nvfs-stat.c/h` | 性能统计 |
| `nvfs-batch.c/h` | 批量 I/O |
| `nvfs-rdma.c/h` | RDMA 支持 |

---

## 10. 注意事项

1. **O_DIRECT 要求**：所有 I/O 操作必须使用 O_DIRECT 标志打开文件
2. **对齐要求**：偏移量和大小必须是 4KB 对齐
3. **GPU 页大小**：GPU 页面大小为 64KB
4. **版本兼容**：NVIDIA 驱动版本 >= 555 使用 persistent_p2p API
5. **线程安全**：所有 IOCTL 操作都是线程安全的
6. **引用计数**：使用 `nvfs_get_ops()/nvfs_put_ops()` 管理模块引用
