# nvfs-mmap.c/h 文件解读

## 1. 文件功能描述

`nvfs-mmap.c/h` 是 NVIDIA GPUDirect Storage (GDS) 内核模块的 **内存映射管理实现文件**，负责 GPU 内存映射、Shadow Buffer 管理、I/O 元数据跟踪以及 VMA 操作。该文件是 GDS 驱动实现 GPU 直接存储访问的核心组件，管理着 GPU 内存与存储设备之间的数据传输通道。

### 主要功能：
- **Shadow Buffer 管理**：分配、固定和释放 CPU 可访问的代理缓冲区
- **GPU 内存映射**：通过 NVIDIA P2P 接口固定 GPU 内存页
- **I/O 元数据管理**：跟踪每个 4KB 块的 I/O 状态
- **VMA 操作**：处理用户空间 mmap/munmap 操作
- **哈希表索引**：通过哈希表快速查找 I/O 组

---

## 2. 核心数据结构说明

### 2.1 `struct nvfs_io_mgroup` - I/O 内存组

```c
struct nvfs_io_mgroup {
    atomic_t ref;                           // 引用计数
    atomic_t dma_ref;                       // DMA 引用计数
    struct hlist_node hash_link;            // 哈希表链接节点
    u64 cpu_base_vaddr;                     // CPU 虚拟地址基址
    unsigned long base_index;               // 基址索引（哈希键）
    unsigned long nvfs_blocks_count;        // 4KB 块数量
    struct page **nvfs_ppages;              // 页指针数组
    struct nvfs_io_metadata *nvfs_metadata; // I/O 元数据数组
    struct nvfs_gpu_args gpu_info;          // GPU 信息
    nvfs_io_t nvfsio;                       // I/O 操作结构
    #ifdef NVFS_ENABLE_KERN_RDMA_SUPPORT
    struct nvfs_rdma_info rdma_info;        // RDMA 信息
    #endif
    atomic_t next_segment;                  // 下一段索引
    #ifdef CONFIG_FAULT_INJECTION
    bool fault_injected;                    // 故障注入标志
    #endif
};
```

**字段说明：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `ref` | `atomic_t` | 引用计数，管理对象生命周期 |
| `dma_ref` | `atomic_t` | DMA 引用计数，跟踪活跃 DMA 操作 |
| `hash_link` | `struct hlist_node` | 哈希表节点，用于快速查找 |
| `cpu_base_vaddr` | `u64` | Shadow Buffer 的 CPU 虚拟地址基址 |
| `base_index` | `unsigned long` | 基址索引，作为哈希键使用 |
| `nvfs_blocks_count` | `unsigned long` | 4KB 块的总数量 |
| `nvfs_ppages` | `struct page **` | 页指针数组，指向 Shadow Buffer 页 |
| `nvfs_metadata` | `struct nvfs_io_metadata *` | 元数据数组，每个 4KB 块一个条目 |
| `gpu_info` | `struct nvfs_gpu_args` | GPU 相关信息（页表、地址等） |
| `nvfsio` | `nvfs_io_t` | 当前 I/O 操作参数 |

---

### 2.2 `struct nvfs_gpu_args` - GPU 参数

```c
struct nvfs_gpu_args {
    nvidia_p2p_page_table_t *page_table;    // NVIDIA P2P 页表
    u64 gpuvaddr;                           // GPU 虚拟地址
    u64 gpu_buf_len;                        // GPU 缓冲区长度
    struct page *end_fence_page;            // 完成栅栏页
    u32 offset_in_page;                     // 栅栏页内偏移
    atomic_t io_state;                      // I/O 状态
    atomic_t dma_mapping_in_progress;       // DMA 映射进行中标志
    atomic_t callback_invoked;              // 回调已调用标志
    wait_queue_head_t callback_wq;          // 回调等待队列
    bool is_bounce_buffer;                  // 是否为弹跳缓冲区
    bool use_legacy_p2p_allocation;         // 使用传统 P2P 分配
    int n_phys_chunks;                      // 物理块数量
    u64 pdevinfo;                           // PCI 设备信息
    unsigned int gpu_hash_index;            // GPU 哈希索引
    DECLARE_HASHTABLE(buckets, MAX_PCI_BUCKETS_BITS);  // PCI 设备桶
};
```

