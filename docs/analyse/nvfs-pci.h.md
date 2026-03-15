# nvfs-pci.c/h 文件解读

## 1. 文件功能描述

`nvfs-pci.c/h` 是 NVIDIA GPUDirect Storage (GDS) 内核模块的 **PCI 拓扑管理实现文件**，负责 GPU 与存储设备之间的拓扑发现、距离计算和亲和性管理。该文件通过分析 PCI 总线拓扑结构，为 GDS 选择最优的数据传输路径，是实现高性能 GPU 直接存储访问的关键组件。

### 主要功能：
- **PCI 设备发现**：扫描系统中的 GPU、NVMe 和网络设备
- **拓扑距离计算**：计算 GPU 与存储设备之间的 PCI 拓扑距离
- **亲和性管理**：选择最优的 GPU-存储设备配对
- **带宽评估**：评估 PCIe 链路带宽
- **ACS 检测**：检测 ACS（Access Control Services）配置
- **统计信息**：跟踪 P2P DMA 操作的分布情况

---

## 2. 核心数据结构说明

### 2.1 `struct nvfs_rank_data` - 设备排名数据

```c
struct nvfs_rank_data {
    u32 rank;       // 综合排名值（用于排序）
    u16 cross;      // 是否跨根端口（0=同根端口，1=跨根端口）
    u16 pci_dist;   // PCI 拓扑距离
    u16 bw_index;   // 带宽指数
    uint64_t count; // P2P DMA 操作计数
};
```

**字段说明：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `rank` | `u32` | 综合排名值，值越小优先级越高 |
| `cross` | `u16` | 是否跨根端口通信 |
| `pci_dist` | `u16` | PCI 拓扑距离（桥接器数量） |
| `bw_index` | `u16` | 带宽指数（链路宽度 × 链路速度） |
| `count` | `uint64_t` | 该 GPU-存储对的 P2P DMA 操作次数 |

---

### 2.2 全局数据结构

```c
// PCI 路径映射表
static uint64_t gpu_bdf_map[MAX_GPU_DEVS][MAX_PCI_DEPTH];    // GPU 路径
static uint64_t peer_bdf_map[MAX_PEER_DEVS][MAX_PCI_DEPTH];  // 存储设备路径

// 索引哈希表
static uint64_t gpu_info_table[MAX_GPU_DEVS];    // GPU 设备信息表
static uint64_t peer_info_table[MAX_PEER_DEVS];  // 存储设备信息表

// 距离矩阵
static struct nvfs_rank_data gpu_rank_matrix[MAX_GPU_DEVS][MAX_PEER_DEVS];
```

---

## 3. 常量定义

### 3.1 设备数量限制

```c
#define MAX_PEER_DEVS   64U    // 最大对等设备数量（DGX-2: 8 IB 端口）
#define MAX_PCI_DEPTH   16U    // 最大 PCI 拓扑深度（DGX-2: 最大深度 5）
#define MAX_GPU_DEVS    64U    // 最大 GPU 数量（DGX-2: 16 GPU）
```

### 3.2 距离常量

```c
#define PROC_LIMIT_PCI_DISTANCE_COMMONRP  (2 * MAX_PCI_DEPTH)  // 同根端口最大距离
#define BASE_PCI_DISTANCE_CROSSRP         S8_MAX               // 跨根端口基础距离
#define PCI_NULL_DEV_NORP                 (UINT_MAX - 1)       // 无根端口标记
```

### 3.3 设备类检测宏

```c
#define PCI_DEV_GPU(class, vendor) \
    (((vendor) == PCI_VENDOR_ID_NVIDIA) && \
     (((class) == PCI_CLASS_DISPLAY_VGA) || \
      ((class) == PCI_CLASS_DISPLAY_3D)))

#define PCI_DEV_IB(class) \
    (((class) == PCI_CLASS_NETWORK_ETHERNET) || \
     ((class) == PCI_CLASS_NETWORK_INFINIBAND))

#define PCI_DEV_NVME(class) \
    ((class) == PCI_CLASS_STORAGE_EXPRESS)
```

---

## 4. 核心函数解析

