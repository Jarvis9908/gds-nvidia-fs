# NVIDIA-FS.KO 命令接收与地址转换机制详解

## 概述

本文档详细分析 NVIDIA GPUDirect Storage (GDS) 内核模块 `nvidia-fs.ko` 的命令接收机制和虚拟地址(VA)到GPU物理地址的转换流程。这是 GDS 实现零拷贝 GPU 直接存储访问的核心技术基础。

---

## 一、命令接收机制

### 1.1 架构概览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           用户空间应用                                        │
│                    (GDS库 / cuFile / 用户程序)                               │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ ioctl() 系统调用
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           内核空间                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    nvidia-fs.ko (字符设备驱动)                        │   │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐           │   │
│  │  │ file_operations│  │  nvfs_ioctl() │  │ 命令处理器     │           │   │
│  │  └───────────────┘  └───────────────┘  └───────────────┘           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    NVIDIA 驱动 (nvidia.ko)                           │   │
│  │              nvidia_p2p_get_pages() / P2P API                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    NVMe 驱动 / 存储子系统                             │   │
│  │                    PCIe P2P DMA 传输                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 设备注册与文件操作

nvidia-fs.ko 作为内核字符设备驱动，通过标准的 Linux 字符设备接口接收上层命令。

#### 1.2.1 文件操作结构体定义

