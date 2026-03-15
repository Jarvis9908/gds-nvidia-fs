# nvfs-dma.c/h 文件解读

## 1. 文件功能描述

`nvfs-dma.c/h` 是 NVIDIA GPUDirect Storage (GDS) 内核模块的 **DMA 操作抽象层实现文件**，定义了与存储供应商模块交互的接口规范和具体实现。该文件建立了 GDS 驱动与各种存储驱动（NVMe、Lustre、BeeGFS 等）之间的标准化通信协议，是 GPU 直接存储访问的关键桥梁。

### 主要功能：
- **DMA 操作接口定义**：定义存储供应商需要实现的回调函数
- **模块管理结构**：定义模块注册/注销机制
- **版本兼容性**：支持 v1 和 v2 两种 DMA 操作接口
- **Scatter-Gather 列表处理**：将块请求映射到 GPU 内存页
- **迭代器 API 支持**：支持内核 6.17+ 的新式 DMA 映射迭代器

---

## 2. 核心数据结构说明

### 2.1 `struct nvfs_dma_rw_ops` - v1 DMA 操作接口

```c
struct nvfs_dma_rw_ops {
    unsigned long long ft_bmap;  // 功能特性位图

    int (*nvfs_blk_rq_map_sg) (struct request_queue *q,
                               struct request *req, 
                               struct scatterlist *sglist);

    int (*nvfs_dma_map_sg_attrs) (struct device *device,
                                  struct scatterlist *sglist,
                                  int nents,
                                  enum dma_data_direction dma_dir,
                                  unsigned long attrs);

    int (*nvfs_dma_unmap_sg) (struct device *device,
                              struct scatterlist *sglist,
                              int nents,
                              enum dma_data_direction dma_dir);

    bool (*nvfs_is_gpu_page) (struct page *page);

    unsigned int (*nvfs_gpu_index) (struct page *page);

    unsigned int (*nvfs_device_priority) (struct device *dev, unsigned int gpu_index);
    
    int (*nvfs_get_gpu_sglist_rdma_info) (struct scatterlist *sglist,
                                           int nents,
                                           struct nvfs_rdma_info *rdma_infop);
};
```

**字段说明：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `ft_bmap` | `u64` | 功能特性位图，标识支持的特性 |
| `nvfs_blk_rq_map_sg` | 函数指针 | 将块设备请求映射到 scatterlist |
| `nvfs_dma_map_sg_attrs` | 函数指针 | 执行 DMA 映射，支持属性标志 |
| `nvfs_dma_unmap_sg` | 函数指针 | 解除 DMA 映射 |
| `nvfs_is_gpu_page` | 函数指针 | 检查页面是否为 GPU 内存页 |
| `nvfs_gpu_index` | 函数指针 | 获取 GPU 设备索引 |
| `nvfs_device_priority` | 函数指针 | 获取存储设备相对于 GPU 的优先级 |
| `nvfs_get_gpu_sglist_rdma_info` | 函数指针 | 获取 scatterlist 的 RDMA 信息 |

---

### 2.2 `struct nvfs_dma_rw_blk_iter_ops` - v2 DMA 操作接口（内核 6.17+）

```c
#ifdef HAVE_BLK_RQ_DMA_MAP_ITER_START
struct nvfs_dma_rw_blk_iter_ops {
    unsigned long long ft_bmap;  // 功能特性位图

    int (*nvfs_blk_rq_dma_map_iter_start) (struct request *req,
                                           struct device *dma_dev,
                                           struct dma_iova_state *state,
                                           struct blk_dma_iter *iter,
                                           void **cookie);

    int (*nvfs_blk_rq_dma_map_iter_next) (struct request *req,
                                          struct device *dma_dev,
                                          struct dma_iova_state *state,
                                          struct blk_dma_iter *iter);

    int (*nvfs_dma_unmap_page) (struct device *device,
                                void *cookie,
                                dma_addr_t addr,
                                size_t size,
                                enum dma_data_direction dir);

    bool (*nvfs_is_gpu_page) (struct page *page);

    unsigned int (*nvfs_gpu_index) (struct page *page);

    unsigned int (*nvfs_device_priority) (struct device *dev, unsigned int gpu_index);
    
    int (*nvfs_get_gpu_sglist_rdma_info) (struct scatterlist *sglist,
                                           int nents,
                                           struct nvfs_rdma_info *rdma_infop);
};
#endif
```