### 4.1 `nvfs_pdevinfo()` - 编码 PCI 设备信息

**函数原型：**

```c
static inline uint64_t nvfs_pdevinfo(struct pci_dev *pdev)
```

**功能**：将 PCI 设备信息编码为 64 位整数

**参数：**
- `pdev`：PCI 设备指针

**返回值：**
- 64 位编码值：`[domain:32][bus:8][device:5][function:3]`

**编码格式：**

```
63                                    31         23        15       7      0
+-------------------------------------+----------+---------+--------+------+
|              Domain (32 bits)       | Bus (8)  | Dev (5) | Fn (3) | Info |
+-------------------------------------+----------+---------+--------+------+
                                                                ↑
                                                    高 16 位存储 ACS/类/带宽信息
```

---

### 4.2 `nvfs_fill_gpu2peer_distance_table_once()` - 初始化距离矩阵

**函数原型：**

```c
void nvfs_fill_gpu2peer_distance_table_once(void)
```

**功能**：扫描系统 PCI 设备并计算距离矩阵

**实现逻辑：**

```c
void nvfs_fill_gpu2peer_distance_table_once(void)
{
    // 1. 清空所有表
    memset(gpu_bdf_map, 0, sizeof(gpu_bdf_map));
    memset(peer_bdf_map, 0, sizeof(peer_bdf_map));
    memset(gpu_info_table, 0, sizeof(gpu_info_table));
    memset(peer_info_table, 0, sizeof(peer_info_table));

    // 2. 扫描 GPU 设备（3D 控制器和 VGA 设备）
    __nvfs_find_all_device_paths(gpu_bdf_map, MAX_GPU_DEVS, PCI_CLASS_DISPLAY_3D << 8);
    __nvfs_find_all_device_paths(gpu_bdf_map, MAX_GPU_DEVS, PCI_CLASS_DISPLAY_VGA << 8);

    // 3. 扫描网络设备（InfiniBand 和以太网）
    __nvfs_find_all_device_paths(peer_bdf_map, MAX_PEER_DEVS, PCI_CLASS_NETWORK_INFINIBAND << 8);
    __nvfs_find_all_device_paths(peer_bdf_map, MAX_PEER_DEVS, PCI_CLASS_NETWORK_ETHERNET << 8);

    // 4. 扫描 NVMe 设备
    __nvfs_find_all_device_paths(peer_bdf_map, MAX_PEER_DEVS, PCI_CLASS_STORAGE_EXPRESS);

    // 5. 计算距离矩阵
    nvfs_get_pci_gpu2peer_distance();
}
```

---

### 4.3 `__nvfs_get_gpu2peer_distance()` - 计算 PCI 拓扑距离

**函数原型：**

```c
static unsigned int __nvfs_get_gpu2peer_distance(unsigned int gpu_index, unsigned int peer_index)
```

**功能**：计算 GPU 与存储设备之间的 PCI 拓扑距离

**参数：**
- `gpu_index`：GPU 哈希索引
- `peer_index`：存储设备哈希索引

**返回值：**
- PCI 拓扑距离值

**算法说明：**

```
1. 扫描 GPU 和存储设备的 PCI 路径（从设备到根桥）
2. 查找最低公共祖先桥接器
3. 距离 = GPU 到祖先的深度 + 存储设备到祖先的深度

特殊情况：
- 无公共祖先（跨根端口）：距离 = BASE_PCI_DISTANCE_CROSSRP + 深度
- NUMA 远程：距离加倍
- ACS 启用：增加 ACS 路径长度
```

---

### 4.4 `nvfs_get_gpu2peer_distance()` - 获取排名值

**函数原型：**

```c
unsigned int nvfs_get_gpu2peer_distance(struct device *dev, unsigned int gpu_index)
```

**功能**：获取存储设备相对于指定 GPU 的排名值

**参数：**
- `dev`：存储设备指针
- `gpu_index`：GPU 哈希索引

**返回值：**
- 排名值，值越小优先级越高

**排名计算：**

```c
rank = (MAX_PCIE_BW_INDEX - bw) | (pci_dist << 16U);
```