**字段说明：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `page_table` | `nvidia_p2p_page_table_t *` | NVIDIA P2P 页表，包含 GPU 物理地址 |
| `gpuvaddr` | `u64` | GPU 缓冲区的虚拟地址 |
| `gpu_buf_len` | `u64` | GPU 缓冲区长度（字节） |
| `end_fence_page` | `struct page *` | 完成栅栏页，用于异步 I/O 通知 |
| `io_state` | `atomic_t` | I/O 状态机当前状态 |
| `pdevinfo` | `u64` | PCI 设备信息，用于亲和性计算 |
| `buckets` | 哈希表 | DMA 映射缓存桶 |

---

### 2.3 `struct nvfs_io_metadata` - I/O 元数据

```c
struct nvfs_io_metadata {
    u64 nvfs_start_magic;                   // 魔数，用于验证
    enum nvfs_block_state nvfs_state;       // 块状态
    struct page *page;                      // 关联的页指针
} __attribute__((packed, aligned(8)));
```

**块状态枚举：**

```c
enum nvfs_block_state {
    NVFS_IO_FREE = 0,       // 初始状态
    NVFS_IO_ALLOC,          // 已分配
    NVFS_IO_INIT,           // 已初始化
    NVFS_IO_QUEUED,         // 已入队
    NVFS_IO_DMA_START,      // DMA 开始
    NVFS_IO_DONE,           // I/O 完成
    NVFS_IO_DMA_ERROR,      // DMA 错误
};
```

---

### 2.4 `nvfs_io_t` - I/O 操作结构

```c
typedef struct nvfs_io {
    char __user *cpuvaddr;                  // Shadow Buffer 地址
    u64 length;                             // I/O 长度
    ssize_t ret;                            // 返回值
    loff_t fd_offset;                       // 文件偏移量
    loff_t gpu_page_offset;                 // GPU 页偏移
    u64 end_fence_value;                    // 栅栏完成值
    struct fd fd;                           // 文件描述符
    int op;                                 // 操作类型（READ/WRITE）
    bool sync;                              // 同步标志
    bool hipri;                             // 高优先级标志
    bool check_sparse;                      // 检查稀疏文件
    bool rw_stats_enabled;                  // 启用统计
    unsigned long cur_gpu_base_index;       // 当前 GPU 基索引
    unsigned long nvfs_active_blocks_start; // 活跃块起始
    unsigned long nvfs_active_blocks_end;   // 活跃块结束
    nvfs_metastate_enum state;              // 元状态
    int retrycnt;                           // 重试计数
    wait_queue_head_t rw_wq;                // 等待队列
    struct kiocb common;                    // 内核 I/O 控制块
    ktime_t start_io;                       // I/O 开始时间
    ssize_t rdma_seg_offset;                // RDMA 段偏移
    bool use_rkeys;                         // 使用 RDMA RKey
} nvfs_io_t;
```

---

### 2.5 `struct nvfs_rdma_info` - RDMA 信息

```c
typedef struct nvfs_rdma_info {
    uint8_t  version;                       // 版本号
    uint8_t  flags;                         // 标志位
    uint16_t lid;                           // 本地标识符
    uint32_t qp_num;                        // QP 号
    uint64_t rem_vaddr;                     // 远程虚拟地址
    uint32_t size;                          // 缓冲区大小
    uint32_t rkey;                          // 远程密钥
    uint64_t gid[2];                        // 全局标识符
    uint32_t dc_key;                        // DC 密钥
} nvfs_rdma_info_t;
```

---

## 3. I/O 状态机

### 3.1 状态定义

```c
typedef enum nvfs_io_state {
    IO_FREE = 0,                // 空闲状态
    IO_INIT = 1,                // 已初始化
    IO_READY = 2,               // 准备就绪
    IO_IN_PROGRESS = 3,         // I/O 进行中
    IO_TERMINATE_REQ = 4,       // 终止请求
    IO_TERMINATED = 5,          // 已终止
    IO_CALLBACK_REQ = 6,        // 回调请求
    IO_CALLBACK_END = 7,        // 回调完成
    IO_UNPIN_PAGES_ALREADY_INVOKED = 8,  // 页面已解除固定
} nvfs_io_state;
```

### 3.2 状态转换图

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
                           │
                           ▼
                   IO_CALLBACK_END