**与 v1 的区别：**
- 使用迭代器 API 替代传统的 scatterlist 遍历
- `nvfs_blk_rq_dma_map_iter_start/next` 替代 `nvfs_blk_rq_map_sg`
- `nvfs_dma_unmap_page` 替代 `nvfs_dma_unmap_sg`
- 仅用于 NVMe 模块

---

### 2.3 `struct module_entry` - 模块条目

```c
struct module_entry {
    bool is_mod;                      // 是否为真实内核模块
    bool found;                       // 是否已发现
    const char *name;                 // 模块名称
    const char *version;              // 模块版本
    const char *reg_ksym;             // 注册函数符号名
    void *reg_func;                   // 注册函数指针
    const char *dreg_ksym;            // 注销函数符号名
    nvfs_unregister_dma_ops_fn_t dreg_func;  // 注销函数指针
    void *ops;                        // DMA 操作结构体指针
};
```

---

### 2.4 功能特性标志（ft_bmap）

```c
enum ft_bits {
    nvfs_ft_prep_sglist               = 1ULL << 0,  // 支持 scatterlist 准备
    nvfs_ft_map_sglist                = 1ULL << 1,  // 支持 DMA 映射
    nvfs_ft_is_gpu_page               = 1ULL << 2,  // 支持 GPU 页面检测
    nvfs_ft_device_priority           = 1ULL << 3,  // 支持设备优先级
    nvfs_ft_get_gpu_sglist_rdma_info  = 1ULL << 4,  // 支持 RDMA 信息获取
    nvfs_ft_blk_dma_map_iter_start    = 1ULL << 5,  // 支持迭代器开始
    nvfs_ft_blk_dma_map_iter_next     = 1ULL << 6,  // 支持迭代器下一个
};
```

---

## 3. 模块列表定义

### 3.1 支持的存储供应商模块

```c
struct module_entry modules_list[] = {
    // NVMe 本地存储
    {
        .is_mod = 1,
        .found = 0,
        .name = NVFS_PROC_MOD_NVME_KEY,  // "nvme"
        #ifdef HAVE_BLK_RQ_DMA_MAP_ITER_START
        .reg_ksym = "nvme_v2_register_nvfs_dma_ops",
        .dreg_ksym = "nvme_v2_unregister_nvfs_dma_ops",
        .ops = &nvfs_nvme_dma_rw_blk_iter_ops,
        #else
        .reg_ksym = "nvme_v1_register_nvfs_dma_ops",
        .dreg_ksym = "nvme_v1_unregister_nvfs_dma_ops",
        .ops = &nvfs_nvme_dma_rw_ops,
        #endif
    },
    // NVMe over RDMA
    {
        .name = NVFS_PROC_MOD_NVME_RDMA_KEY,  // "nvme_rdma"
        .reg_ksym = "nvme_rdma_v1_register_nvfs_dma_ops",
        .ops = &nvfs_nvme_dma_rw_ops,
    },
    // ScaleFlux CSD
    {
        .name = NVFS_PROC_MOD_SCALEFLUX_CSD_KEY,  // "sfxvdriver"
        .reg_ksym = "sfxv_v1_register_nvfs_dma_ops",
        .ops = &nvfs_sfxv_dma_rw_ops,
    },
    // NVMesh
    {
        .name = NVFS_PROC_MOD_NVMESH_KEY,  // "nvmeib_common"
        .reg_ksym = "nvmesh_v1_register_nvfs_dma_ops",
        .ops = &nvfs_nvmesh_dma_rw_ops,
    },
    // DDN Lustre
    {
        .name = NVFS_PROC_MOD_DDN_LUSTRE_KEY,  // "lnet"
        .reg_ksym = "lustre_v1_register_nvfs_dma_ops",
        .ops = &nvfs_dev_dma_rw_ops,
    },
    // NetApp BeeGFS
    {
        .name = NVFS_PROC_MOD_NTAP_BEEGFS_KEY,  // "beegfs"
        .reg_ksym = "beegfs_v1_register_nvfs_dma_ops",
        .ops = &nvfs_dev_dma_rw_ops,
    },
    // IBM GPFS (需要 RDMA 支持)
    #ifdef NVFS_ENABLE_KERN_RDMA_SUPPORT
    {
        .name = NVFS_PROC_MOD_GPFS_KEY,  // "mmfslinux"
        .reg_ksym = "ibm_scale_v1_register_nvfs_dma_ops",
        .ops = &nvfs_ibm_scale_rdma_ops,
    },
    #endif
    // NFS over RDMA
    {
        .name = NVFS_PROC_MOD_NFS_KEY,  // "rpcrdma"
        .reg_ksym = "rpcrdma_register_nvfs_dma_ops",
        .ops = &nvfs_dev_dma_rw_ops,
    },
    // ScateFS
    {
        .name = NVFS_PROC_MOD_SCATEFS_KEY,  // "scatefs"
        .reg_ksym = "scatefs_register_nvfs_dma_ops",
        .ops = &nvfs_dev_dma_rw_ops,
    },
    // SCSI (非模块)
    {
        .is_mod = 0,
        .name = "scsi_mod",
        .reg_ksym = "scsi_v1_register_dma_scsi_ops",
        .ops = &nvfs_dev_dma_rw_ops,
    },
};
```

