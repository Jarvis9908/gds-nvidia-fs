# nvfs-p2p.h 文件解读

## 1. 文件功能描述

`nvfs-p2p.h` 是 NVIDIA GPUDirect Storage (GDS) 内核模块的 **P2P（点对点）内存访问定义头文件**，定义了与 NVIDIA GPU P2P 机制相关的常量和宏。该文件是 GDS 驱动实现 GPU 与存储设备直接内存访问的基础。

### 主要功能：
- **P2P 内存类型定义**：定义 GPU 内存页类型
- **地址转换**：GPU 虚拟地址与物理地址转换
- **页表管理**：GPU 页表相关常量
- **兼容性支持**：不同 NVIDIA 驱动版本的兼容性处理

---

## 2. 常量定义

### 2.1 P2P 内存类型

```c
#define NVFS_P2P_PAGE_TYPE_PCI      0x01    // PCI 内存页
#define NVFS_P2P_PAGE_TYPE_PCI_BAR  0x02    // PCI BAR 内存页
#define NVFS_P2P_PAGE_TYPE_SYSMEM   0x03    // 系统内存页
```

### 2.2 GPU 页大小

```c
#define NVFS_GPU_PAGE_SHIFT         16      // GPU 页大小：64KB
#define NVFS_GPU_PAGE_SIZE          (1 << NVFS_GPU_PAGE_SHIFT)
#define NVFS_GPU_PAGE_MASK          (~(NVFS_GPU_PAGE_SIZE - 1))
```

### 2.3 BAR1 相关常量

```c
#define NVFS_BAR1_MAP_SIZE          (256ULL << 20)  // BAR1 映射大小：256MB
#define NVFS_BAR1_PAGE_SHIFT        16              // BAR1 页大小：64KB
#define NVFS_BAR1_PAGE_SIZE         (1 << NVFS_BAR1_PAGE_SHIFT)
```

---

## 3. 辅助宏

### 3.1 GPU 地址对齐

```c
#define NVFS_GPU_ADDR_ALIGN(addr) \
    (((addr) + NVFS_GPU_PAGE_SIZE - 1) & NVFS_GPU_PAGE_MASK)

#define NVFS_GPU_ADDR_OFFSET(addr) \
    ((addr) & (NVFS_GPU_PAGE_SIZE - 1))
```

### 3.2 页数计算

```c
#define NVFS_GPU_PAGES(size) \
    (((size) + NVFS_GPU_PAGE_SIZE - 1) >> NVFS_GPU_PAGE_SHIFT)
```

---

## 4. 相关文件

- `nv-p2p.h`：NVIDIA 驱动提供的 P2P 头文件
- `nvfs-mmap.c`：使用 P2P 定义进行内存映射
- `nvfs-dma.c`：使用 P2P 定义进行 DMA 操作
