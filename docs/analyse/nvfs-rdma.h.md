# nvfs-rdma.c/h 文件解读

## 1. 文件功能描述

`nvfs-rdma.c/h` 是 NVIDIA GPUDirect Storage (GDS) 内核模块的 **RDMA（远程直接内存访问）支持实现文件**，提供了 GPU 内存与 RDMA 网络设备之间的信息交换功能。该模块使 GDS 能够支持通过 RDMA 网络（如 InfiniBand、RoCE）进行 GPU 直接存储访问，主要用于 IBM GPFS 等分布式文件系统。

### 主要功能：
- **RDMA 注册信息设置**：将 RDMA 设备信息关联到 GPU 内存组
- **RDMA 注册信息获取**：获取 GPU 内存的 RDMA 访问信息
- **RDMA 注册信息清除**：清除关联的 RDMA 信息
- **RDMA 段设置**：设置当前 RDMA 段信息
- **版本兼容性检查**：确保 RDMA 版本兼容

---

## 2. 编译条件

RDMA 支持仅在定义 `NVFS_ENABLE_KERN_RDMA_SUPPORT` 时编译：

```c
#ifdef NVFS_ENABLE_KERN_RDMA_SUPPORT
// RDMA 支持代码
#endif
```

**启用方式：**
- 在 Makefile 中不设置 `CONFIG_DISABLE_NVFS_KERN_RDMA_SUPPORT=1`

---

## 3. 核心函数解析

### 3.1 `nvfs_set_rdma_reg_info_to_mgroup()` - 设置 RDMA 注册信息

**函数原型：**

```c
int nvfs_set_rdma_reg_info_to_mgroup(
    nvfs_ioctl_set_rdma_reg_info_args_t* rdma_reg_info_args)
```

**功能**：将 RDMA 设备注册信息关联到 GPU 内存组

**参数：**
| 参数名 | 类型 | 说明 |
|--------|------|------|
| `rdma_reg_info_args` | `nvfs_ioctl_set_rdma_reg_info_args_t*` | RDMA 注册信息参数指针 |

**参数结构体字段：**
- `cpuvaddr`：CPU 虚拟地址，用于查找内存组
- `version`：RDMA 版本号
- `flags`：标志位
- `lid`：子网本地标识符
- `qp_num`：队列对号
- `gid[2]`：16 字节全局标识符
- `dc_key`：DC 密钥
- `nkeys`：rkey 数量（当前仅支持 1）
- `rkey[0]`：远程密钥

**返回值：**
| 返回值 | 说明 |
|--------|------|
| `0` | 成功 |
| `-EINVAL` | 参数无效（内存组不存在、rkey 数量无效、版本不兼容） |

**实现逻辑：**

```c
int nvfs_set_rdma_reg_info_to_mgroup(
    nvfs_ioctl_set_rdma_reg_info_args_t* rdma_reg_info_args)
{
    nvfs_mgroup_ptr_t nvfs_mgroup = NULL;
    struct nvfs_gpu_args* gpu_info;
    struct nvfs_rdma_info* rdma_infop;
    unsigned long shadow_buf_size;
    uint64_t gpuvaddr;
    uint32_t nkeys;
    int ret = -EINVAL;

    // 1. 根据 CPU 虚拟地址获取内存组
    nvfs_mgroup = nvfs_get_mgroup_from_vaddr(rdma_reg_info_args->cpuvaddr);
    if (nvfs_mgroup == NULL || unlikely(IS_ERR(nvfs_mgroup))) {
        nvfs_err("Error: nvfs_mgroup NULL\n");
        return -EINVAL;
    }

    // 2. 验证 rkey 数量（当前仅支持单个 rkey）
    nkeys = rdma_reg_info_args->nkeys;
    if ((nkeys <= 0) || (nkeys > 1)) {
        nvfs_err("Invalid number of rkeys passed: %d\n", nkeys);
        goto error;
    }

    // 3. 版本兼容性检查
    if (rdma_reg_info_args->version < NVFS_RDMA_MIN_SUPPORTED_VERSION) {
        nvfs_err("RDMA registration version %d is not supported by this driver.\n",
                 rdma_reg_info_args->version);
        goto error;
    }

    // 4. 复制 RDMA 设备信息到内存组
    rdma_infop = &nvfs_mgroup->rdma_info;
    rdma_infop->version = rdma_reg_info_args->version;
    rdma_infop->flags   = rdma_reg_info_args->flags;
    rdma_infop->lid     = rdma_reg_info_args->lid;
    rdma_infop->qp_num  = rdma_reg_info_args->qp_num;
    rdma_infop->gid[0]  = rdma_reg_info_args->gid[0];
    rdma_infop->gid[1]  = rdma_reg_info_args->gid[1];
    rdma_infop->dc_key  = rdma_reg_info_args->dc_key;

    // 5. 设置 GPU 内存访问信息
    gpu_info = &nvfs_mgroup->gpu_info;
    gpuvaddr = gpu_info->gpuvaddr;
    rdma_infop->rkey      = rdma_reg_info_args->rkey[0];
    rdma_infop->rem_vaddr = gpuvaddr;
    rdma_infop->size      = gpu_info->gpu_buf_len;

    nvfs_mgroup_put(nvfs_mgroup);
    return 0;

error:
    memset(&nvfs_mgroup->rdma_info, 0, sizeof(struct nvfs_rdma_info));
    nvfs_mgroup_put(nvfs_mgroup);
    return ret;
}
```

