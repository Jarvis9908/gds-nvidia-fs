# nvfs-stat.c/h 文件解读

## 1. 文件功能描述

`nvfs-stat.c/h` 是 NVIDIA GPUDirect Storage (GDS) 内核模块的 **性能统计实现文件**，提供了全面的 I/O 操作统计、性能监控和调试信息收集功能。该模块通过 procfs 接口向用户空间暴露统计信息，用于性能分析、故障排查和容量规划。

### 主要功能：
- **I/O 操作统计**：跟踪读写操作的数量、字节数和错误
- **性能指标计算**：计算吞吐量、延迟等性能指标
- **GPU 内存统计**：跟踪每个 GPU 的内存使用情况
- **错误统计**：记录各类错误的发生次数
- **Procfs 接口**：通过 `/proc/fs/nvfs/stats` 暴露统计信息

---

## 2. 常量定义

```c
#define NVFS_STAT_VERSION   4           // 统计接口版本
#define BYTES_TO_MB(b)      ((b) >> 20ULL)  // 字节转 MiB
#define NVFS_MAX_GPU        16          // 最大 GPU 数量
#define NVFS_MAX_GPU_BITS   ilog2(NVFS_MAX_GPU)  // 哈希表位数
```

---

## 3. 统计计数器定义

### 3.1 读操作统计

```c
atomic64_t nvfs_n_reads;              // 读操作总数
atomic64_t nvfs_n_reads_ok;           // 成功的读操作数
atomic_t nvfs_n_read_err;             // 读错误数
atomic_t nvfs_n_read_iostate_err;     // I/O 状态错误数
atomic64_t nvfs_n_read_bytes;         // 读取的总字节数
atomic_t nvfs_read_throughput;        // 读吞吐量 (MiB/s)
atomic64_t nvfs_read_bytes_per_sec;   // 每秒读取字节数
atomic_t nvfs_read_ops_per_sec;       // 每秒读操作数
atomic64_t nvfs_read_latency_per_sec; // 每秒读延迟总和
atomic_t nvfs_avg_read_latency;       // 平均读延迟 (usec)
```

### 3.2 写操作统计

```c
atomic64_t nvfs_n_writes;             // 写操作总数
atomic64_t nvfs_n_writes_ok;          // 成功的写操作数
atomic_t nvfs_n_write_err;            // 写错误数
atomic_t nvfs_n_write_iostate_err;    // I/O 状态错误数
atomic64_t nvfs_n_write_bytes;        // 写入的总字节数
atomic_t nvfs_write_throughput;       // 写吞吐量 (MiB/s)
atomic64_t nvfs_write_bytes_per_sec;  // 每秒写入字节数
atomic_t nvfs_write_ops_per_sec;      // 每秒写操作数
atomic64_t nvfs_write_latency_per_sec;// 每秒写延迟总和
atomic_t nvfs_avg_write_latency;      // 平均写延迟 (usec)
```

### 3.3 批量 I/O 统计

```c
atomic64_t nvfs_n_batches;            // 批量操作总数
atomic64_t nvfs_n_batches_ok;         // 成功的批量操作数
atomic_t nvfs_n_batch_err;            // 批量操作错误数
atomic_t nvfs_batch_ops_per_sec;      // 每秒批量操作数
atomic64_t nvfs_batch_submit_latency_per_sec;  // 批量提交延迟总和
atomic_t nvfs_batch_submit_avg_latency;        // 平均批量提交延迟 (usec)
atomic64_t prev_batch_submit_avg_latency;      // 上次批量延迟时间戳
```

### 3.4 内存映射统计

```c
atomic64_t nvfs_n_mmap;               // mmap 操作总数
atomic64_t nvfs_n_mmap_ok;            // 成功的 mmap 操作数
atomic_t nvfs_n_mmap_err;             // mmap 错误数
atomic64_t nvfs_n_munmap;             // munmap 操作数
```