```

---

## 4. 核心函数解析

### 4.1 `nvfs_mgroup_get()` - 获取 I/O 组

**函数原型：**

```c
nvfs_mgroup_ptr_t nvfs_mgroup_get(unsigned long base_index)
```

**功能**：根据基址索引获取 I/O 内存组

**参数：**
- `base_index`：基址索引

**返回值：**
- 成功：指向 `nvfs_io_mgroup` 的指针
- 失败：NULL

**实现逻辑：**

```c
nvfs_mgroup_ptr_t nvfs_mgroup_get(unsigned long base_index)
{
    nvfs_mgroup_ptr_t nvfs_mgroup;
    
    rcu_read_lock();
    hash_for_each_possible_rcu(nvfs_io_mgroup_hash, nvfs_mgroup, hash_link, base_index) {
        if (nvfs_mgroup->base_index == base_index) {
            // 检查 I/O 状态
            if (atomic_read(&gpu_info->io_state) > IO_IN_PROGRESS) {
                nvfs_info("nvfs_mgroup found but IO is in %s state\n",
                         nvfs_io_state_status(atomic_read(&gpu_info->io_state)));
            }
            nvfs_mgroup_get_ref(nvfs_mgroup);
            rcu_read_unlock();
            return nvfs_mgroup;
        }
    }
    rcu_read_unlock();
    
    return NULL;
}
```

---

### 4.2 `nvfs_mgroup_put()` - 释放 I/O 组引用

**函数原型：**

```c
void nvfs_mgroup_put(nvfs_mgroup_ptr_t nvfs_mgroup)
```

**功能**：释放 I/O 内存组引用

**参数：**
- `nvfs_mgroup`：I/O 内存组指针

**实现逻辑：**

```c
void nvfs_mgroup_put(nvfs_mgroup_ptr_t nvfs_mgroup)
{
    if (nvfs_mgroup == NULL)
        return;
    
    if (nvfs_mgroup_put_ref(nvfs_mgroup)) {
        // 引用计数为 0，释放资源
        nvfs_mgroup_free(nvfs_mgroup, false);
    }
}
```

---

### 4.3 `nvfs_mgroup_mmap()` - mmap 操作

**函数原型：**

```c
int nvfs_mgroup_mmap(struct file *filp, struct vm_area_struct *vma)
```

**功能**：处理用户空间的 mmap 请求

**参数：**
- `filp`：文件指针
- `vma`：VMA 结构

**返回值：**
- `0`：成功
- 负值：错误码

**实现逻辑：**

```c
static int nvfs_mgroup_mmap_internal(struct file *filp, struct vm_area_struct *vma)
{
    unsigned long length = vma->vm_end - vma->vm_start;
    
    // 1. 验证映射大小
    if (length > NVFS_MAX_SHADOW_PAGES * PAGE_SIZE)
        goto error;
    
    // 2. 验证对齐
    if ((length < GPU_PAGE_SIZE) && (length % NVFS_BLOCK_SIZE))
        goto error;
    if (length > GPU_PAGE_SIZE && (length % GPU_PAGE_SIZE))
        goto error;
    
    // 3. 验证 VMA 标志
    if ((vm_flags & (VM_MAYREAD|VM_READ|VM_MAYWRITE|VM_WRITE)) != 
        (VM_MAYREAD|VM_READ|VM_MAYWRITE|VM_WRITE))
        goto error;
    
    // 4. 设置 VMA 标志
    vm_flags_set(vma, VM_MIXEDMAP | VM_DONTEXPAND | VM_DONTDUMP | VM_DONTCOPY);
    vma->vm_ops = &nvfs_mmap_ops;
    
    // 5. 分配 I/O 组
    nvfs_mgroup = kzalloc(sizeof(struct nvfs_io_mgroup), GFP_KERNEL);
    
    // 6. 生成随机基址索引
    base_index = NVFS_MIN_BASE_INDEX + get_random_u32();
    atomic_set(&nvfs_mgroup->ref, 1);
    hash_add_rcu(nvfs_io_mgroup_hash, &nvfs_mgroup->hash_link, base_index);
    
    // 7. 分配页指针数组和元数据
    nvfs_mgroup->nvfs_ppages = kzalloc(os_pages_count * sizeof(struct page*), GFP_KERNEL);
    nvfs_mgroup->nvfs_metadata = kzalloc(nvfs_blocks_count * sizeof(struct nvfs_io_metadata), GFP_KERNEL);
    
    // 8. 插入页面到 VMA
    for (i = 0; i < nvfs_blocks_count; i++) {
        j = i / nvfs_block_count_per_page;
        if (nvfs_mgroup->nvfs_ppages[j] == NULL) {
            nvfs_mgroup->nvfs_ppages[j] = alloc_page(GFP_USER|__GFP_ZERO);
            NVFS_PAGE_INDEX(nvfs_mgroup->nvfs_ppages[j]) = (base_index * NVFS_MAX_SHADOW_PAGES) + j;
            ret = vm_insert_page(vma, vma->vm_start + j * PAGE_SIZE, nvfs_mgroup->nvfs_ppages[j]);
        }
        nvfs_mgroup->nvfs_metadata[i].nvfs_start_magic = NVFS_START_MAGIC;
        nvfs_mgroup->nvfs_metadata[i].nvfs_state = NVFS_IO_ALLOC;
        nvfs_mgroup->nvfs_metadata[i].page = nvfs_mgroup->nvfs_ppages[j];
    }
    
    return 0;
}
```

---

### 4.4 `nvfs_mgroup_pin_shadow_pages()` - 固定 Shadow Buffer 页面

**函数原型：**

```c
nvfs_mgroup_ptr_t nvfs_mgroup_pin_shadow_pages(u64 cpuvaddr, unsigned long length)
```

**功能**：固定用户空间的 Shadow Buffer 页面

**参数：**
- `cpuvaddr`：CPU 虚拟地址
- `length`：缓冲区长度

**返回值：**
- 成功：指向 `nvfs_io_mgroup` 的指针
- 失败：NULL

**实现逻辑：**

```c
nvfs_mgroup_ptr_t nvfs_mgroup_pin_shadow_pages(u64 cpuvaddr, unsigned long length)
{
    count = DIV_ROUND_UP(length, PAGE_SIZE);
    pages = kmalloc(count * sizeof(struct page *), GFP_KERNEL);
    
    // 固定用户页面
    #ifdef HAVE_PIN_USER_PAGES_FAST
    ret = pin_user_pages_fast(cpuvaddr, count, 1, pages);
    #else
    ret = get_user_pages_fast(cpuvaddr, count, 1, pages);
    #endif
    
    // 验证页面并获取 I/O 组
    for (j = 0; j < count; j++) {
        cur_base_index = (NVFS_PAGE_INDEX(pages[j]) >> NVFS_MAX_SHADOW_PAGES_ORDER);
        if (j == 0) {
            nvfs_mgroup = nvfs_mgroup_get(cur_base_index);
        }
        // 验证页面匹配
        BUG_ON(nvfs_mgroup->base_index != cur_base_index);
        BUG_ON(nvfs_mgroup->nvfs_ppages[j] != pages[j]);
    }
    
    nvfs_mgroup->cpu_base_vaddr = cpuvaddr;
    nvfs_mgroup_check_and_set(nvfs_mgroup, NVFS_IO_INIT, true, false);
    
    return nvfs_mgroup;
}
```

---

### 4.5 `nvfs_is_gpu_page()` - 检查是否为 GPU 页面

**函数原型：**

```c
bool nvfs_is_gpu_page(struct page *page)
```

**功能**：检查页面是否属于 GPU 请求

**参数：**
- `page`：页指针

**返回值：**
- `true`：是 GPU 页面
- `false`：不是 GPU 页面

**实现逻辑：**

```c
bool nvfs_is_gpu_page(struct page *page)
{
    nvfs_mgroup_ptr_t nvfs_mgroup;
    
    // 检查页面映射是否为空
    if (page == NULL || page->mapping != NULL)
        return false;
    
    // 获取基址索引
    base_index = (NVFS_PAGE_INDEX(page) >> NVFS_MAX_SHADOW_PAGES_ORDER);
    if (base_index < NVFS_MIN_BASE_INDEX)
        return false;
    
    // 查找 I/O 组
    nvfs_mgroup = nvfs_mgroup_get(base_index);
    if (nvfs_mgroup == NULL)
        return false;
    
    nvfs_mgroup_put(nvfs_mgroup);
    return true;
}
```

---

### 4.6 `nvfs_mgroup_get_gpu_physical_address()` - 获取 GPU 物理地址

**函数原型：**

```c
uint64_t nvfs_mgroup_get_gpu_physical_address(nvfs_mgroup_ptr_t nvfs_mgroup, struct page* page)
```

**功能**：获取页面对应的 GPU 物理地址

**参数：**
- `nvfs_mgroup`：I/O 内存组指针
- `page`：页指针

**返回值：**
- GPU 物理地址

**实现逻辑：**

```c
uint64_t nvfs_mgroup_get_gpu_physical_address(nvfs_mgroup_ptr_t nvfs_mgroup, struct page* page)
{
    struct nvfs_gpu_args *gpu_info = &nvfs_mgroup->gpu_info;
    unsigned long gpu_page_index;
    pgoff_t pgoff;
    dma_addr_t phys_base_addr, phys_start_addr;
    
    // 获取 GPU 页索引和偏移
    nvfs_mgroup_get_gpu_index_and_off(nvfs_mgroup, page, &gpu_page_index, &pgoff);
    
    // 从 P2P 页表获取物理地址
    phys_base_addr = gpu_info->page_table->pages[gpu_page_index]->physical_address;
    phys_start_addr = phys_base_addr + pgoff;
    
    return phys_start_addr;
}
```

---

## 5. 常量定义

```c
#define KiB4                        (4096)
#define NVFS_BLOCK_SIZE             (4096)          // 4KB 块大小
#define NVFS_BLOCK_SHIFT            (12)
#define NVFS_MIN_BASE_INDEX         ((unsigned long)1L<<32)  // 最小基址索引
#define NVFS_MAX_SHADOW_PAGES_ORDER (12 - NVFS_PAGE_TO_BLOCK_ORDER)
#define NVFS_MAX_SHADOW_PAGES       (1 << NVFS_MAX_SHADOW_PAGES_ORDER)  // 最大 Shadow 页数
#define NVFS_MAX_SHADOW_ALLOCS_ORDER 12