---

### 3.2 `nvfs_get_rdma_reg_info_from_mgroup()` - 获取 RDMA 注册信息

**函数原型：**

```c
int nvfs_get_rdma_reg_info_from_mgroup(
    nvfs_ioctl_get_rdma_reg_info_args_t* rdma_reg_info_args)
```

**功能**：从 GPU 内存组获取 RDMA 访问信息

**参数：**
| 参数名 | 类型 | 说明 |
|--------|------|------|
| `rdma_reg_info_args` | `nvfs_ioctl_get_rdma_reg_info_args_t*` | 输入/输出参数，包含 CPU 地址并接收 RDMA 信息 |

**返回值：**
| 返回值 | 说明 |
|--------|------|
| `0` | 成功 |
| `-EINVAL` | 参数无效（内存组不存在） |

**实现逻辑：**

```c
int nvfs_get_rdma_reg_info_from_mgroup(
    nvfs_ioctl_get_rdma_reg_info_args_t* rdma_reg_info_args)
{
    nvfs_mgroup_ptr_t nvfs_mgroup = NULL;
    struct nvfs_rdma_info* rdma_infop = NULL;
    uint64_t shadow_buf_size;

    // 1. 根据 CPU 虚拟地址获取内存组
    nvfs_mgroup = nvfs_get_mgroup_from_vaddr(rdma_reg_info_args->cpuvaddr);
    if (nvfs_mgroup == NULL || unlikely(IS_ERR(nvfs_mgroup))) {
        nvfs_err("Error: nvfs_mgroup NULL\n");
        return -EINVAL;
    }

    // 2. 计算 shadow buffer 大小
    shadow_buf_size = (nvfs_mgroup->nvfs_blocks_count) * NVFS_BLOCK_SIZE;

    // 3. 复制 RDMA 信息到输出参数
    rdma_infop = &nvfs_mgroup->rdma_info;
    rdma_reg_info_args->nvfs_rdma_info = *rdma_infop;

    nvfs_mgroup_put(nvfs_mgroup);
    return 0;
}
```

---

### 3.3 `nvfs_clear_rdma_reg_info_in_mgroup()` - 清除 RDMA 注册信息

**函数原型：**

```c
int nvfs_clear_rdma_reg_info_in_mgroup(
    nvfs_ioctl_clear_rdma_reg_info_args_t* rdma_clear_info_args)
```

**功能**：清除 GPU 内存组关联的 RDMA 信息

**参数：**
| 参数名 | 类型 | 说明 |
|--------|------|------|
| `rdma_clear_info_args` | `nvfs_ioctl_clear_rdma_reg_info_args_t*` | 清除参数（包含 CPU 虚拟地址） |

**返回值：**
| 返回值 | 说明 |
|--------|------|
| `0` | 成功 |
| `-1` | 参数无效（内存组不存在） |