### 3.5 BAR1 映射统计

```c
atomic64_t nvfs_n_maps;               // BAR1 映射总数
atomic64_t nvfs_n_maps_ok;            // 成功的映射数
atomic_t nvfs_n_map_err;              // 映射错误数
atomic64_t nvfs_n_free;               // 释放操作数
atomic_t nvfs_n_callbacks;            // 回调次数
atomic64_t nvfs_n_delayed_frees;      // 延迟释放次数
```

### 3.6 错误统计

```c
atomic_t nvfs_n_err_mix_cpu_gpu;      // CPU/GPU 页面混合错误
atomic_t nvfs_n_err_sg_err;           // Scatterlist 扩展错误
atomic_t nvfs_n_err_dma_map;          // DMA 映射错误
atomic_t nvfs_n_err_dma_ref;          // DMA 引用错误
```

### 3.7 稀疏文件统计

```c
atomic64_t nvfs_n_reads_sparse_files;    // 稀疏文件读次数
atomic64_t nvfs_n_reads_sparse_io;       // 稀疏 I/O 次数
atomic64_t nvfs_n_reads_sparse_region;   // 稀疏区域数
atomic64_t nvfs_n_reads_sparse_pages;    // 稀疏页面数
```

### 3.8 页缓存统计

```c
atomic_t nvfs_n_pg_cache;             // 页缓存命中数
atomic_t nvfs_n_pg_cache_fail;        // 页缓存失败数
atomic_t nvfs_n_pg_cache_eio;         // 页缓存 I/O 错误数
```

### 3.9 活跃状态统计

```c
atomic64_t nvfs_n_active_shadow_buf_sz;  // 活跃 Shadow Buffer 大小
atomic_t nvfs_n_op_reads;             // 活跃读操作数
atomic_t nvfs_n_op_writes;            // 活跃写操作数
atomic_t nvfs_n_op_maps;              // 活跃映射数
atomic_t nvfs_n_op_process;           // 活跃进程数
atomic_t nvfs_n_op_batches;           // 活跃批量操作数
```

### 3.10 时间戳记录

```c
atomic_t prev_read_throughput;        // 上次读吞吐量计算时间戳
atomic_t prev_write_throughput;       // 上次写吞吐量计算时间戳
atomic64_t prev_read_latency;         // 上次读延迟计算时间戳
atomic64_t prev_write_latency;        // 上次写延迟计算时间戳
```

---

## 4. GPU 统计结构

### 4.1 `struct nvfs_gpu_stat` - GPU 统计信息

```c
struct nvfs_gpu_stat {
    uint8_t gpu_uuid[16];                 // GPU UUID（16字节）
    struct hlist_node hash_link;          // 哈希表链接节点
    atomic64_t active_bar_memory_pinned;  // 活跃的 BAR1 内存（字节）
    atomic64_t active_bounce_buffer_memory;  // 活跃的弹跳缓冲区（字节）
    atomic64_t max_bar_memory_pinned;     // 历史最大 BAR1 内存使用量（字节）
    unsigned int gpu_index;               // GPU 索引（用于 PCI 拓扑查找）
};
```

**字段说明：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `gpu_uuid` | `uint8_t[16]` | GPU 唯一标识符 |
| `hash_link` | `struct hlist_node` | 哈希表链接，用于快速查找 |
| `active_bar_memory_pinned` | `atomic64_t` | 当前通过 BAR1 映射的 GPU 内存量 |
| `active_bounce_buffer_memory` | `atomic64_t` | 当前使用的弹跳缓冲区内存量 |
| `max_bar_memory_pinned` | `atomic64_t` | 历史峰值 BAR1 内存使用量 |
| `gpu_index` | `unsigned int` | GPU 在系统中的索引 |

---

## 5. 核心函数解析

### 5.1 `nvfs_stat_init()` - 初始化统计子系统

**函数原型：**

```c
void nvfs_stat_init(void)
```