---

## 4. 核心函数解析

### 4.1 `nvfs_blk_rq_map_sg()` - 块请求映射到 Scatterlist

**函数原型：**

```c
static int nvfs_blk_rq_map_sg(struct request_queue *q,
                              struct request *req,
                              struct scatterlist *iod_sglist)
```

**功能**：将块设备请求映射到 scatter-gather 列表

**参数：**
- `q`：请求队列
- `req`：块设备请求
- `iod_sglist`：输出 scatterlist

**返回值：**
- 正值：scatterlist 条目数（GPU 页面）
- `0`：无 GPU 页面（CPU 页面）
- `NVFS_IO_ERR`：错误

**实现逻辑：**

```c
static int nvfs_blk_rq_map_sg_internal(struct request_queue *q,
                                       struct request *req,
                                       struct scatterlist *iod_sglist,
                                       bool nvme)
{
    // 1. 验证请求有效性
    if (!nvfs_blk_rq_check(req))
        return 0;
    
    // 2. 遍历请求中的每个段
    rq_for_each_segment(bvec, req, iter) {
        // 检查是否为 GPU 页面
        nvfs_mgroup = nvfs_mgroup_from_page(bvec.bv_page);
        curr_page_gpu = (nvfs_mgroup != NULL);
        
        // 不允许混合 CPU/GPU 页面
        if (found_cpu_page && found_gpu_page)
            return NVFS_IO_ERR;
        
        // 跳过 CPU 页面
        if (found_cpu_page)
            continue;
        
        // 获取 GPU 物理地址
        curr_phys_addr = nvfs_mgroup_get_gpu_physical_address(nvfs_mgroup, bvec.bv_page);
        
        // 检查是否可以合并到当前段
        if (is_gpu_page_contiguous(prev_phys_addr, curr_phys_addr)) {
            sg->length += bvec.bv_len;
        } else {
            // 创建新段
            nsegs++;
            sg_set_page(sg, bvec.bv_page, bvec.bv_len, bvec.bv_offset);
        }
    }
    
    return nsegs;
}
```

---

### 4.2 `nvfs_dma_map_sg_attrs()` - DMA 映射

**函数原型：**

```c
static int nvfs_dma_map_sg_attrs(struct device *device,
                                 struct scatterlist *sglist,
                                 int nents,
                                 enum dma_data_direction dma_dir,
                                 unsigned long attrs)
```

**功能**：为 scatterlist 执行 DMA 映射

**参数：**
- `device`：DMA 设备
- `sglist`：scatterlist
- `nents`：条目数
- `dma_dir`：DMA 方向
- `attrs`：DMA 属性

**返回值：**
- 正值：映射的条目数
- `NVFS_BAD_REQ`：无 GPU 页面
- `NVFS_IO_ERR`：错误

**实现逻辑：**

```c
static int nvfs_dma_map_sg_attrs_internal(struct device *device,
                                          struct scatterlist *sglist,
                                          int nents,
                                          enum dma_data_direction dma_dir,
                                          unsigned long attrs, bool nvme)
{
    for_each_sg(sglist, sg, nents, i) {
        // 获取 DMA 地址
        ret = nvfs_get_dma(to_pci_dev(device), sg_page(sg), &gpu_base_dma, sg->length);
        
        if (ret == NVFS_BAD_REQ) {
            // CPU 页面
            nr_cpu_dma++;
        } else {
            // GPU 页面
            sg_dma_address(sg) = (dma_addr_t)gpu_base_dma + sg->offset;
            sg_dma_len(sg) = sg->length;
            nr_gpu_dma++;
        }
    }
    
    return nr_gpu_dma ? nents : NVFS_BAD_REQ;
}
```

