# nvfs-proc.c 文件解读

## 1. 文件功能描述

`nvfs-proc.c` 是 NVIDIA GPUDirect Storage (GDS) 内核模块的 **procfs 接口实现文件**，提供了用户空间通过 `/proc` 文件系统与驱动交互的接口。该文件实现了状态查询、配置管理和调试信息导出功能。

### 主要功能：
- **状态查询**：提供驱动运行状态、统计信息的查询接口
- **配置管理**：支持运行时配置参数的调整
- **调试支持**：导出调试信息用于故障排查
- **模块信息**：显示已注册的存储供应商模块信息

---

## 2. 核心功能说明

### 2.1 procfs 入口创建

```c
static int __init nvfs_proc_init(void) {
    struct proc_dir_entry *entry;
    
    // 创建 /proc/nvidia-fs 目录
    nvfs_proc_dir = proc_mkdir("nvidia-fs", NULL);
    if (!nvfs_proc_dir)
        return -ENOMEM;
    
    // 创建状态文件
    entry = proc_create("status", 0444, nvfs_proc_dir, &nvfs_status_fops);
    if (!entry)
        goto err;
    
    // 创建统计文件
    entry = proc_create("stats", 0444, nvfs_proc_dir, &nvfs_stats_fops);
    if (!entry)
        goto err;
    
    // 创建模块信息文件
    entry = proc_create("modules", 0444, nvfs_proc_dir, &nvfs_modules_fops);
    if (!entry)
        goto err;
    
    return 0;
err:
    nvfs_proc_cleanup();
    return -ENOMEM;
}
```

### 2.2 状态查询实现

```c
static int nvfs_status_show(struct seq_file *m, void *v) {
    seq_printf(m, "NVIDIA GDS Driver Status\n");
    seq_printf(m, "========================\n");
    seq_printf(m, "Version: %s\n", NVFS_VERSION_STRING);
    seq_printf(m, "IOCTL Version: %d\n", NVFS_IOCTL_VERSION);
    seq_printf(m, "Active I/O Groups: %d\n", atomic_read(&nvfs_active_mgroups));
    seq_printf(m, "Registered Modules: %d\n", nvfs_count_registered_modules());
    
    #ifdef NVFS_ENABLE_KERN_RDMA_SUPPORT
    seq_printf(m, "RDMA Support: Enabled\n");
    #else
    seq_printf(m, "RDMA Support: Disabled\n");
    #endif
    
    #ifdef NVFS_BATCH_SUPPORT
    seq_printf(m, "Batch I/O Support: Enabled\n");
    #else
    seq_printf(m, "Batch I/O Support: Disabled\n");
    #endif
    
    return 0;
}
```

### 2.3 统计信息显示

```c
static int nvfs_stats_show(struct seq_file *m, void *v) {
    struct nvfs_stat stat;
    
    nvfs_stat_get(&stat);
    
    seq_printf(m, "NVIDIA GDS Statistics\n");
    seq_printf(m, "=====================\n");
    seq_printf(m, "I/O Operations:\n");
    seq_printf(m, "  Read Count:  %lld\n", atomic64_read(&stat.io_read_count));
    seq_printf(m, "  Write Count: %lld\n", atomic64_read(&stat.io_write_count));
    seq_printf(m, "  Read Bytes:  %lld\n", atomic64_read(&stat.io_read_bytes));
    seq_printf(m, "  Write Bytes: %lld\n", atomic64_read(&stat.io_write_bytes));
    seq_printf(m, "  Error Count: %lld\n", atomic64_read(&stat.io_error_count));
    
    seq_printf(m, "\nDMA Operations:\n");
    seq_printf(m, "  Map Count:   %lld\n", atomic64_read(&stat.dma_map_count));
    seq_printf(m, "  Unmap Count: %lld\n", atomic64_read(&stat.dma_unmap_count));
    seq_printf(m, "  Error Count: %lld\n", atomic64_read(&stat.dma_error_count));
    
    return 0;
}
```

### 2.4 模块信息显示

```c
static int nvfs_modules_show(struct seq_file *m, void *v) {
    int i;
    struct module_entry *mod_entry;
    
    seq_printf(m, "Registered Storage Modules\n");
    seq_printf(m, "==========================\n");
    seq_printf(m, "%-20s %-10s %-10s\n", "Name", "Version", "Status");
    seq_printf(m, "%-20s %-10s %-10s\n", "----", "-------", "------");
    
    for (i = 0; i < nr_modules(); i++) {
        mod_entry = &modules_list[i];
        if (!mod_entry->name)
            continue;
        
        seq_printf(m, "%-20s %-10s %-10s\n",
                   mod_entry->name,
                   mod_entry->version ? mod_entry->version : "N/A",
                   mod_entry->found ? "Active" : "Inactive");
    }
    
    return 0;
}
```

---

## 3. procfs 文件列表

| 文件路径 | 权限 | 功能 |
|----------|------|------|
| `/proc/nvidia-fs/status` | 0444 | 显示驱动状态信息 |
| `/proc/nvidia-fs/stats` | 0444 | 显示性能统计信息 |
| `/proc/nvidia-fs/modules` | 0444 | 显示已注册的存储模块 |
| `/proc/nvidia-fs/version` | 0444 | 显示版本信息 |

---

## 4. 使用示例

### 4.1 查看驱动状态

```bash
cat /proc/nvidia-fs/status
```

**输出示例：**
```
NVIDIA GDS Driver Status
========================
Version: 2.19.6
IOCTL Version: 1
Active I/O Groups: 5
Registered Modules: 3
RDMA Support: Enabled
Batch I/O Support: Enabled
```

### 4.2 查看统计信息

```bash
cat /proc/nvidia-fs/stats
```

### 4.3 查看模块信息

```bash
cat /proc/nvidia-fs/modules
```

**输出示例：**
```
Registered Storage Modules
==========================
Name                 Version    Status
----                 -------    ------
nvme                 1.0        Active
lustre               2.14       Active
beegfs               7.2        Inactive
```

---

## 5. 相关文件

- `nvfs-proc.c`：procfs 接口实现
- `nvfs-stat.c`：统计信息收集
- `nvfs-mod.c`：模块管理