**功能**：初始化统计子系统，包括自旋锁和 GPU 统计哈希表

**实现：**

```c
void nvfs_stat_init(void)
{
    spin_lock_init(&lock);
    hash_init(nvfs_gpu_stat_hash);
}
```

---

### 5.2 `nvfs_stat_destroy()` - 销毁统计子系统

**函数原型：**

```c
void nvfs_stat_destroy(void)
```

**功能**：释放所有 GPU 统计结构内存

**实现：**

```c
void nvfs_stat_destroy(void)
{
    struct hlist_node *tmp;
    struct nvfs_gpu_stat *gpustat;
    int bkt = 0;

    hash_for_each_safe(nvfs_gpu_stat_hash, bkt, tmp, gpustat, hash_link) {
        hash_del(&gpustat->hash_link);
        kfree(gpustat);
    }
}
```

---

### 5.3 `nvfs_update_read_throughput()` - 更新读吞吐量

**函数原型：**

```c
void nvfs_update_read_throughput(unsigned long total_bytes, atomic64_t *stat)
```

**功能**：计算并更新读吞吐量统计

**参数：**
| 参数名 | 类型 | 说明 |
|--------|------|------|
| `total_bytes` | `unsigned long` | 本次传输的字节数 |
| `stat` | `atomic64_t*` | 累计字节数统计器 |

**实现逻辑：**
1. 首次调用时记录当前时间戳
2. 每秒计算一次吞吐量：`throughput = MiB / 秒`
3. 重置累计器并更新时间戳

---

### 5.4 `nvfs_update_read_latency()` - 更新读延迟

**函数原型：**

```c
void nvfs_update_read_latency(unsigned long avg_latency, atomic64_t *stat)
```

**功能**：计算并更新平均读延迟

**参数：**
| 参数名 | 类型 | 说明 |
|--------|------|------|
| `avg_latency` | `unsigned long` | 本次操作的延迟（微秒） |
| `stat` | `atomic64_t*` | 累计延迟统计器 |

**实现逻辑：**
1. 累计延迟和操作数
2. 每秒计算一次平均延迟：`avg = 总延迟 / 操作数`
3. 更新 `nvfs_avg_read_latency`

---

### 5.5 `nvfs_update_write_latency()` - 更新写延迟

**函数原型：**

```c
void nvfs_update_write_latency(unsigned long avg_latency, atomic64_t *stat)
```

**功能**：计算并更新平均写延迟

**参数：** 与 `nvfs_update_read_latency()` 相同

---

### 5.6 `nvfs_update_batch_latency()` - 更新批量提交延迟

**函数原型：**

```c
void nvfs_update_batch_latency(unsigned long avg_latency, atomic64_t *stat)
```

**功能**：计算并更新批量 I/O 提交的平均延迟

**参数：**
| 参数名 | 类型 | 说明 |
|--------|------|------|
| `avg_latency` | `unsigned long` | 本次批量提交的延迟（微秒） |
| `stat` | `atomic64_t*` | 累计延迟统计器 |

**实现逻辑：**

```c
void nvfs_update_batch_latency(unsigned long avg_latency, atomic64_t *stat)
{
    int delta;
    int average_latency;

    if (atomic64_read(&prev_batch_submit_avg_latency) == 0) {
        atomic64_set(&prev_batch_submit_avg_latency, ktime_to_us(ktime_get()));
        nvfs_stat(&nvfs_batch_ops_per_sec);
        nvfs_stat64_add(avg_latency, stat);
        return;
    }

    delta = ktime_to_us(ktime_get()) - atomic64_read(&prev_batch_submit_avg_latency);

    if (delta > USEC_PER_SEC) {
        nvfs_stat64_add(avg_latency, stat);
        nvfs_stat(&nvfs_batch_ops_per_sec);

        average_latency = div64_safe(atomic64_read(stat),
                                     (unsigned long) atomic_read(&nvfs_batch_ops_per_sec));
        atomic_set(&nvfs_batch_submit_avg_latency, average_latency);
        nvfs_stat64_reset(stat);
        nvfs_stat_reset(&nvfs_batch_ops_per_sec);

        atomic64_set(&prev_batch_submit_avg_latency, ktime_to_us(ktime_get()));
    } else {
        nvfs_stat64_add(avg_latency, stat);
        nvfs_stat(&nvfs_batch_ops_per_sec);
    }
}
```

