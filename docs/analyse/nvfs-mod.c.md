# nvfs-mod.c 文件解读

## 1. 文件功能描述

`nvfs-mod.c` 是 NVIDIA GPUDirect Storage (GDS) 内核模块的**模块管理组件**，负责动态探测和注册存储供应商模块。该文件实现了与各种存储驱动（如 NVMe、Lustre、BeeGFS 等）的自动发现和注册机制，是 GDS 驱动与底层存储系统之间的桥梁。

### 主要功能：
- **模块自动探测**：扫描预定义的存储供应商模块列表
- **动态注册/注销**：通过内核符号表获取注册函数并调用
- **版本兼容性处理**：支持 v1 和 v2 两种 DMA 操作接口
- **资源管理**：确保符号引用正确获取和释放

---

## 2. 核心逻辑说明

### 2.1 模块探测流程

```
probe_module_list()
├── 遍历 modules_list[] 数组
├── 跳过无效条目（无注册/注销符号）
├── 获取注册函数指针 (__symbol_get)
├── 获取注销函数指针 (__symbol_get)
├── 根据模块类型选择 v1/v2 注册接口
├── 调用注册函数
└── 标记模块状态 (found = true/false)
```

### 2.2 模块注销流程

```
cleanup_module_list()
├── 遍历 modules_list[] 数组
├── 检查模块是否已注册 (found == true)
├── 调用注销函数 (dreg_func)
├── 释放符号引用 (__symbol_put)
└── 重置模块状态
```

### 2.3 版本兼容性

- **v1 接口**：传统 DMA 操作接口，适用于所有内核版本
- **v2 接口**：基于迭代器的新式 DMA 映射接口（内核 6.17+），仅用于 NVMe 模块

---

## 3. 关键函数解析

### 3.1 `probe_module_list()`

**功能**：探测并注册所有可用的存储供应商模块

**参数**：无

**返回值**：
- `0`：成功
- 非零：最后一个处理的模块的错误码

**详细逻辑**：

```c
int probe_module_list(void) {
    // 1. 遍历所有模块条目
    for (i = 0; i < nr_modules(); i++) {
        mod_entry = &modules_list[i];
        
        // 2. 跳过伪模块依赖（无注册/注销符号）
        if (!mod_entry->reg_ksym || !mod_entry->dreg_ksym)
            continue;
        
        // 3. 跳过已发现的模块
        if (mod_entry->found)
            continue;
        
        // 4. 获取注册函数指针
        mod_entry->reg_func = __symbol_get(mod_entry->reg_ksym);
        if (!mod_entry->reg_func) continue;
        
        // 5. 获取注销函数指针（必须成对存在）
        mod_entry->dreg_func = __symbol_get(mod_entry->dreg_ksym);
        if (!mod_entry->dreg_func) {
            // 错误处理：释放已获取的注册符号
            __symbol_put(mod_entry->reg_ksym);
            continue;
        }
        
        // 6. 根据模块类型选择注册接口
        #ifdef HAVE_BLK_RQ_DMA_MAP_ITER_START
        if (strcmp(mod_entry->name, NVFS_PROC_MOD_NVME_KEY) == 0) {
            // NVMe 使用 v2 接口（支持迭代器 API）
            ret = reg_func_v2((struct nvfs_dma_rw_blk_iter_ops *)mod_entry->ops);
        } else {
            // 其他模块使用 v1 接口
            ret = reg_func_v1((struct nvfs_dma_rw_ops *)mod_entry->ops);
        }
        #else
        // 旧内核统一使用 v1 接口
        ret = reg_func_v1((struct nvfs_dma_rw_ops *)mod_entry->ops);
        #endif
        
        // 7. 处理注册结果
        if (ret) {
            // 注册失败：释放所有符号引用
            __symbol_put(mod_entry->dreg_ksym);
            __symbol_put(mod_entry->reg_ksym);
            mod_entry->found = false;
        } else {
            // 注册成功
            mod_entry->found = true;
        }
    }
}
```

### 3.2 `cleanup_module_list()`

**功能**：注销所有已注册的存储供应商模块

**参数**：无

**返回值**：无

**详细逻辑**：

```c
void cleanup_module_list(void) {
    for (i = 0; i < nr_modules(); i++) {
        mod_entry = &modules_list[i];
        
        // 跳过无效条目
        if (!mod_entry->reg_ksym || !mod_entry->dreg_ksym)
            continue;
        
        // 只处理已注册的模块
        if (mod_entry->found) {
            // 1. 调用注销函数
            mod_entry->dreg_func();
            
            // 2. 释放注销符号引用
            __symbol_put(mod_entry->dreg_ksym);
            mod_entry->dreg_func = NULL;
            
            // 3. 释放注册符号引用
            __symbol_put(mod_entry->reg_ksym);
            mod_entry->reg_func = NULL;
            
            // 4. 重置状态
            mod_entry->found = false;
        }
    }
}
```

---

## 4. 参数说明

### 4.1 `struct module_entry` 结构

定义在 `nvfs-dma.h` 中：