**排名格式：**

```
31                16                0
+------------------+----------------+
|   PCI Distance   |  BW Index      |
+------------------+----------------+
```

---

### 4.5 `nvfs_pcie_bw_available()` - 获取 PCIe 带宽

**函数原型：**

```c
static u32 nvfs_pcie_bw_available(struct pci_dev *pdev,
                                  enum pci_bus_speed *speed,
                                  enum pcie_link_width *width)
```

**功能**：读取 PCIe 链路速度和宽度

**参数：**
- `pdev`：PCI 设备指针
- `speed`：输出链路速度
- `width`：输出链路宽度

**返回值：**
- 带宽指数 = 链路宽度 × 链路速度索引

**链路速度表：**

```c
const unsigned char nvfs_pcie_link_speed_table[MAX_LNKSPEED_ENTRIES] = {
    PCI_SPEED_UNKNOWN,   /* 0 */
    PCIE_SPEED_2_5GT,    /* 1 - Gen1 */
    PCIE_SPEED_5_0GT,    /* 2 - Gen2 */
    PCIE_SPEED_8_0GT,    /* 3 - Gen3 */
    PCIE_SPEED_16_0GT,   /* 4 - Gen4 */
    PCIE_SPEED_32_0GT,   /* 5 - Gen5 */
    PCIE_SPEED_64_0GT,   /* 6 - Gen6 */
};
```

---

### 4.6 `nvfs_pcie_acs_enabled()` - 检测 ACS 配置

**函数原型：**

```c
static bool nvfs_pcie_acs_enabled(struct pci_dev *pdev)
```

**功能**：检查 PCI 桥接器是否启用了 ACS

**参数：**
- `pdev`：PCI 设备指针

**返回值：**
- `true`：ACS 已启用
- `false`：ACS 未启用

**说明：**
- ACS（Access Control Services）可能影响 P2P DMA 性能
- 启用 ACS 时，P2P 流量可能需要经过根桥

---

## 5. 辅助函数

### 5.1 哈希索引操作

```c
// 创建 GPU 哈希条目
unsigned int nvfs_get_gpu_hash_index(u64 pdevinfo);

// 查找 GPU 哈希条目
uint64_t nvfs_lookup_gpu_hash_index_entry(unsigned int index);

// 查找存储设备哈希条目
uint64_t nvfs_lookup_peer_hash_index_entry(unsigned int index);
```

### 5.2 设备信息编码/解码

```c
// 设置 ACS 标志
void nvfs_pdevinfo_set_acs(uint64_t *pdevinfo);

// 获取 ACS 标志
bool nvfs_pdevinfo_get_acs(uint64_t pdevinfo);

// 设置设备类
void nvfs_pdevinfo_set_class(uint64_t *pdevinfo, unsigned int dev_class);

// 获取设备类名
const char* nvfs_pdevinfo_get_class_name(uint64_t pdevinfo);

// 设置链路速度
void nvfs_pdevinfo_set_link_speed(uint64_t *pdevinfo, enum pci_bus_speed link_speed);

// 获取链路速度
u32 nvfs_pdevinfo_get_link_speed(uint64_t pdevinfo);

// 设置链路宽度
void nvfs_pdevinfo_set_link_width(uint64_t *pdevinfo, enum pcie_link_width link_width);

// 获取链路宽度
u32 nvfs_pdevinfo_get_link_width(uint64_t pdevinfo);

// 获取 NUMA 节点
int nvfs_get_numa_node_from_pdevinfo(uint64_t pdevinfo);
```

---

## 6. 统计函数

### 6.1 `nvfs_update_peer_usage()` - 更新使用统计

```c
void nvfs_update_peer_usage(unsigned int gpu_index, u64 peer_pdevinfo)
```

**功能**：增加 GPU-存储设备对的 P2P DMA 操作计数

### 6.2 `nvfs_aggregate_cross_peer_usage()` - 计算跨节点比例

```c
unsigned int nvfs_aggregate_cross_peer_usage(unsigned int gpu_index)
```

**功能**：计算 GPU 的跨根端口 P2P 操作百分比