#define MAX_PCI_BUCKETS             32
#define MAX_PCI_BUCKETS_BITS        ilog2(MAX_PCI_BUCKETS)
#define MAX_RDMA_REGS_SUPPORTED     16

// 页面索引访问宏
#ifdef HAVE_PAGE_FOLIO_INDEX
#define NVFS_PAGE_INDEX(page)       page->__folio_index
#else
#define NVFS_PAGE_INDEX(page)       page->index
#endif
```

---

## 6. VMA 操作结构

```c
static const struct vm_operations_struct nvfs_mmap_ops = {
    .open = nvfs_vma_open,
    .split = nvfs_vma_split,
    .mremap = nvfs_vma_mremap,
    .close = nvfs_vma_close,
    .fault = nvfs_vma_fault,
    .pfn_mkwrite = nvfs_pfn_mkwrite,
    .page_mkwrite = nvfs_page_mkwrite,
};
```

**操作说明：**

| 操作 | 说明 |
|------|------|
| `open` | VMA 打开（禁止） |
| `split` | VMA 分割（禁止） |
| `mremap` | VMA 重映射（禁止） |
| `close` | VMA 关闭，释放资源 |
| `fault` | 页面错误（禁止） |
| `pfn_mkwrite` | PFN 写保护（禁止） |
| `page_mkwrite` | 页面写保护（禁止） |

---

## 7. 使用示例

### 7.1 用户空间 mmap 调用

```c
// 用户空间分配 Shadow Buffer
void *shadow_buf = mmap(NULL, size, PROT_READ|PROT_WRITE, 
                        MAP_SHARED, fd, 0);