**源码位置**: [nvfs-core.c:L2446-L2453](file:///e:/code/gds-nvidia-fs/src/nvfs-core.c#L2446-L2453)

```c
struct file_operations nvfs_dev_fops = {
    .compat_ioctl = nvfs_ioctl,      // 32位兼容ioctl
    .unlocked_ioctl = nvfs_ioctl,    // 64位ioctl
    .open = nvfs_open,               // 打开设备
    .release = nvfs_close,           // 关闭设备
    .mmap = nvfs_mgroup_mmap,        // 内存映射
    .owner = THIS_MODULE,            // 模块所有者
};
```

**字段说明**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `compat_ioctl` | 函数指针 | 处理32位应用程序在64位内核上的ioctl调用 |
| `unlocked_ioctl` | 函数指针 | 处理64位应用程序的ioctl调用（无BKL） |
| `open` | 函数指针 | 设备打开时调用，初始化进程上下文 |
| `release` | 函数指针 | 设备关闭时调用，清理资源 |
| `mmap` | 函数指针 | 内存映射操作，用于映射shadow buffer |

#### 1.2.2 模块初始化流程

**源码位置**: [nvfs-core.c:L2503-L2589](file:///e:/code/gds-nvidia-fs/src/nvfs-core.c#L2503-L2589)

```c
static int __init nvfs_init(void)
{
    int i;
    int version = get_nvidia_driver_version();

    // 根据NVIDIA驱动版本选择P2P API
    // 驱动版本 >= 555 使用持久化P2P API
    if (version >= NVIDIA_MIN_DRIVER_FOR_VGPU) {
        nvfs_use_legacy_p2p_allocation = 0;
    }

    // x86裸机环境也使用持久化P2P API
    #if defined(CONFIG_X86_64)
    if (nvfs_use_legacy_p2p_allocation == 1 && 
        !cpu_feature_enabled(X86_FEATURE_HYPERVISOR)) {
        nvfs_use_legacy_p2p_allocation = 0;
    }
    #endif

    pr_info("nvidia_fs: Initializing nvfs driver module\n");

    // 1. 注册字符设备，动态获取主设备号
    major_number = register_chrdev(0, DEVICE_NAME, &nvfs_dev_fops);
    if (major_number < 0) {
        pr_err("nvidia_fs: failed to register a major number\n");
        return major_number;
    }
    pr_info("nvidia_fs: registered correctly with major number %d\n",
            major_number);

    // 2. 创建设备类
    #ifdef CLASS_CREATE_HAS_TWO_PARAMS
    nvfs_class = class_create(THIS_MODULE, CLASS_NAME);
    #else
    nvfs_class = class_create(CLASS_NAME);
    #endif

    if (IS_ERR(nvfs_class)) {
        unregister_chrdev(major_number, DEVICE_NAME);
        pr_err("nvidia_fs: Failed to register device class\n");
        return PTR_ERR(nvfs_class);
    }
    nvfs_class->devnode = nvfs_devnode;  // 设置设备节点权限为0666

    // 3. 创建多个设备实例
    nvfs_set_device_count(nvfs_max_devices);
    nvfs_curr_devices = nvfs_get_device_count();

    for (i = 0; i < nvfs_curr_devices; i++) {
        nvfs_device[i] = device_create(nvfs_class, NULL,
                MKDEV(major_number, i),
                NULL, DEVICE_NAME"%d", i);
        if (IS_ERR(nvfs_device[i])) {
            // 创建失败，清理已创建的设备
            goto error;
        }
    }

    // 4. 初始化核心子系统
    nvfs_mgroup_init();                        // 内存组管理初始化
    atomic_set(&nvfs_shutdown, 0);             // 清除关闭标志
    init_waitqueue_head(&wq);                  // 初始化等待队列
    nvfs_proc_init();                          // 初始化procfs接口
    #ifdef CONFIG_FAULT_INJECTION
    nvfs_init_debugfs();                       // 初始化debugfs（故障注入）
    #endif
    nvfs_stat_init();                          // 初始化统计子系统
    nvfs_fill_gpu2peer_distance_table_once();  // 构建PCI拓扑距离表

    return 0;

error:
    while (i >= 0) {
        device_destroy(nvfs_class, MKDEV(major_number, i));
        i -= 1;
    }
    return -1;
}
```

**初始化流程图**:

```
nvfs_init()
    │
    ├── 检测NVIDIA驱动版本
    │       └── 选择P2P API类型
    │
    ├── register_chrdev()
    │       └── 注册字符设备，获取主设备号
    │
    ├── class_create()
    │       └── 创建设备类 /sys/class/nvidia-fs
    │
    ├── device_create() × N
    │       └── 创建设备节点 /dev/nvidia-fs0, /dev/nvidia-fs1, ...
    │
    └── 子系统初始化
            ├── nvfs_mgroup_init()      (内存组管理)
            ├── nvfs_proc_init()        (procfs接口)
            ├── nvfs_stat_init()        (统计子系统)
            └── nvfs_fill_gpu2peer_distance_table_once() (PCI拓扑)
```

### 1.3 IOCTL 命令分发

#### 1.3.1 IOCTL 入口函数

**源码位置**: [nvfs-core.c:L2210-L2444](file:///e:/code/gds-nvidia-fs/src/nvfs-core.c#L2210-L2444)

```c
static long nvfs_ioctl(struct file *file, unsigned int ioctl_num,
                        unsigned long ioctl_param)
{
    int pid = current->tgid;
    nvfs_ioctl_param_union local_param;

    // 1. 从用户空间拷贝参数
    if (copy_from_user((void *) &local_param, (void *) ioctl_param,
        sizeof(nvfs_ioctl_param_union))) {
        nvfs_err("%s:%d copy_from_user failed\n", __func__, __LINE__);
        return -ENOMEM;
    }

    // 2. 检查驱动是否正在关闭
    if (atomic_read(&nvfs_shutdown) == 1)
        return -EINVAL;

    // 3. 根据命令号分发处理
    switch (ioctl_num) {

    case NVFS_IOCTL_REMOVE:
    {
        nvfs_dbg("nvfs ioctl remove invoked\n");
        nvfs_remove(pid, NULL);
        return 0;
    }

    case NVFS_IOCTL_READ:
    case NVFS_IOCTL_WRITE:
    {
        nvfs_io_t* nvfsio = NULL;
        int op = get_rwop(ioctl_num);  // READ 或 WRITE
        const char *io = (op == READ) ? "Read" : "Write";
        bool rw_stats_enabled = (nvfs_rw_stats_enabled > 0);

        // 统计计数
        if (op == READ) {
            if (rw_stats_enabled) {
                nvfs_stat64(&nvfs_n_reads);
                nvfs_stat(&nvfs_n_op_reads);
            }
        } else {
            if (rw_stats_enabled) {
                nvfs_stat64(&nvfs_n_writes);
                nvfs_stat(&nvfs_n_op_writes);
            }
        }

        // 初始化I/O请求
        nvfsio = nvfs_io_init(op, &local_param.ioargs);

        if (IS_ERR(nvfsio)) {
            local_param.ioargs.ioctl_return = PTR_ERR(nvfsio);
            // 错误处理...
            return -1;
        }
        nvfsio->rw_stats_enabled = rw_stats_enabled;

        // 执行I/O操作
        local_param.ioargs.ioctl_return = nvfs_io_start_op(nvfsio);

        // 将结果返回用户空间
        if (copy_to_user((void *) ioctl_param, (void*) &local_param,
                        sizeof(nvfs_ioctl_param_union))) {
            local_param.ioargs.ioctl_return = -EFAULT;
        }

        return ((local_param.ioargs.ioctl_return < 0) ? -1 : 0);
    }

    case NVFS_IOCTL_BATCH_IO:
    {
        nvfs_batch_io_t* nvfs_batch = NULL;
        bool rw_stats_enabled = (nvfs_rw_stats_enabled > 0);

        nvfs_dbg("nvfs batch ioctl invoked\n");
        if (rw_stats_enabled) {
            nvfs_stat64(&nvfs_n_batches);
            nvfs_stat(&nvfs_n_op_batches);
        }

        // 初始化批量I/O
        nvfs_batch = nvfs_io_batch_init(&local_param);

        if (IS_ERR(nvfs_batch)) {
            // 错误处理...
            return -1;
        }

        // 提交批量I/O
        local_param.ioargs.ioctl_return = nvfs_io_batch_submit(nvfs_batch);

        // 统计和返回结果...
        return ((local_param.ioargs.ioctl_return < 0) ? -1 : 0);
    }

    case NVFS_IOCTL_MAP:
    {
        int ret;

        nvfs_stat64(&nvfs_n_maps);
        nvfs_stat(&nvfs_n_op_maps);

        ret = nvfs_map(&(local_param.map_args));
        if (ret) {
            nvfs_stat(&nvfs_n_map_err);
            nvfs_stat_d(&nvfs_n_op_maps);
            return -1;
        }

        nvfs_stat64(&nvfs_n_maps_ok);
        // 返回结果...
        return 0;
    }

    case NVFS_IOCTL_SET_RDMA_REG_INFO:
    {
        #ifdef NVFS_ENABLE_KERN_RDMA_SUPPORT
        int ret = nvfs_set_rdma_reg_info_to_mgroup(
                (nvfs_ioctl_set_rdma_reg_info_args_t*)
                &local_param.rdma_set_reg_info);
        return ret;
        #else
        return -1;
        #endif
    }

    case NVFS_IOCTL_GET_RDMA_REG_INFO:
    {
        #ifdef NVFS_ENABLE_KERN_RDMA_SUPPORT
        int ret = nvfs_get_rdma_reg_info_from_mgroup(
                (nvfs_ioctl_get_rdma_reg_info_args_t*)
                &local_param.rdma_get_reg_info);
        // 返回结果...
        return ret;
        #else
        return -1;
        #endif
    }

    case NVFS_IOCTL_CLEAR_RDMA_REG_INFO:
    {
        #ifdef NVFS_ENABLE_KERN_RDMA_SUPPORT
        return nvfs_clear_rdma_reg_info_in_mgroup(
                (nvfs_ioctl_clear_rdma_reg_info_args_t*)
                &local_param.rdma_clear_reg_info);
        #else
        return -1;
        #endif
    }

    default:
    {
        nvfs_err("%s:%d Invalid IOCTL invoked\n", __func__, __LINE__);
        return -ENOTTY;
    }
    }

    return 0;
}
```

#### 1.3.2 支持的 IOCTL 命令

| 命令 | 值 | 功能 | 参数类型 | 说明 |
|------|-----|------|----------|------|
| `NVFS_IOCTL_MAP` | - | GPU内存映射 | `nvfs_ioctl_map_args_t` | 建立VA到GPU物理地址的映射关系 |
| `NVFS_IOCTL_READ` | - | 读操作 | `nvfs_ioctl_ioargs_t` | 从存储设备读取数据到GPU内存 |
| `NVFS_IOCTL_WRITE` | - | 写操作 | `nvfs_ioctl_ioargs_t` | 将GPU内存数据写入存储设备 |
| `NVFS_IOCTL_BATCH_IO` | - | 批量I/O | `nvfs_ioctl_batch_ioargs_t` | 一次提交最多256个I/O请求 |
| `NVFS_IOCTL_REMOVE` | - | 移除映射 | `void` | 释放GPU内存映射资源 |
| `NVFS_IOCTL_SET_RDMA_REG_INFO` | - | 设置RDMA信息 | `nvfs_ioctl_set_rdma_reg_info_args_t` | 用于IBM GPFS等RDMA场景 |
| `NVFS_IOCTL_GET_RDMA_REG_INFO` | - | 获取RDMA信息 | `nvfs_ioctl_get_rdma_reg_info_args_t` | 获取GPU内存的RDMA访问信息 |
| `NVFS_IOCTL_CLEAR_RDMA_REG_INFO` | - | 清除RDMA信息 | `nvfs_ioctl_clear_rdma_reg_info_args_t` | 清除关联的RDMA信息 |

#### 1.3.3 参数联合体定义

```c
typedef union nvfs_ioctl_param_union {
    nvfs_ioctl_ioargs_t ioargs;                        // 读写操作参数
    nvfs_ioctl_map_args_t map_args;                    // 映射操作参数
    nvfs_ioctl_batch_ioargs_t batch_args;              // 批量I/O参数
    nvfs_ioctl_set_rdma_reg_info_args_t rdma_set_reg_info;   // RDMA设置参数
    nvfs_ioctl_get_rdma_reg_info_args_t rdma_get_reg_info;   // RDMA获取参数
    nvfs_ioctl_clear_rdma_reg_info_args_t rdma_clear_reg_info; // RDMA清除参数
} nvfs_ioctl_param_union_t;
```

---

## 二、VA地址转换为GPU物理地址

### 2.1 整体流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        VA → GPU物理地址转换流程                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  用户空间 VA (CPU虚拟地址)                                                    │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1. nvfs_get_mgroup_from_vaddr()                                      │   │
│  │    ├── pin_user_pages_fast()      将用户VA转换为struct page          │   │
│  │    ├── NVFS_PAGE_INDEX(page)      获取页索引                         │   │
│  │    └── nvfs_mgroup_get()          通过索引查找mgroup                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 2. nvfs_mgroup (内存组管理结构)                                       │   │
│  │    ├── cpu_base_vaddr             CPU虚拟地址基址                    │   │
│  │    ├── gpu_info                   GPU信息结构                        │   │
│  │    └── page_table                 GPU页表指针                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 3. nvfs_init_gpu_info()                                              │   │
│  │    ├── 计算GPU虚拟地址范围                                            │   │
│  │    ├── nvidia_p2p_get_pages()     调用NVIDIA P2P API (旧版)          │   │
│  │    └── nvidia_p2p_get_pages_persistent()  (新版持久化API)            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 4. nvidia_p2p_page_table (GPU页表)                                   │   │
│  │    ├── version                    版本号                             │   │
│  │    ├── page_size                  页大小 (64KB)                      │   │
│  │    ├── entries                    条目数                             │   │
│  │    └── pages[i]->physical_address  GPU物理地址 ← 最终目标！           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 5. nvfs_update_gpu_info()                                            │   │
│  │    ├── 构建PRP列表                供NVMe DMA使用                      │   │
│  │    └── 构建Scatter-Gather列表     供其他存储协议使用                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 第一步：用户VA到mgroup的查找

#### 2.2.1 核心函数实现

**源码位置**: [nvfs-mmap.c:L225-L341](file:///e:/code/gds-nvidia-fs/src/nvfs-mmap.c#L225-L341)

```c
/**
 * nvfs_get_mgroup_from_vaddr_internal - 通过CPU虚拟地址查找内存组
 * @cpuvaddr: 用户空间虚拟地址
 * 
 * 返回: 成功返回nvfs_mgroup指针，失败返回NULL
 */
static nvfs_mgroup_ptr_t nvfs_get_mgroup_from_vaddr_internal(u64 cpuvaddr)
{
    struct page *page = NULL;
    int ret;
    unsigned long cur_base_index = 0;
    nvfs_mgroup_ptr_t nvfs_mgroup = NULL;
    nvfs_mgroup_page_ptr_t nvfs_mpage;
    int nvfs_block_count_per_page = (int) PAGE_SIZE / NVFS_BLOCK_SIZE;

    // 1. 参数校验
    if (!cpuvaddr) {
        nvfs_err("%s:%d Invalid shadow buffer address\n", __func__, __LINE__);
        goto out;
    }

    // 2. 检查地址对齐（必须按NVFS_BLOCK_SIZE对齐）
    if (cpuvaddr % NVFS_BLOCK_SIZE) {
        nvfs_err("%s:%d Shadow buffer allocation not aligned\n", 
                __func__, __LINE__);
        goto out;
    }

    // 3. 使用内核API将用户虚拟地址转换为物理页
    // pin_user_pages_fast() 会锁定页面防止被换出
    #ifdef HAVE_PIN_USER_PAGES_FAST
    ret = pin_user_pages_fast(cpuvaddr, 1, 1, &page);
    #else
    ret = get_user_pages_fast(cpuvaddr, 1, 1, &page);
    #endif
    if (ret <= 0) {
        nvfs_err("%s:%d invalid VA %llx ret %d\n",
                __func__, __LINE__, cpuvaddr, ret);
        goto out;
    }

    // 4. 计算页索引
    // NVFS_PAGE_INDEX 获取page的index字段
    // 右移 NVFS_MAX_SHADOW_PAGES_ORDER 得到mgroup的base_index
    cur_base_index = NVFS_PAGE_INDEX(page) >> NVFS_MAX_SHADOW_PAGES_ORDER;

    // 5. 从哈希表中获取对应的mgroup
    nvfs_mgroup = nvfs_mgroup_get(cur_base_index);
    if (nvfs_mgroup == NULL || unlikely(IS_ERR(nvfs_mgroup))) {
        nvfs_err("%s:%d nvfs_mgroup is invalid for index %lu cpuvaddr %llx\n",
                __func__, __LINE__, NVFS_PAGE_INDEX(page), cpuvaddr);
        goto release_page;
    }

    // 6. 验证地址匹配
    if (cpuvaddr != nvfs_mgroup->cpu_base_vaddr) {
        nvfs_err("%s:%d shadow buffer address mismatch %llx vs %llx\n",
                __func__, __LINE__, cpuvaddr, nvfs_mgroup->cpu_base_vaddr);
        goto failed;
    }

    // 7. 验证元数据完整性
    nvfs_mpage = &nvfs_mgroup->nvfs_metadata[
        (NVFS_PAGE_INDEX(page) % NVFS_MAX_SHADOW_PAGES) * nvfs_block_count_per_page];
    if (nvfs_mpage == NULL || 
        nvfs_mpage->nvfs_start_magic != NVFS_START_MAGIC ||
        nvfs_mpage->page != page) {
        nvfs_err("%s:%d found invalid page %p\n", __func__, __LINE__, page);
        goto failed;
    }

    // 8. 释放页面引用（mgroup已持有引用）
    #ifdef HAVE_PIN_USER_PAGES_FAST
    unpin_user_page(page);
    #else
    put_page(page);
    #endif

    return nvfs_mgroup;

failed:
    nvfs_mgroup_put(nvfs_mgroup);
release_page:
    #ifdef HAVE_PIN_USER_PAGES_FAST
    unpin_user_page(page);
    #else
    put_page(page);
    #endif
out:
    return NULL;
}

/**
 * nvfs_get_mgroup_from_vaddr - 公共接口函数
 * @cpuvaddr: 用户空间虚拟地址
 */
nvfs_mgroup_ptr_t nvfs_get_mgroup_from_vaddr(u64 cpuvaddr)
{
    nvfs_mgroup_ptr_t nvfs_mgroup_s;

    // 检查第一页
    nvfs_mgroup_s = nvfs_get_mgroup_from_vaddr_internal(cpuvaddr);

    if (!nvfs_mgroup_s) {
        nvfs_err("%s:%d Invalid vaddr %llx\n", __func__, __LINE__, cpuvaddr);
        goto out;
    }

    return nvfs_mgroup_s;
out:
    return NULL;
}
```

#### 2.2.2 关键技术点

| 技术点 | 说明 |
|--------|------|
| `pin_user_pages_fast()` | Linux内核API，将用户虚拟地址转换为`struct page`并锁定页面 |
| `NVFS_PAGE_INDEX(page)` | 获取page结构体的index字段，用于标识页面 |
| `nvfs_mgroup_get()` | 从哈希表中根据索引查找mgroup |
| 地址对齐检查 | 确保地址按`NVFS_BLOCK_SIZE`（通常64KB）对齐 |
| 魔数验证 | `NVFS_START_MAGIC`用于验证元数据完整性 |

### 2.3 第二步：调用NVIDIA P2P API获取GPU物理地址

#### 2.3.1 核心函数实现

**源码位置**: [nvfs-core.c:L1279-L1411](file:///e:/code/gds-nvidia-fs/src/nvfs-core.c#L1279-L1411)

```c
/**
 * nvfs_init_gpu_info - 初始化GPU信息并获取GPU物理页表
 * @input_param: 输入参数，包含GPU虚拟地址和长度
 * @gpu_info: 输出参数，GPU信息结构
 * @nvfs_mgroup: 关联的内存组
 * 
 * 返回: 0成功，负值失败
 */
int nvfs_init_gpu_info(struct nvfs_gpu_args* input_param,
    struct nvfs_gpu_info* gpu_info, struct nvfs_io_mgroup* nvfs_mgroup)
{
    u64 gpu_virt_start, gpu_virt_end;
    u64 gpu_buf_len;
    u64 rounded_size;
    int ret = 0;
    int i;
    int n_phys_chunks = 1;
    bool is_invalid_page_table_version = false;
    bool is_invalid_page_size = false;

    // 1. 从输入参数获取GPU虚拟地址范围
    gpu_virt_start = input_param->gpuva;
    gpu_buf_len = input_param->gpu_buf_len;
    gpu_virt_end = gpu_virt_start + gpu_buf_len - 1;

    // 2. 按64KB对齐计算大小（GPU页大小）
    rounded_size = round_up((gpu_virt_end - gpu_virt_start + 1),
            GPU_PAGE_SIZE);  // GPU_PAGE_SIZE = 64KB

    // 3. 初始化gpu_info结构
    gpu_info->gpuvaddr = gpu_virt_start;
    gpu_info->gpu_buf_len = gpu_buf_len;
    gpu_info->page_table = NULL;
    gpu_info->n_phys_chunks = 0;

    // 4. 调用NVIDIA P2P API获取GPU页表
    if (nvfs_use_legacy_p2p_allocation) {
        // 旧版API (NVIDIA驱动版本 < 555)
        nvfs_dbg("Invoking p2p_get_pages (0x%lx - 0x%lx) rounded size %lx\n",
                 (unsigned long)gpu_virt_start,
                 (unsigned long)gpu_virt_end, (unsigned long)rounded_size);

        ret = nvfs_nvidia_p2p_get_pages(
                0,                          // p2p_token (未使用)
                0,                          // va_space (未使用)
                gpu_virt_start,             // GPU虚拟地址
                rounded_size,               // 大小
                &gpu_info->page_table,      // 输出：页表指针
                nvfs_get_pages_free_callback, // 回调函数
                nvfs_mgroup                 // 回调数据
            );
        if (ret < 0) {
            nvfs_err("Error ret %d invoking nvidia_p2p_get_pages\n", ret);
            goto error;
        }
    } else {
        // 新版持久化P2P API (NVIDIA驱动版本 >= 555)
        nvfs_dbg("Invoking nvidia_p2p_get_pages_persistent (0x%lx - 0x%lx)\n",
                 (unsigned long)gpu_virt_start, (unsigned long)gpu_virt_end);

        ret = nvfs_nvidia_p2p_get_pages_persistent(
                gpu_virt_start,             // GPU虚拟地址
                rounded_size,               // 大小
                &gpu_info->page_table,      // 输出：页表指针
                0                           // 标志
            );
        if (ret < 0) {
            nvfs_err("Error ret %d invoking nvidia_p2p_get_pages_persistent\n", ret);
            goto error;
        }
    }

    // 5. 打印GPU物理页信息
    nvfs_dbg("GPU page table entries: %d\n", gpu_info->page_table->entries);

    for (i = 0; i < gpu_info->page_table->entries - 1; i++) {
        nvfs_dbg("GPU Physical page[%d]=0x%016llx\n",
            i, gpu_info->page_table->pages[i]->physical_address);

        // 6. 分析物理地址连续性
        // 如果物理地址不连续，需要增加物理块计数
        if ((gpu_info->page_table->pages[i]->physical_address + GPU_PAGE_SIZE) !=
                        gpu_info->page_table->pages[i + 1]->physical_address)
            n_phys_chunks += 1;
        // 或者达到最大连续页数限制
        else if (i > 0 && (i % NVFS_P2P_MAX_CONTIG_GPU_PAGES == 0))
            n_phys_chunks += 1;
    }

    gpu_info->n_phys_chunks = n_phys_chunks;

    // 7. 验证页表版本和页大小
    is_invalid_page_table_version =
        (!NVIDIA_P2P_PAGE_TABLE_VERSION_COMPATIBLE(gpu_info->page_table));
    is_invalid_page_size = (gpu_info->page_table->page_size !=
                    NVIDIA_P2P_PAGE_SIZE_64KB);

    if (is_invalid_page_table_version || is_invalid_page_size) {
        nvfs_err("Invalid page table version or page size\n");
        ret = -EINVAL;
        goto unpin_gpu_pages;
    }

    // 8. 更新GPU统计信息
    nvfs_update_alloc_gpustat(gpu_info);
    nvfs_dbg("GPU pages pinned successfully gpu_info %p\n", gpu_info);

    return 0;

unpin_gpu_pages:
    // 释放GPU页表
    if (gpu_info->use_legacy_p2p_allocation) {
        nvidia_p2p_put_pages(gpu_info->page_table);
    } else {
        nvidia_p2p_put_pages_persistent(gpu_info->page_table);
    }
error:
    return ret;
}
```

#### 2.3.2 NVIDIA P2P API 选择策略

```c
// nvfs-core.c 模块初始化时确定API选择
static int __init nvfs_init(void)
{
    int version = get_nvidia_driver_version();

    // 策略1: 驱动版本 >= 555 使用持久化P2P API
    if (version >= NVIDIA_MIN_DRIVER_FOR_VGPU) {
        nvfs_use_legacy_p2p_allocation = 0;
    }

    // 策略2: x86裸机环境也使用持久化P2P API
    #if defined(CONFIG_X86_64)
    if (nvfs_use_legacy_p2p_allocation == 1 && 
        !cpu_feature_enabled(X86_FEATURE_HYPERVISOR)) {
        nvfs_use_legacy_p2p_allocation = 0;
    }
    #endif
}
```

| API类型 | 函数名 | 适用场景 | 特点 |
|---------|--------|----------|------|
| 旧版API | `nvidia_p2p_get_pages()` | NVIDIA驱动 < 555 | 需要回调函数处理页面失效 |
| 持久化API | `nvidia_p2p_get_pages_persistent()` | NVIDIA驱动 >= 555 或 x86裸机 | 页面持久有效，无需回调 |

### 2.4 NVIDIA P2P 页表结构

#### 2.4.1 数据结构定义

```c
/**
 * NVIDIA P2P 页表结构 (由NVIDIA驱动提供)
 */
struct nvidia_p2p_page_table {
    u32 version;                              // 版本号
    enum nvidia_p2p_page_size_type page_size; // 页大小类型
    u32 entries;                              // 页表条目数
    struct nvidia_p2p_dma_mapping *dma_mapping; // DMA映射信息
    struct nvidia_p2p_page *pages[];          // 页数组（柔性数组）
};

/**
 * NVIDIA P2P 页结构
 */
struct nvidia_p2p_page {
    u64 physical_address;                     // GPU物理地址！
    struct nvidia_p2p_dma_mapping *dma_mapping;
};

/**
 * 页大小类型枚举
 */
enum nvidia_p2p_page_size_type {
    NVIDIA_P2P_PAGE_SIZE_4KB = 0,
    NVIDIA_P2P_PAGE_SIZE_64KB = 1,            // GDS使用64KB
    NVIDIA_P2P_PAGE_SIZE_128KB = 2,
};
```

#### 2.4.2 页表结构示意图

```
nvidia_p2p_page_table
├── version: 1
├── page_size: NVIDIA_P2P_PAGE_SIZE_64KB
├── entries: 16
├── dma_mapping: 指向DMA映射信息
│
└── pages[] (柔性数组)
    ├── [0] ──► nvidia_p2p_page
    │            ├── physical_address: 0x00007FFF00000000  ← GPU物理地址
    │            └── dma_mapping
    │
    ├── [1] ──► nvidia_p2p_page
    │            ├── physical_address: 0x00007FFF00100000
    │            └── dma_mapping
    │
    ├── [2] ──► nvidia_p2p_page
    │            ├── physical_address: 0x00007FFF00200000
    │            └── dma_mapping
    │
    └── ... (更多页)
```

### 2.5 第三步：构建PRP列表供DMA使用

#### 2.5.1 核心函数实现

**源码位置**: [nvfs-core.c:L1562-L1596](file:///e:/code/gds-nvidia-fs/src/nvfs-core.c#L1562-L1596)

```c
/**
 * nvfs_update_gpu_info - 更新GPU信息，构建PRP列表
 * @io: I/O请求结构
 */
void nvfs_update_gpu_info(struct nvfs_io* io)
{
    int i;
    struct nvfs_gpu_info* gpu_info = io->gpu_info;
    struct nvfs_gpu_args* gpu_args = io->gpu_args;

    // 1. 将GPU物理地址填入PRP列表
    if (gpu_info->page_table && (gpu_info->page_table->entries > 1)) {
        for (i = 0; i < gpu_info->page_table->entries; i++) {
            nvfs_dbg("PRP[%d]=0x%016llx\n",
                i, gpu_info->page_table->pages[i]->physical_address);
            
            // PRP列表直接使用GPU物理地址
            // io->io_cmd->prp_list[i] = gpu_info->page_table->pages[i]->physical_address;
        }
    }

    // 2. 设置NVMe命令的P2P相关字段
    if (gpu_info->page_table) {
        io->io_cmd->nvfs_p2p_page_table_version =
            (u32)gpu_info->page_table->version;
        io->io_cmd->nvfs_p2p_page_table_entries =
            (u32)gpu_info->page_table->entries;
        io->io_cmd->nvfs_p2p_page_size_id =
            (u32)gpu_info->page_table->page_size;
        io->io_cmd->nvfs_p2p_page_table_entries_in_use =
            (u32)gpu_info->page_table->entries;
    }
}
```

#### 2.5.2 PRP列表与DMA传输

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        NVMe PRP (Physical Region Page) 机制                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  NVMe命令                                                                    │
│  ├── PRP1: 第一个GPU物理地址 (或PRP列表指针)                                  │
│  └── PRP2: 第二个GPU物理地址 (或PRP列表指针)                                  │
│                                                                             │
│  当数据跨越多个页时：                                                         │
│                                                                             │
│  PRP列表:                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ PRP[0] = GPU物理地址 0x00007FFF00000000                             │   │
│  │ PRP[1] = GPU物理地址 0x00007FFF00100000                             │   │
│  │ PRP[2] = GPU物理地址 0x00007FFF00200000                             │   │
│  │ ...                                                                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  NVMe控制器直接通过PCIe P2P DMA访问这些GPU物理地址                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 三、完整调用链

### 3.1 MAP操作调用链

```
用户空间: ioctl(fd, NVFS_IOCTL_MAP, &args)
              │
              ▼
内核空间: nvfs_ioctl() [nvfs-core.c]
              │
              ├── copy_from_user() 拷贝参数
              │
              ▼
          nvfs_map() [nvfs-core.c]
              │
              ├── nvfs_mgroup_pin_shadow_pages() [nvfs-mmap.c]
              │       │
              │       ├── pin_user_pages_fast() 锁定用户页面
              │       │
              │       ├── nvfs_mgroup_get() 获取/创建mgroup
              │       │
              │       └── 初始化mgroup元数据
              │
              ├── nvfs_init_gpu_info() [nvfs-core.c]
              │       │
              │       ├── 计算GPU虚拟地址范围
              │       │
              │       ├── nvidia_p2p_get_pages() 获取GPU页表
              │       │       │
              │       │       └── 返回 page_table
              │       │               └── pages[i]->physical_address
              │       │
              │       └── 分析物理地址连续性
              │
              └── copy_to_user() 返回结果
```

### 3.2 READ/WRITE操作调用链

```
用户空间: ioctl(fd, NVFS_IOCTL_READ/WRITE, &args)
              │
              ▼
内核空间: nvfs_ioctl() [nvfs-core.c]
              │
              ├── copy_from_user() 拷贝参数
              │
              ▼
          nvfs_io_init() [nvfs-core.c]
              │
              ├── 分配nvfs_io结构
              │
              ├── nvfs_get_mgroup_from_vaddr() 获取mgroup
              │       │
              │       └── pin_user_pages_fast() → nvfs_mgroup_get()
              │
              └── 初始化I/O状态机
              │
              ▼
          nvfs_io_start_op() [nvfs-core.c]
              │
              ├── nvfs_update_gpu_info() 更新GPU信息
              │       │
              │       └── 构建PRP列表
              │
              ├── 提交到存储子系统
              │       │
              │       └── NVMe驱动使用PRP列表进行DMA
              │
              └── 等待I/O完成
              │
              ▼
          copy_to_user() 返回结果
```

---

## 四、关键数据结构关系

### 4.1 数据结构关系图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           nvfs_mgroup (内存组)                               │
├─────────────────────────────────────────────────────────────────────────────┤
│  cpu_base_vaddr        │ CPU虚拟地址基址                                     │
│  nvfs_blocks_count     │ 块数量                                              │
│  gpu_info              │ ──────────────────────────────────────┐            │
│  rdma_info             │ RDMA信息                              │            │
│  ref                   │ 引用计数                              │            │
│  hash_link             │ 哈希表链接                            │            │
│  nvfs_metadata         │ 元数据数组                            │            │
└─────────────────────────────────────────────────────────────────────────────┘
                                                         │
                                                         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           nvfs_gpu_info                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│  gpuvaddr              │ GPU虚拟地址                                         │
│  gpu_buf_len           │ GPU缓冲区长度                                       │
│  page_table            │ ──────────────────────────────────────┐            │
│  n_phys_chunks         │ 物理块数量                            │            │
│  io_state              │ I/O状态 (IO_FREE/IO_INIT/IO_READY...) │            │
│  is_bounce_buffer      │ 是否使用弹跳缓冲区                    │            │
│  use_legacy_p2p_allocation │ 使用旧版P2P API                  │            │
└─────────────────────────────────────────────────────────────────────────────┘
                                                         │
                                                         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        nvidia_p2p_page_table                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│  version               │ 版本号                                              │
│  page_size             │ 页大小 (NVIDIA_P2P_PAGE_SIZE_64KB)                 │
│  entries               │ 条目数                                              │
│  dma_mapping           │ DMA映射信息                                         │
│  pages[0]              │ ────┐                                               │
│  pages[1]              │     │                                               │
│  ...                   │     │                                               │
│  pages[N]              │     │                                               │
└─────────────────────────────────────────────────┼───────────────────────────┘
                                                  │
                                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           nvidia_p2p_page                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│  physical_address      │ GPU物理地址 ← 这是最终目标！                         │
│  dma_mapping           │ DMA映射信息                                         │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 I/O状态机

```c
enum nvfs_io_state {
    IO_FREE = 0,          // 空闲，可分配
    IO_INIT,              // 初始化中
    IO_READY,             // 就绪，可提交I/O
    IO_IN_PROGRESS,       // I/O进行中
    IO_ERROR,             // I/O错误
    IO_DONE               // I/O完成
};
```

```
                    ┌─────────┐
                    │ IO_FREE │
                    └────┬────┘
                         │ nvfs_io_init()
                         ▼
                    ┌─────────┐
                    │ IO_INIT │
                    └────┬────┘
                         │ 初始化完成
                         ▼
                    ┌─────────┐
                    │ IO_READY│
                    └────┬────┘
                         │ nvfs_io_start_op()
                         ▼
                ┌───────────────┐
                │IO_IN_PROGRESS │
                └───────┬───────┘
                        │
            ┌───────────┴───────────┐
            │                       │
            ▼                       ▼
       ┌─────────┐            ┌─────────┐
       │ IO_DONE │            │IO_ERROR │
       └────┬────┘            └────┬────┘
            │                      │
            └──────────┬───────────┘
                       │ 资源释放
                       ▼
                  ┌─────────┐
                  │ IO_FREE │
                  └─────────┘
```

---

## 五、核心API总结

### 5.1 Linux内核API

| API | 功能 | 说明 |
|-----|------|------|
| `pin_user_pages_fast()` | 锁定用户页面 | 将用户VA转换为`struct page`并锁定 |
| `get_user_pages_fast()` | 锁定用户页面（旧版） | 功能同上，用于旧内核 |
| `unpin_user_page()` | 解锁页面 | 释放页面引用 |
| `copy_from_user()` | 从用户空间拷贝 | IOCTL参数拷贝 |
| `copy_to_user()` | 拷贝到用户空间 | IOCTL结果返回 |

### 5.2 NVIDIA P2P API

| API | 功能 | 适用场景 |
|-----|------|----------|
| `nvidia_p2p_get_pages()` | 获取GPU页表（旧版） | NVIDIA驱动 < 555 |
| `nvidia_p2p_get_pages_persistent()` | 获取GPU页表（持久化） | NVIDIA驱动 >= 555 或 x86裸机 |
| `nvidia_p2p_put_pages()` | 释放GPU页表（旧版） | 与get_pages配对 |
| `nvidia_p2p_put_pages_persistent()` | 释放GPU页表（持久化） | 与get_pages_persistent配对 |

---

## 六、技术要点总结

### 6.1 命令接收机制

1. **字符设备接口**：通过 `/dev/nvidia-fs*` 设备文件接收命令
2. **IOCTL分发**：`nvfs_ioctl()` 根据命令号分发到具体处理函数
3. **参数传递**：使用联合体 `nvfs_ioctl_param_union` 传递不同类型的参数
4. **统计跟踪**：每个操作都有对应的统计计数器

### 6.2 地址转换机制

1. **用户VA → struct page**：通过 `pin_user_pages_fast()` 实现
2. **struct page → mgroup**：通过页索引在哈希表中查找
3. **mgroup → GPU页表**：通过 NVIDIA P2P API 获取
4. **GPU页表 → GPU物理地址**：直接从 `page_table->pages[i]->physical_address` 获取

### 6.3 关键优化

1. **持久化P2P**：新版API支持持久化映射，减少重复映射开销
2. **物理连续性分析**：自动检测GPU物理地址连续性，优化DMA传输
3. **引用计数**：mgroup使用引用计数管理生命周期
4. **状态机管理**：I/O状态机确保操作顺序正确

---

## 七、相关源文件

| 文件 | 说明 |
|------|------|
| [nvfs-core.c](file:///e:/code/gds-nvidia-fs/src/nvfs-core.c) | 核心功能实现，IOCTL处理，GPU信息初始化 |
| [nvfs-mmap.c](file:///e:/code/gds-nvidia-fs/src/nvfs-mmap.c) | 内存映射管理，mgroup操作，VA转换 |
| [nvfs-core.h](file:///e:/code/gds-nvidia-fs/src/nvfs-core.h) | 核心头文件，数据结构定义 |
| [nvfs-mmap.h](file:///e:/code/gds-nvidia-fs/src/nvfs-mmap.h) | 内存映射相关定义 |

---

## 八、参考文档

- [NVIDIA GPUDirect Storage 技术白皮书](https://developer.nvidia.com/gpudirect-storage)
- [Linux Kernel DMA-BUF 文档](https://www.kernel.org/doc/html/latest/driver-api/dma-buf.html)
- [NVMe 规范 - PRP 机制](https://nvmexpress.org/specifications/)