| 字段 | 类型 | 说明 |
|------|------|------|
| `is_mod` | `bool` | 是否为真实内核模块（如 scsi_mod 不是模块） |
| `found` | `bool` | 模块是否已成功注册 |
| `name` | `const char *` | 模块名称标识 |
| `version` | `const char *` | 模块版本号 |
| `reg_ksym` | `const char *` | 注册函数在内核符号表中的名称 |
| `reg_func` | `void *` | 注册函数指针（运行时获取） |
| `dreg_ksym` | `const char *` | 注销函数在内核符号表中的名称 |
| `dreg_func` | `nvfs_unregister_dma_ops_fn_t` | 注销函数指针 |
| `ops` | `void *` | DMA 操作结构体指针（v1 或 v2） |

### 4.2 支持的存储供应商模块

| 模块键名 | 注册函数名 | 说明 |
|----------|------------|------|
| `NVFS_PROC_MOD_NVME_KEY` | `nvme_v1/v2_register_nvfs_dma_ops` | NVMe 本地存储 |
| `NVFS_PROC_MOD_NVME_RDMA_KEY` | `nvme_rdma_v1_register_nvfs_dma_ops` | NVMe over RDMA |
| `NVFS_PROC_MOD_SCALEFLUX_CSD_KEY` | `sfxv_v1_register_nvfs_dma_ops` | ScaleFlux CSD |
| `NVFS_PROC_MOD_NVMESH_KEY` | `nvmesh_v1_register_nvfs_dma_ops` | NVMesh |
| `NVFS_PROC_MOD_DDN_LUSTRE_KEY` | `lustre_v1_register_nvfs_dma_ops` | DDN Lustre |
| `NVFS_PROC_MOD_NTAP_BEEGFS_KEY` | `beegfs_v1_register_nvfs_dma_ops` | NetApp BeeGFS |
| `NVFS_PROC_MOD_GPFS_KEY` | `ibm_scale_v1_register_nvfs_dma_ops` | IBM GPFS |
| `NVFS_PROC_MOD_NFS_KEY` | `rpcrdma_register_nvfs_dma_ops` | NFS over RDMA |
| `NVFS_PROC_MOD_SCATEFS_KEY` | `scatefs_register_nvfs_dma_ops` | ScateFS |

### 4.3 内核符号操作宏

| 宏/函数 | 说明 |
|---------|------|
| `__symbol_get(name)` | 从内核符号表获取符号地址，增加引用计数 |
| `__symbol_put(name)` | 释放符号引用，减少引用计数 |
| `HAVE_MODULE_MUTEX` | 编译选项：内核是否导出 module_mutex |
| `HAVE_BLK_RQ_DMA_MAP_ITER_START` | 编译选项：是否支持迭代器式 DMA 映射 |

---

## 5. 使用示例

### 5.1 模块加载时注册

```c
// 在 nvfs-core.c 的 nvfs_open() 中调用
static int nvfs_open(struct inode *inode, struct file *file) {
    mutex_lock(&nvfs_module_mutex);
    
    // 探测并注册所有可用的存储供应商模块
    ret = nvfs_blk_register_dma_ops();
    if (ret < 0) {
        nvfs_err("nvfs modules probe failed with error :%d\n", ret);
        mutex_unlock(&nvfs_module_mutex);
        return ret;
    }
    
    mutex_unlock(&nvfs_module_mutex);
    return 0;
}
```

### 5.2 模块卸载时注销

```c
// 在 nvfs-core.c 的 nvfs_close() 中调用
static int nvfs_close(struct inode *inode, struct file *file) {
    mutex_lock(&nvfs_module_mutex);
    
    // 减少操作计数
    nvfs_put_ops();
    
    // 当没有活跃操作时，注销所有模块
    if (nvfs_count_ops() == 0) {
        nvfs_blk_unregister_dma_ops();
    }
    
    mutex_unlock(&nvfs_module_mutex);
    return 0;
}
```

### 5.3 模块条目定义示例

```c
// 在 nvfs-dma.c 中定义
struct module_entry modules_list[] = {
    {
        .is_mod = 1,
        .found = 0,
        .name = NVFS_PROC_MOD_NVME_KEY,
        .version = 0,
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
    // ... 其他模块条目
};
```

---

## 6. 线程安全

- 使用 `module_mutex` 保护符号表操作（如果内核支持）
- `probe_module_list()` 和 `cleanup_module_list()` 应在持有 `nvfs_module_mutex` 时调用
- 符号引用计数通过 `__symbol_get/__symbol_put` 自动管理

---

## 7. 错误处理

| 错误场景 | 处理方式 |
|----------|----------|
| 符号获取失败 | 跳过该模块，继续探测其他模块 |
| 注册函数获取成功但注销函数失败 | 释放注册函数符号，标记模块未找到 |
| 注册函数调用失败 | 释放所有符号引用，标记模块未找到 |
| 模块已注册 | 跳过，避免重复注册 |

---

## 8. 相关文件

- `nvfs-dma.h`：定义 `struct module_entry` 和 `modules_list[]`
- `nvfs-dma.c`：实现 DMA 操作接口，定义模块条目列表
- `nvfs-core.c`：调用 `nvfs_blk_register_dma_ops()` 和 `nvfs_blk_unregister_dma_ops()`