---

### 4.3 `nvfs_blk_rq_dma_map_iter_start()` - 迭代器开始（v2 API）

**函数原型：**

```c
static int nvfs_blk_rq_dma_map_iter_start(struct request *req,
                                          struct device *dma_dev,
                                          struct dma_iova_state *state,
                                          struct blk_dma_iter *iter,
                                          void **cookie)
```

**功能**：启动 DMA 映射迭代（内核 6.17+）

**参数：**
- `req`：块设备请求
- `dma_dev`：DMA 设备
- `state`：DMA IOVA 状态
- `iter`：块 DMA 迭代器
- `cookie`：输出参数，返回 nvfs_mgroup 指针

**返回值：**
- `1`：成功映射一个段
- `0`：无 GPU 页面
- `NVFS_IO_ERR`：错误

**实现逻辑：**

```c
static int nvfs_blk_rq_dma_map_iter_start(struct request *req,
                                          struct device *dma_dev,
                                          struct dma_iova_state *state,
                                          struct blk_dma_iter *iter,
                                          void **cookie)
{
    // 初始化迭代器
    iter->iter.bio = req->bio;
    iter->iter.iter = iter->iter.bio->bi_iter;
    
    // 映射第一个 GPU 段
    return nvfs_map_next_gpu_segment(req, dma_dev, iter, true, cookie);
}
```

---

### 4.4 `nvfs_get_dma()` - 获取 DMA 地址

**函数原型：**

```c
int nvfs_get_dma(void *device, struct page *page, void **gpu_base_dma, int dma_length)
```

**功能**：获取 GPU 页面的 DMA 地址

**参数：**
- `device`：PCI 设备
- `page`：页面指针
- `gpu_base_dma`：输出 DMA 地址
- `dma_length`：DMA 长度

**返回值：**
- `0`：成功
- `NVFS_BAD_REQ`：CPU 页面
- `NVFS_IO_ERR`：错误

---

## 5. 常量定义

```c
#define NVFS_IO_ERR     -1      // I/O 错误
#define NVFS_BAD_REQ    -2      // 非法请求（CPU 页面）

#define NVME_MAX_SEGS   127     // NVMe 最大段数

#define SECTOR_SHIFT    12      // 扇区偏移
#define SECTOR_SIZE     (1 << SECTOR_SHIFT)  // 4KB

// 模块键名
#define NVFS_PROC_MOD_NVME_KEY          "nvme"
#define NVFS_PROC_MOD_NVME_RDMA_KEY     "nvme_rdma"
#define NVFS_PROC_MOD_SCSI_KEY          "scsi_mod"
#define NVFS_PROC_MOD_SCALEFLUX_CSD_KEY "sfxvdriver"
#define NVFS_PROC_MOD_NVMESH_KEY        "nvmeib_common"
#define NVFS_PROC_MOD_DDN_LUSTRE_KEY    "lnet"
#define NVFS_PROC_MOD_NTAP_BEEGFS_KEY   "beegfs"
#define NVFS_PROC_MOD_GPFS_KEY          "mmfslinux"
#define NVFS_PROC_MOD_NFS_KEY           "rpcrdma"
#define NVFS_PROC_MOD_WEKAFS_KEY        "wekafsio"
#define NVFS_PROC_MOD_SCATEFS_KEY       "scatefs"
```

---

## 6. DMA 操作结构实例

### 6.1 通用设备操作

```c
struct nvfs_dma_rw_ops nvfs_dev_dma_rw_ops = {
    .ft_bmap = NVIDIA_FS_SET_FT_ALL,
    .nvfs_blk_rq_map_sg = nvfs_blk_rq_map_sg,
    .nvfs_dma_map_sg_attrs = nvfs_dma_map_sg_attrs,
    .nvfs_dma_unmap_sg = nvfs_dma_unmap_sg,
    .nvfs_is_gpu_page = nvfs_is_gpu_page,
    .nvfs_gpu_index = nvfs_gpu_index,
    .nvfs_device_priority = nvfs_device_priority,
    .nvfs_get_gpu_sglist_rdma_info = nvfs_get_gpu_sglist_rdma_info,
};
```