---

### 5.7 `nvfs_update_write_throughput()` - 更新写吞吐量

**函数原型：**

```c
void nvfs_update_write_throughput(unsigned long total_bytes, atomic64_t *stat)
```

**功能**：计算并更新写吞吐量统计

---

### 5.8 `nvfs_update_alloc_gpustat()` - 更新 GPU 分配统计

**函数原型：**

```c
void nvfs_update_alloc_gpustat(struct nvfs_gpu_args *gpuinfo)
```

**功能**：更新 GPU 内存分配统计

**参数：**
| 参数名 | 类型 | 说明 |
|--------|------|------|
| `gpuinfo` | `struct nvfs_gpu_args*` | GPU 参数信息 |

**实现逻辑：**

```c
void nvfs_update_alloc_gpustat(struct nvfs_gpu_args *gpuinfo)
{
    struct nvfs_gpu_stat *gpustat;
    uint64_t gpu_uuid_hash;
    uint64_t active_memory;

    if (gpuinfo->page_table == NULL || gpuinfo->page_table->gpu_uuid == NULL)
        return;

    gpu_uuid_hash = *(uint64_t *)gpuinfo->page_table->gpu_uuid;
    spin_lock(&lock);
    gpustat = nvfs_get_gpustat_unlocked(gpu_uuid_hash);
    if (gpustat == NULL) {
        // 首次遇到此 GPU，创建新的统计结构
        gpustat = kzalloc(sizeof(struct nvfs_gpu_stat), GFP_KERNEL);
        if (!gpustat) {
            nvfs_err("Failed to allocated memory\n");
            spin_unlock(&lock);
            return;
        }
        atomic64_set(&gpustat->max_bar_memory_pinned, 0);
        memcpy(gpustat->gpu_uuid, gpuinfo->page_table->gpu_uuid, 16);
        hash_add_rcu(nvfs_gpu_stat_hash, &gpustat->hash_link, gpu_uuid_hash);
        gpustat->gpu_index = gpuinfo->gpu_hash_index;
    }
    spin_unlock(&lock);

    // 根据缓冲区类型更新对应统计
    if (gpuinfo->is_bounce_buffer) {
        nvfs_stat64_add(gpuinfo->gpu_buf_len, &gpustat->active_bounce_buffer_memory);
    } else {
        nvfs_stat64_add(gpuinfo->gpu_buf_len, &gpustat->active_bar_memory_pinned);
    }

    // 更新历史峰值
    active_memory = atomic64_read(&gpustat->active_bar_memory_pinned);
    if (active_memory > atomic64_read(&gpustat->max_bar_memory_pinned))
        atomic64_set(&gpustat->max_bar_memory_pinned, active_memory);
}
```

---

### 5.9 `nvfs_update_free_gpustat()` - 更新 GPU 释放统计

**函数原型：**

```c
void nvfs_update_free_gpustat(struct nvfs_gpu_args *gpuinfo)
```

**功能**：更新 GPU 内存释放统计

**参数：**
| 参数名 | 类型 | 说明 |
|--------|------|------|
| `gpuinfo` | `struct nvfs_gpu_args*` | GPU 参数信息 |

---

### 5.10 `nvfs_stats_show()` - 显示统计信息

**函数原型：**

```c
static int nvfs_stats_show(struct seq_file *m, void *v)
```

**功能**：生成并输出统计信息到 procfs

