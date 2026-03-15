# nvfs-fault.c/h 文件解读

## 1. 文件功能描述

`nvfs-fault.c/h` 是 NVIDIA GPUDirect Storage (GDS) 内核模块的 **故障注入框架实现文件**，基于 Linux 内核的 `CONFIG_FAULT_INJECTION` 机制，提供了系统化的错误注入能力，用于测试驱动的错误处理路径和健壮性。

### 主要功能：
- **故障注入点定义**：在关键代码路径定义可配置的故障注入点
- **debugfs 接口**：通过 `/sys/kernel/debug/nvfs_inject_fault/` 提供用户空间配置接口
- **细粒度控制**：支持独立的故障率、间隔、次数等参数配置
- **测试覆盖**：覆盖 DMA 映射、页面固定、状态转换等关键路径

---

## 2. 故障注入点定义

### 2.1 故障属性声明

```c
#ifdef CONFIG_FAULT_INJECTION

DECLARE_FAULT_ATTR(nvfs_dma_error);
DECLARE_FAULT_ATTR(nvfs_rw_verify_area_error);
DECLARE_FAULT_ATTR(nvfs_end_fence_get_user_pages_fast_error);
DECLARE_FAULT_ATTR(nvfs_invalid_p2p_get_page);
DECLARE_FAULT_ATTR(nvfs_io_transit_state_fail);
DECLARE_FAULT_ATTR(nvfs_pin_shadow_pages_error);
DECLARE_FAULT_ATTR(nvfs_vm_insert_page_error);

#endif
```

**故障注入点说明：**

| 故障属性 | 说明 | 影响路径 |
|----------|------|----------|
| `nvfs_dma_error` | DMA 映射错误 | DMA 映射操作 |
| `nvfs_rw_verify_area_error` | 读写验证区域错误 | 文件读写验证 |
| `nvfs_end_fence_get_user_pages_fast_error` | 栅栏页面固定错误 | 异步 I/O 栅栏设置 |
| `nvfs_invalid_p2p_get_page` | 无效 P2P 页面获取 | GPU 页面固定 |
| `nvfs_io_transit_state_fail` | I/O 状态转换失败 | I/O 状态机 |
| `nvfs_pin_shadow_pages_error` | Shadow Buffer 页面固定错误 | 内存映射 |
| `nvfs_vm_insert_page_error` | VMA 页面插入错误 | mmap 操作 |

---

## 3. 核心函数解析

### 3.1 `nvfs_init_debugfs()` - 初始化 debugfs 接口

**函数原型：**

```c
void nvfs_init_debugfs(void)
```

**功能**：创建 debugfs 目录和故障注入控制文件

**实现逻辑：**

```c
void nvfs_init_debugfs(void)
{
    // 1. 创建 debugfs 根目录
    dbgfs_root = debugfs_create_dir("nvfs_inject_fault", NULL);
    
    if (!dbgfs_root || IS_ERR(dbgfs_root)) {
        dbgfs_root = NULL;
        nvfs_err("Could not initialise debugfs!\n");
        return;
    }
    
    // 2. 为每个故障属性创建控制文件
    fault_create_debugfs_attr("dma_error", dbgfs_root, &nvfs_dma_error);
    fault_create_debugfs_attr("rw_verify_area_error", dbgfs_root, &nvfs_rw_verify_area_error);
    fault_create_debugfs_attr("end_fence_get_user_pages_fast_error", dbgfs_root, 
                              &nvfs_end_fence_get_user_pages_fast_error);
    fault_create_debugfs_attr("invalid_p2p_get_page", dbgfs_root, &nvfs_invalid_p2p_get_page);
    fault_create_debugfs_attr("io_transit_state_fail", dbgfs_root, &nvfs_io_transit_state_fail);
    fault_create_debugfs_attr("pin_shadow_pages_error", dbgfs_root, &nvfs_pin_shadow_pages_error);
    fault_create_debugfs_attr("vm_insert_page_error", dbgfs_root, &nvfs_vm_insert_page_error);
}
```

---

### 3.2 `nvfs_free_debugfs()` - 清理 debugfs 接口

**函数原型：**

```c
void nvfs_free_debugfs(void)
```

**功能**：移除 debugfs 目录和所有控制文件

