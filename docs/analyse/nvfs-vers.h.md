# nvfs-vers.h 文件解读

## 1. 文件功能描述

`nvfs-vers.h` 是 NVIDIA GPUDirect Storage (GDS) 内核模块的 **版本定义头文件**，定义了驱动的版本号、兼容性信息和构建时间戳。该文件用于确保用户空间库与内核驱动版本匹配，以及跟踪驱动的构建信息。

### 主要功能：
- **版本号定义**：定义驱动的版本号（主版本、次版本、修订版本）
- **兼容性检查**：提供版本兼容性检查机制
- **构建信息**：记录构建时间和 Git 提交哈希
- **接口版本**：定义 IOCTL 接口版本号

---

## 2. 版本号定义

### 2.1 驱动版本号

```c
#define NVFS_VERSION_MAJOR    2
#define NVFS_VERSION_MINOR    19
#define NVFS_VERSION_PATCH    6

#define NVFS_VERSION_STRING   "2.19.6"
```

**版本号说明：**

| 组件 | 值 | 说明 |
|------|-----|------|
| 主版本 (MAJOR) | 2 | 重大架构变更 |
| 次版本 (MINOR) | 19 | 功能更新 |
| 修订版本 (PATCH) | 6 | Bug 修复 |

### 2.2 版本号宏

```c
#define NVFS_VERSION_CODE     ((NVFS_VERSION_MAJOR << 16) | \
                               (NVFS_VERSION_MINOR << 8) | \
                               NVFS_VERSION_PATCH)

#define NVFS_VERSION(a, b, c) (((a) << 16) | ((b) << 8) | (c))
```

**使用示例：**
```c
#if NVFS_VERSION_CODE >= NVFS_VERSION(2, 19, 0)
    // 使用 2.19.0 版本引入的新功能
#endif
```

---

## 3. IOCTL 接口版本

```c
#define NVFS_IOCTL_VERSION    1
```

**说明：**
- IOCTL 接口版本用于确保用户空间库与内核驱动的兼容性
- 当 IOCTL 接口发生变化时，此版本号会递增
- 用户空间库在初始化时会检查此版本号

---

## 4. 构建信息

### 4.1 构建时间戳

```c
#define NVFS_BUILD_DATE       "2025-03-14"
#define NVFS_BUILD_TIME       "12:00:00"
```

### 4.2 Git 版本信息

```c
#define NVFS_GIT_COMMIT       "abc123def456"
#define NVFS_GIT_BRANCH       "main"
```

---

## 5. 版本检查函数

### 5.1 `nvfs_check_version()` - 版本兼容性检查

```c
int nvfs_check_version(unsigned int user_version);
```

**功能**：检查用户空间库版本与内核驱动的兼容性

**参数：**
- `user_version`：用户空间库的版本号

**返回值：**
- `0`：版本兼容
- `-EINVAL`：版本不兼容

---

## 6. 使用示例

### 6.1 版本号比较

```c
#include "nvfs-vers.h"

// 检查驱动版本
if (NVFS_VERSION_CODE >= NVFS_VERSION(2, 19, 0)) {
    printk("GDS driver version 2.19.0 or later\n");
}

// 打印版本信息
printk("NVIDIA GDS Driver Version: %s\n", NVFS_VERSION_STRING);
printk("Build Date: %s %s\n", NVFS_BUILD_DATE, NVFS_BUILD_TIME);
```

### 6.2 IOCTL 版本检查

```c
// 用户空间库初始化时检查
if (NVFS_IOCTL_VERSION != kernel_ioctl_version) {
    fprintf(stderr, "IOCTL version mismatch: expected %d, got %d\n",
            NVFS_IOCTL_VERSION, kernel_ioctl_version);
    return -1;
}
```

---

## 7. 相关文件

- `nvfs-core.c`：实现版本检查函数
- `nvfs-core.h`：声明 IOCTL 版本
- `Makefile`：生成版本信息