### 6.2 NVMe 设备操作

```c
struct nvfs_dma_rw_ops nvfs_nvme_dma_rw_ops = {
    .ft_bmap = NVIDIA_FS_SET_FT_ALL,
    .nvfs_blk_rq_map_sg = nvfs_nvme_blk_rq_map_sg,
    .nvfs_dma_map_sg_attrs = nvfs_dma_map_sg_attrs_nvme,
    // ... 其他字段同上
};
```

### 6.3 NVMe v2 迭代器操作（内核 6.17+）

```c
#ifdef HAVE_BLK_RQ_DMA_MAP_ITER_START
struct nvfs_dma_rw_blk_iter_ops nvfs_nvme_dma_rw_blk_iter_ops = {
    .ft_bmap = NVIDIA_FS_SET_BLK_DMA_MAP_ITER_FT_ALL,
    .nvfs_blk_rq_dma_map_iter_start = nvfs_blk_rq_dma_map_iter_start,
    .nvfs_blk_rq_dma_map_iter_next = nvfs_blk_rq_dma_map_iter_next,
    .nvfs_dma_unmap_page = nvfs_dma_unmap_page,
    // ... 其他字段
};
#endif
```

---

## 7. 使用示例

### 7.1 存储供应商模块注册

```c
// 在 NVMe 驱动中
#include "nvfs-dma.h"

// 定义注册函数（导出到内核符号表）
int nvme_v1_register_nvfs_dma_ops(struct nvfs_dma_rw_ops *ops) {
    gds_dma_ops = ops;  // 保存 GDS 提供的操作指针
    return 0;
}
EXPORT_SYMBOL(nvme_v1_register_nvfs_dma_ops);

void nvme_v1_unregister_nvfs_dma_ops(void) {
    gds_dma_ops = NULL;
}
EXPORT_SYMBOL(nvme_v1_unregister_nvfs_dma_ops);
```

### 7.2 使用 DMA 操作接口

```c
// 在块设备驱动中
struct nvfs_dma_rw_ops *ops = gds_dma_ops;

// 检查是否支持 GPU 页面检测
if (ops && NVIDIA_FS_CHECK_FT_GPU_PAGE(ops)) {
    if (ops->nvfs_is_gpu_page(page)) {
        // 这是 GPU 页面
        unsigned int gpu_idx = ops->nvfs_gpu_index(page);
        
        // 执行 DMA 映射
        int nents = ops->nvfs_dma_map_sg_attrs(device, sglist, nents, 
                                                DMA_BIDIRECTIONAL, 0);
    }
}
```

### 7.3 使用迭代器 API（内核 6.17+）

```c
#ifdef HAVE_BLK_RQ_DMA_MAP_ITER_START
struct nvfs_dma_rw_blk_iter_ops *ops = gds_dma_ops_v2;
struct blk_dma_iter iter;
void *cookie;

// 启动迭代
int ret = ops->nvfs_blk_rq_dma_map_iter_start(req, dma_dev, &state, &iter, &cookie);

while (ret > 0) {
    // 使用 iter.addr 和 iter.len
    process_segment(iter.addr, iter.len);
    
    // 获取下一个段
    ret = ops->nvfs_blk_rq_dma_map_iter_next(req, dma_dev, &state, &iter);
}

// 解除映射
ops->nvfs_dma_unmap_page(dma_dev, cookie, iter.addr, iter.len, DMA_TO_DEVICE);
#endif
```

---

## 8. 相关文件

| 文件 | 说明 |
|------|------|
| `nvfs-dma.c` | DMA 操作实现 |
| `nvfs-dma.h` | DMA 接口定义 |
| `nvfs-mod.c` | 模块注册/注销 |
| `nvfs-mmap.c/h` | 内存映射管理 |
| `nvfs-core.c/h` | IOCTL 处理 |

---

## 9. 注意事项

1. **符号导出**：存储供应商模块必须使用 `EXPORT_SYMBOL` 导出注册/注销函数
2. **成对出现**：注册和注销函数必须同时存在
3. **混合页面**：不支持同一请求中混合 CPU 和 GPU 页面
4. **段数限制**：NVMe 最多支持 127 个 scatter-gather 段
5. **迭代器 API**：仅内核 6.17+ 支持 v2 迭代器接口
6. **线程安全**：DMA 操作回调可能在多线程环境下被调用