**实现逻辑：**

```c
int nvfs_clear_rdma_reg_info_in_mgroup(
    nvfs_ioctl_clear_rdma_reg_info_args_t* rdma_clear_info_args)
{
    nvfs_mgroup_ptr_t nvfs_mgroup = NULL;

    // 1. 根据 CPU 虚拟地址获取内存组
    nvfs_mgroup = nvfs_get_mgroup_from_vaddr(rdma_clear_info_args->cpuvaddr);
    if (nvfs_mgroup == NULL || unlikely(IS_ERR(nvfs_mgroup))) {
        nvfs_err("%s Error: nvfs_mgroup NULL\n", __func__);
        return -1;
    }

    // 2. 清零 RDMA 信息结构体
    memset(&nvfs_mgroup->rdma_info, 0, sizeof(struct nvfs_rdma_info));

    nvfs_mgroup_put(nvfs_mgroup);
    return 0;
}
```

---

### 3.4 `nvfs_set_curr_rdma_seg_to_mgroup()` - 设置当前 RDMA 段

**函数原型：**

```c
void nvfs_set_curr_rdma_seg_to_mgroup(
    nvfs_mgroup_ptr_t nvfs_mgroup, int rdma_segment)
```

**功能**：设置内存组的当前 RDMA 段信息

**参数：**
| 参数名 | 类型 | 说明 |
|--------|------|------|
| `nvfs_mgroup` | `nvfs_mgroup_ptr_t` | 内存组指针 |
| `rdma_segment` | `int` | RDMA 段号 |

**返回值**：无

**说明**：此函数在头文件中声明，具体实现在其他模块中。

---

## 4. 相关数据结构

### 4.1 `struct nvfs_rdma_info` - RDMA 信息

定义在 `nvfs-mmap.h` 中：

```c
typedef struct nvfs_rdma_info {
    uint8_t  version;       // 版本号
    uint8_t  flags;         // 标志位（bit 0: GID 有效）
    uint16_t lid;           // 子网本地标识符
    uint32_t qp_num;        // QP 号

    // 客户端 Shadow Buffer 信息
    uint64_t rem_vaddr;     // GPU 虚拟地址
    uint32_t size;          // 缓冲区大小
    uint32_t rkey;          // 远程密钥

    // 客户端节点信息
    uint64_t gid[2];        // 16 字节全局标识符
    uint32_t dc_key;        // DC 密钥
} nvfs_rdma_info_t;
```

**字段说明：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `version` | `uint8_t` | RDMA 协议版本号 |
| `flags` | `uint8_t` | 标志位，bit 0 表示 GID 是否有效 |
| `lid` | `uint16_t` | InfiniBand 子网本地标识符 |
| `qp_num` | `uint32_t` | 队列对号，用于 RDMA 通信 |
| `rem_vaddr` | `uint64_t` | 远程 GPU 虚拟地址 |
| `size` | `uint32_t` | 可访问的缓冲区大小 |
| `rkey` | `uint32_t` | 远程密钥，用于 RDMA 访问权限验证 |
| `gid[2]` | `uint64_t[2]` | 16 字节全局标识符（IPv6 格式） |
| `dc_key` | `uint32_t` | DC（Dynamically Connected）密钥 |

---

## 5. IOCTL 命令

| 命令 | 功能 | 参数类型 |
|------|------|----------|
| `NVFS_IOCTL_SET_RDMA_REG_INFO` | 设置 RDMA 注册信息 | `nvfs_ioctl_set_rdma_reg_info_args_t` |
| `NVFS_IOCTL_GET_RDMA_REG_INFO` | 获取 RDMA 注册信息 | `nvfs_ioctl_get_rdma_reg_info_args_t` |
| `NVFS_IOCTL_CLEAR_RDMA_REG_INFO` | 清除 RDMA 注册信息 | `nvfs_ioctl_clear_rdma_reg_info_args_t` |

---

## 6. 测试支持

### 6.1 GPFS 回调测试

当定义 `NVFS_TEST_GPFS_CALLBACK` 时，`nvfs_get_rdma_reg_info_from_mgroup()` 函数会包含额外的测试代码：