**输出内容：**
- GDS 版本信息
- NVFS 统计版本
- 驱动版本
- Mellanox PeerDirect 支持状态
- IO 统计开关状态
- 日志级别
- Shadow Buffer 使用量
- 批量 I/O 统计
- 读/写统计
- 稀疏文件统计
- 内存映射统计
- BAR1 映射统计
- 错误统计
- 各 GPU 详细信息

---

### 5.11 `nvfs_stats_reset()` - 重置统计计数器

**函数原型：**

```c
static int nvfs_stats_reset(void)
```

**功能**：重置所有累计统计计数器（不包括自管理计数器）

---

## 6. 统计辅助宏和内联函数

### 6.1 基本操作

```c
// 增加 64 位计数器
static inline void nvfs_stat64(atomic64_t *stat) {
    atomic64_inc(stat);
}

// 增加 32 位计数器
static inline void nvfs_stat(atomic_t *stat) {
    atomic_inc(stat);
}

// 增加指定值
static inline void nvfs_stat64_add(long i, atomic64_t *stat) {
    atomic64_add(i, stat);
}

// 减少指定值
static inline void nvfs_stat64_sub(long i, atomic64_t *stat) {
    atomic64_sub(i, stat);
}

// 重置 32 位计数器
static inline void nvfs_stat_reset(atomic_t *stat) {
    atomic_set(stat, 0);
}

// 重置 64 位计数器
static inline void nvfs_stat64_reset(atomic64_t *stat) {
    atomic64_set(stat, 0);
}
```

### 6.2 辅助函数

```c
// 安全除法（避免除零）
static inline unsigned long div64_safe(unsigned long sum, unsigned long nr)
{
    return nr ? div64_ul(sum, nr) : 0;
}

// 获取当前 jiffies
static inline u64 jiffies_now(void)
{
    return get_jiffies_64();
}

// 计算时间差（纳秒）
static inline s64 ktime_ns_delta(const ktime_t later, const ktime_t earlier)
{
    return ktime_to_ns(ktime_sub(later, earlier));
}
```

### 6.3 条件编译

```c
#ifdef CONFIG_NVFS_STATS
// 统计代码
#else
#define nvfs_stat(stat) do {} while (0)
#define nvfs_stat64(stat) do {} while (0)
#define nvfs_stat64_add(x, stat) do {} while (0)
#define nvfs_stat64_sub(x, stat) do {} while (0)
#define nvfs_stat_reset(stat) do {} while (0)
#define nvfs_stat64_reset(stat) do {} while (0)
// ... 其他空操作宏
#endif
```

---

## 7. Procfs 接口

### 7.1 统计文件

**路径**：`/proc/fs/nvfs/stats`

**权限**：`0444`（读）/ `0200`写，用于重置）

**文件操作结构：**

```c
#ifdef HAVE_STRUCT_PROC_OPS
const struct proc_ops nvfs_stats_fops = {
    .proc_open      = nvfs_stats_open,
    .proc_read      = seq_read,
    .proc_write     = nvfs_stats_clear,
    .proc_lseek     = seq_lseek,
    .proc_release   = single_release,
};
#else
const struct file_operations nvfs_stats_fops = {
    .open           = nvfs_stats_open,
    .read           = seq_read,
    .write          = nvfs_stats_clear,
    .llseek         = seq_lseek,
    .release        = single_release,
};
#endif
```

### 7.2 输出示例