**实现逻辑：**

```c
void nvfs_free_debugfs(void)
{
    if (!dbgfs_root)
        return;
    
    debugfs_remove_recursive(dbgfs_root);
    dbgfs_root = NULL;
}
```

---

### 3.3 `nvfs_fault_trigger()` - 触发故障检查

**函数原型：**

```c
static inline bool nvfs_fault_trigger(void *fault)
{
    return should_fail(fault, 1);
}
```

**功能**：检查是否应该注入故障

**参数：**
- `fault`：指向 `struct fault_attr` 的指针

**返回值：**
- `true`：应该注入故障
- `false`：不注入故障

---

## 4. 使用示例

### 4.1 在代码中插入故障注入点

```c
#include "nvfs-fault.h"

// DMA 映射时注入故障
#ifdef CONFIG_FAULT_INJECTION
if (nvfs_fault_trigger(&nvfs_dma_error)) {
    nvfs_err("Fault injected: DMA mapping error\n");
    return -EIO;
}
#endif

// 页面固定时注入故障
#ifdef CONFIG_FAULT_INJECTION
if (nvfs_fault_trigger(&nvfs_pin_shadow_pages_error)) {
    ret = -EFAULT;
} else
#endif
{
    ret = pin_user_pages_fast(cpuvaddr, count, 1, pages);
}
```

### 4.2 通过 debugfs 配置故障率

```bash
# 查看 DMA 错误注入配置
cat /sys/kernel/debug/nvfs_inject_fault/dma_error/probability

# 设置 10% 的故障概率
echo 10 > /sys/kernel/debug/nvfs_inject_fault/dma_error/probability

# 设置故障间隔
echo 100 > /sys/kernel/debug/nvfs_inject_fault/dma_error/interval

# 设置故障次数限制
echo 5 > /sys/kernel/debug/nvfs_inject_fault/dma_fault/times

# 启用故障注入
echo Y > /sys/kernel/debug/nvfs_inject_fault/dma_error/verbose
```

### 4.3 debugfs 文件结构

```
/sys/kernel/debug/nvfs_inject_fault/
├── dma_error/
│   ├── probability    # 故障概率 (0-100)
│   ├── interval       # 故障间隔
│   ├── times          # 故障次数限制
│   ├── space          # 故障空间限制
│   ├── verbose        # 详细输出开关
│   └── task-filter    # 任务过滤
├── rw_verify_area_error/
├── end_fence_get_user_pages_fast_error/
├── invalid_p2p_get_page/
├── io_transit_state_fail/
├── pin_shadow_pages_error/
└── vm_insert_page_error/
```

---

## 5. 编译条件

故障注入功能仅在定义 `CONFIG_FAULT_INJECTION` 时编译：

```c
#ifdef CONFIG_FAULT_INJECTION
// 故障注入代码
#else
#define nvfs_init_debugfs() do{} while (0)
#define nvfs_free_debugfs() do{} while (0)
#endif
```

**启用方式：**
- 内核配置启用 `CONFIG_FAULT_INJECTION`
- 或在 Makefile 中添加 `ccflags-y += -DCONFIG_FAULT_INJECTION`

---

## 6. 故障注入参数说明

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `probability` | 故障发生概率 (0-100%) | 0 |
| `interval` | 故障间隔（调用次数） | 1 |
| `times` | 最大故障次数 (-1 表示无限) | -1 |
| `space` | 故障空间大小 | 0 |
| `verbose` | 详细输出开关 | 0 |
| `task-filter` | 任务过滤开关 | 0 |

---

## 7. 相关文件

| 文件 | 说明 |
|------|------|
| `nvfs-fault.c` | 故障注入实现 |
| `nvfs-fault.h` | 故障注入头文件 |
| `nvfs-core.c` | 使用故障注入点 |
| `nvfs-mmap.c` | 使用故障注入点 |

---

## 8. 注意事项

1. **仅用于测试**：故障注入功能仅用于开发和测试，不应在生产环境启用
2. **性能影响**：启用故障注入会引入额外开销
3. **内核依赖**：需要内核启用 `CONFIG_FAULT_INJECTION` 选项
4. **权限要求**：访问 debugfs 需要 root 权限
5. **状态清理**：测试完成后应重置所有故障注入参数