### 6.3 `nvfs_reset_peer_affinity_stats()` - 重置统计

```c
void nvfs_reset_peer_affinity_stats(void)
```

**功能**：重置所有 P2P 操作计数

---

## 7. Procfs 接口

### 7.1 `nvfs_peer_distance_show()` - 显示距离矩阵

**输出格式：**

```
gpu             peer            peerrank        p2pdist link    gen     numa    np2p    class
0000:01:00.0    0000:02:00.0    0x00000001      0x0001  0x04    0x03    0x00    100     nvme
```

**字段说明：**

| 字段 | 说明 |
|------|------|
| gpu | GPU PCI 地址 |
| peer | 存储设备 PCI 地址 |
| peerrank | 排名值 |
| p2pdist | PCI 距离 |
| link | 链路宽度 |
| gen | PCIe 代数 |
| numa | NUMA 节点 |
| np2p | P2P 操作次数 |
| class | 设备类名 |

### 7.2 `nvfs_peer_affinity_show()` - 显示亲和性分布

**输出格式：**

```
GPU P2P DMA distribution based on pci-distance
(last column indicates p2p via root complex)
GPU :0000:01:00.0: 100 50 25 10 5 1 0
```

---

## 8. 使用示例

### 8.1 初始化 PCI 拓扑

```c
// 模块初始化时调用
nvfs_fill_gpu2peer_distance_table_once();
```

### 8.2 获取设备排名

```c
struct device *storage_dev = ...;
unsigned int gpu_index = nvfs_get_gpu_hash_index(gpu_pdevinfo);

unsigned int rank = nvfs_get_gpu2peer_distance(storage_dev, gpu_index);
if (rank == UINT_MAX) {
    printk("Device not found in topology\n");
} else {
    printk("Device rank: 0x%08x\n", rank);
}
```

### 8.3 选择最优 GPU

```c
unsigned int best_gpu = 0;
unsigned int min_rank = UINT_MAX;

for (i = 0; i < MAX_GPU_DEVS; i++) {
    if (!nvfs_lookup_gpu_hash_index_entry(i))
        continue;
    
    unsigned int rank = nvfs_get_gpu2peer_distance(storage_dev, i);
    if (rank < min_rank) {
        min_rank = rank;
        best_gpu = i;
    }
}

printk("Best GPU: %u (rank: 0x%08x)\n", best_gpu, min_rank);
```

---

## 9. PCI 拓扑示例

```
Root Complex (Domain 0000)
    |
    +-- Root Port 01:00.0
    |       |
    |       +-- Switch 02:00.0
    |               |
    |               +-- GPU0 (03:00.0)        [距离: 1]
    |               +-- NVMe (04:00.0)        [距离: 1]
    |
    +-- Root Port 81:00.0
            |
            +-- Switch 82:00.0
                    |
                    +-- GPU1 (83:00.0)        [距离: 1]
                    +-- NVMe (84:00.0)        [距离: 1]

GPU0 -> NVMe(04:00.0): 距离 = 1（同交换机）
GPU0 -> NVMe(84:00.0): 距离 = BASE_PCI_DISTANCE_CROSSRP + 3（跨根端口）
```

---

## 10. 相关文件

| 文件 | 说明 |
|------|------|
| `nvfs-pci.c` | PCI 拓扑管理实现 |
| `nvfs-pci.h` | PCI 拓扑管理头文件 |
| `nvfs-dma.c` | 使用设备排名进行 DMA 操作 |
| `nvfs-proc.c` | 通过 procfs 暴露拓扑信息 |

---

## 11. 注意事项

1. **一次性初始化**：`nvfs_fill_gpu2peer_distance_table_once()` 只应在模块加载时调用一次
2. **设备热插拔**：当前实现不支持设备热插拔后的拓扑更新
3. **ACS 影响**：启用 ACS 的桥接器会增加 P2P 距离
4. **NUMA 亲和性**：跨 NUMA 节点的距离会加倍
5. **带宽评估**：带宽指数仅用于排名，不代表实际吞吐量
6. **用户空间依赖**：procfs 输出格式与用户空间库有依赖关系