```c
#ifdef NVFS_TEST_GPFS_CALLBACK
extern int nvfs_get_gpu_sglist_rdma_info(struct scatterlist *, int, struct nvfs_rdma_info*);
#endif
```

测试代码会：
1. 创建一个 scatterlist
2. 调用 `nvfs_get_gpu_sglist_rdma_info()` 获取 RDMA 信息
3. 验证返回的 RDMA 信息是否正确

---

## 7. 使用场景

### 7.1 IBM GPFS (Spectrum Scale)

GPFS 服务器需要 RDMA 信息来直接访问客户端 GPU 内存：

```
客户端                                    GPFS 服务器
   |                                          |
   |  1. 映射 GPU 缓冲区                       |
   |     ioctl(NVFS_IOCTL_MAP)                |
   |                                          |
   |  2. 设置 RDMA 信息                        |
   |     ioctl(NVFS_IOCTL_SET_RDMA_REG_INFO)  |
   |     [rkey, gid, qp_num, ...]             |
   |                                          |
   |  3. 发送 RDMA 信息到服务器                 |
   |----------------------------------------->|
   |                                          |
   |  4. 服务器使用 rkey 直接访问 GPU 内存      |
   |<-----------------------------------------|
```

---

## 8. 使用示例

### 8.1 设置 RDMA 信息

```c
#include <sys/ioctl.h>

struct nvfs_ioctl_set_rdma_reg_info_args args = {
    .cpuvaddr = shadow_buf_addr,
    .version = 1,
    .flags = 0x01,  // GID 有效
    .lid = 0x0001,
    .qp_num = 0x123456,
    .gid = {0x0001020304050607ULL, 0x08090a0b0c0d0e0fULL},
    .dc_key = 0xabcdef,
    .nkeys = 1,
    .rkey = {0x12345678}
};

int ret = ioctl(fd, NVFS_IOCTL_SET_RDMA_REG_INFO, &args);
if (ret < 0) {
    perror("Failed to set RDMA info");
}
```

### 8.2 获取 RDMA 信息

```c
struct nvfs_ioctl_get_rdma_reg_info_args args = {
    .cpuvaddr = shadow_buf_addr
};

int ret = ioctl(fd, NVFS_IOCTL_GET_RDMA_REG_INFO, &args);
if (ret == 0) {
    printf("rkey: 0x%x, size: %u, vaddr: 0x%llx\n",
           args.nvfs_rdma_info.rkey,
           args.nvfs_rdma_info.size,
           args.nvfs_rdma_info.rem_vaddr);
}
```

### 8.3 清除 RDMA 信息

```c
struct nvfs_ioctl_clear_rdma_reg_info_args args = {
    .cpuvaddr = shadow_buf_addr
};

int ret = ioctl(fd, NVFS_IOCTL_CLEAR_RDMA_REG_INFO, &args);
```

---

## 9. 相关文件

| 文件 | 说明 |
|------|------|
| [nvfs-rdma.c](file:///e:/code/gds-nvidia-fs/src/nvfs-rdma.c) | RDMA 支持实现 |
| [nvfs-rdma.h](file:///e:/code/gds-nvidia-fs/src/nvfs-rdma.h) | RDMA 支持头文件 |
| [nvfs-mmap.h](file:///e:/code/gds-nvidia-fs/src/nvfs-mmap.h) | 定义 `nvfs_rdma_info_t` |
| [nvfs-core.c](file:///e:/code/gds-nvidia-fs/src/nvfs-core.c) | IOCTL 处理入口 |

---

## 10. 注意事项

1. **版本兼容性**：RDMA 版本必须 >= `NVFS_RDMA_MIN_SUPPORTED_VERSION`
2. **rkey 数量**：当前仅支持单个 rkey（`nkeys` 必须为 1）
3. **编译条件**：需要定义 `NVFS_ENABLE_KERN_RDMA_SUPPORT`
4. **内存组引用**：操作完成后必须调用 `nvfs_mgroup_put()` 释放引用
5. **线程安全**：所有操作都是线程安全的
6. **错误处理**：设置失败时会清零整个 `rdma_info` 结构体