```
GDS Version: 2.19.6
NVFS statistics(ver: 4.0)
NVFS Driver(version: 2.19.6)
Mellanox PeerDirect Supported: True
IO stats: Enabled, peer IO stats: Enabled
Logging level: info

Active Shadow-Buffer (MiB): 1024
Active Process: 1

Batches                         : n=100 ok=98 err=2 Avg-Submit-Latency(usec)=50
Reads                           : n=1000 ok=995 err=5 readMiB=4096 io_state_err=2
Reads                           : Bandwidth(MiB/s)=1024 Avg-Latency(usec)=100
Sparse Reads                    : n=10 io=15 holes=20 pages=100
Writes                          : n=500 ok=498 err=2 writeMiB=2048 io_state_err=1 pg-cache=50 pg-cache-fail=0 pg-cache-eio=0
Writes                          : Bandwidth(MiB/s)=512 Avg-Latency(usec)=150
Mmap                            : n=10 ok=10 err=0 munmap=5
Bar1-map                        : n=20 ok=20 err=0 free=10 callbacks=15 active=10 delay-frees=5
Error                           : cpu-gpu-pages=0 sg-ext=0 dma-map=2 dma-ref=0
Ops                             : Read=5 Write=3 BatchIO=2

GPU 0000:01:00.0 uuid:... : Registered_MiB=1024 Cache_MiB=0 max_pinned_MiB=1024 cross_root_port(%)=10
```

### 7.3 重置统计

```bash
# 重置所有统计计数器
echo reset > /proc/fs/nvfs/stats
```

---

## 8. 使用示例

### 8.1 在代码中记录统计

```c
#include "nvfs-stat.h"

// 记录读操作
nvfs_stat64(&nvfs_n_reads);
nvfs_stat64_add(bytes_read, &nvfs_n_read_bytes);

// 记录错误
nvfs_stat(&nvfs_n_read_err);

// 更新吞吐量
nvfs_update_read_throughput(bytes_read, &nvfs_read_bytes_per_sec);

// 更新延迟
nvfs_update_read_latency(latency_us, &nvfs_read_latency_per_sec);

// 更新批量延迟
nvfs_update_batch_latency(batch_latency_us, &nvfs_batch_submit_latency_per_sec);
```

### 8.2 查看统计信息

```bash
# 查看完整统计
cat /proc/fs/nvfs/stats

# 查看特定 GPU 信息
cat /proc/fs/nvfs/stats | grep "GPU 0000:01:00.0"

# 查看吞吐量
cat /proc/fs/nvfs/stats | grep "Bandwidth"

# 查看错误统计
cat /proc/fs/nvfs/stats | grep "Error"
```

---

## 9. 内部数据结构

### 9.1 GPU 统计哈希表

```c
static DEFINE_HASHTABLE(nvfs_gpu_stat_hash, NVFS_MAX_GPU_BITS);
static spinlock_t lock ____cacheline_aligned;
```

- 使用 RCU 保护的哈希表存储 GPU 统计
- 哈希键为 GPU UUID 的前 8 字节
- 自旋锁用于保护新增操作

---

## 10. 相关文件

| 文件 | 说明 |
|------|------|
| [nvfs-stat.c](file:///e:/code/gds-nvidia-fs/src/nvfs-stat.c) | 统计实现 |
| [nvfs-stat.h](file:///e:/code/gds-nvidia-fs/src/nvfs-stat.h) | 统计头文件 |
| [nvfs-proc.c](file:///e:/code/gds-nvidia-fs/src/nvfs-proc.c) | Procfs 接口 |
| [nvfs-core.c](file:///e:/code/gds-nvidia-fs/src/nvfs-core.c) | 调用统计函数 |
| [nvfs-pci.c](file:///e:/code/gds-nvidia-fs/src/nvfs-pci.c) | peer 统计 |

---

## 11. 注意事项

1. **编译条件**：需要定义 `CONFIG_NVFS_STATS` 以启用统计功能
2. **性能影响**：启用统计会引入少量原子操作开销
3. **原子操作**：所有计数器使用原子操作，保证线程安全
4. **重置操作**：写入任意内容到 stats 文件会重置所有累计计数器
5. **版本兼容性**：统计输出格式与用户空间库有依赖关系
6. **内存管理**：GPU 统计结构在模块卸载时自动释放
7. **RCU 保护**：GPU 统计哈希表使用 RCU 保护读取操作