```

### 7.2 内核处理 mmap

```c
// 内核 mmap 处理
int nvfs_mgroup_mmap(struct file *filp, struct vm_area_struct *vma)
{
    return nvfs_mgroup_mmap_internal(filp, vma);
}
```

### 7.3 固定 Shadow Buffer

```c
// 在 nvfs_map() 中调用
nvfs_mgroup = nvfs_mgroup_pin_shadow_pages(map_args->cpuvaddr, map_args->size);
```

---

## 8. 相关文件

| 文件 | 说明 |
|------|------|
| `nvfs-mmap.c` | 内存映射实现 |
| `nvfs-mmap.h` | 内存映射头文件 |
| `nvfs-core.c` | IOCTL 处理 |
| `nvfs-dma.c` | DMA 操作 |
| `nvfs-pci.c` | PCI 设备管理 |

---

## 9. 注意事项

1. **引用计数**：使用 `nvfs_mgroup_get/put` 管理对象生命周期
2. **RCU 保护**：哈希表查找使用 RCU 读锁保护
3. **页面索引**：Shadow Buffer 页面使用自定义索引，范围 `[NVFS_MIN_BASE_INDEX, ...)`
4. **VMA 限制**：禁止分割、重映射和复制
5. **状态验证**：每个块状态转换都有严格验证
6. **内存对齐**：Shadow Buffer 必须是 4KB 或 64KB 对齐
