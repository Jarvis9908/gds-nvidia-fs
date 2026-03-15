# Makefile 文件解读

## 1. 文件功能描述

`Makefile` 是 NVIDIA GPUDirect Storage (GDS) 内核模块的 **构建配置文件**，定义了驱动的编译规则、依赖关系和安装流程。该文件使用 Linux 内核的 kbuild 系统进行构建。

### 主要功能：
- **模块编译**：定义内核模块的编译规则
- **依赖管理**：管理源文件和头文件的依赖关系
- **版本检测**：自动检测 NVIDIA 驱动版本和内核版本
- **平台适配**：支持 x86_64 和 ARM64 架构
- **功能配置**：支持可选功能（RDMA、批量 I/O、统计等）的启用/禁用

---

## 2. 变量定义

### 2.1 内核版本和路径

```makefile
KVER ?= $(shell uname -r)           # 内核版本，默认为当前运行内核
MODULES_DIR := /lib/modules/$(KVER)
KDIR := $(MODULES_DIR)/build        # 内核构建目录
MODULE_DESTDIR := $(MODULES_DIR)/extra/  # 模块安装目录
```

### 2.2 NVIDIA 驱动检测

```makefile
# 获取 NVIDIA 驱动版本
nv_version=$(shell /sbin/modinfo -F version -k $(KVER) nvidia 2>/dev/null)
nv_sources=$(shell /bin/ls -d /usr/src/nvidia-$(nv_version)/ 2>/dev/null)

# 查找 NVIDIA 驱动头文件路径
ifneq ($(shell test -d "$(nv_sources)" && echo "true" || echo "" ),)
    NVIDIA_SRC_DIR ?= $(shell find "$(nv_sources)" -name "nv-p2p.h"|head -1|xargs dirname || echo "NVIDIA_DRIVER_MISSING")
else
    NVIDIA_SRC_DIR ?= $(shell find /usr/src/nvidia-* -name "nv-p2p.h"|head -1|xargs dirname || echo "NVIDIA_DRIVER_MISSING")
endif
```

**说明：**
- 自动检测系统中安装的 NVIDIA 驱动版本
- 查找 NVIDIA P2P 头文件（`nv-p2p.h`）的路径
- 如果找不到 NVIDIA 驱动，显示警告信息

### 2.3 架构检测

```makefile
ARCH ?= $(shell uname -m)

ifeq ($(ARCH),aarch64)
    ccflags-y += -DAARCH64_PLATFORM
endif

ifeq ($(CONFIG_AARCH64),1)
    ccflags-y += -DAARCH64_PLATFORM
endif
```

**说明：**
- 支持 x86_64 和 ARM64 (aarch64) 架构
- 为 ARM64 平台定义特定的编译标志

---

## 3. 编译标志

### 3.1 基本编译标志

```makefile
ccflags-y += -Wall                  # 启用所有警告
ccflags-y += -I$(NVIDIA_SRC_DIR)    # 包含 NVIDIA 驱动头文件路径
ccflags-y += -I/usr/lib/gcc/x86_64-linux-gnu/7/include/
```

### 3.2 模块编译标志

```makefile
GDS_VERSION ?= $(shell cat GDS_VERSION)
NVFS_MODULE_FLAGS = -DCONFIG_NVFS_STATS=y -DGDS_VERSION=$(GDS_VERSION)

# 启用内核 RDMA 支持（默认启用）
ifneq ($(CONFIG_DISABLE_NVFS_KERN_RDMA_SUPPORT),1)
    NVFS_MODULE_FLAGS += -DNVFS_ENABLE_KERN_RDMA_SUPPORT
    nvidia-fs-y += nvfs-rdma.o
endif

# 启用批量 I/O 支持
ifeq ($(CONFIG_NVFS_BATCH_SUPPORT),y)
    NVFS_MODULE_FLAGS += -DNVFS_BATCH_SUPPORT=y
    nvidia-fs-y += nvfs-batch.o
endif
```

**可选功能：**

| 功能 | 编译标志 | 说明 |
|------|----------|------|
| 统计 | `CONFIG_NVFS_STATS` | 启用性能统计 |
| RDMA | `CONFIG_DISABLE_NVFS_KERN_RDMA_SUPPORT` | 禁用 RDMA 支持 |
| 批量 I/O | `CONFIG_NVFS_BATCH_SUPPORT` | 启用批量 I/O |
| 故障注入 | `CONFIG_FAULT_INJECTION` | 启用故障注入 |
| 代码覆盖率 | `CONFIG_CODE_COVERAGE` | 启用 GCOV 覆盖率 |

---

## 4. 源文件列表

```makefile
nvidia-fs-y = nvfs-core.o nvfs-dma.o nvfs-mmap.o nvfs-pci.o nvfs-proc.o nvfs-mod.o nvfs-kernel-interface.o
nvidia-fs-$(CONFIG_NVFS_STATS) += nvfs-stat.o
nvidia-fs-$(CONFIG_FAULT_INJECTION) += nvfs-fault.o
```

**源文件说明：**

| 对象文件 | 源文件 | 功能 |
|----------|--------|------|
| `nvfs-core.o` | `nvfs-core.c` | 核心 IOCTL 处理 |
| `nvfs-dma.o` | `nvfs-dma.c` | DMA 操作接口 |
| `nvfs-mmap.o` | `nvfs-mmap.c` | 内存映射管理 |
| `nvfs-pci.o` | `nvfs-pci.c` | PCI 拓扑管理 |
| `nvfs-proc.o` | `nvfs-proc.c` | Procfs 接口 |
| `nvfs-mod.o` | `nvfs-mod.c` | 模块管理 |
| `nvfs-kernel-interface.o` | `nvfs-kernel-interface.c` | 内核接口适配 |
| `nvfs-stat.o` | `nvfs-stat.c` | 性能统计（可选） |
| `nvfs-fault.o` | `nvfs-fault.c` | 故障注入（可选） |
| `nvfs-rdma.o` | `nvfs-rdma.c` | RDMA 支持（可选） |
| `nvfs-batch.o` | `nvfs-batch.c` | 批量 I/O（可选） |

---

## 5. 构建目标

### 5.1 默认目标

```makefile
all: module
```

### 5.2 模块编译

```makefile
module: nv_symbols nv_configure
	@ KCPPFLAGS="$(NVFS_MODULE_FLAGS) -DNVFS_BATCH_SUPPORT=y" \
	  CONFIG_NVFS_BATCH_SUPPORT=y CONFIG_NVFS_STATS=y \
	  $(MAKE) -j4 -C $(KDIR) $(MAKE_PARAMS) M=$$PWD modules
```

**编译流程：**
1. `nv_symbols`：生成 NVIDIA 符号版本文件
2. `nv_configure`：运行配置脚本检测内核功能
3. 调用内核 kbuild 系统编译模块

### 5.3 配置脚本

```makefile
nv_configure:
	@ ./configure $(KVER)
```

**说明：**
- `configure` 脚本检测内核版本和功能
- 生成 `config-host.h` 头文件

### 5.4 符号版本文件

```makefile
nv_symbols:
	@ echo "Picking NVIDIA driver sources from NVIDIA_SRC_DIR=$(NVIDIA_SRC_DIR)"
	@ chmod +x ./create_nv.symvers.sh
	@ ./create_nv.symvers.sh
	@ cat nv.symvers >> Module.symvers
```

**说明：**
- 生成 NVIDIA 驱动的符号版本信息
- 确保 GDS 模块与 NVIDIA 驱动版本兼容

---

## 6. 安装目标

### 6.1 安装模块

```makefile
install:
	[ -d $(DESTDIR)/$(MODULE_DESTDIR) ] || mkdir -p $(DESTDIR)/$(MODULE_DESTDIR)
	cp $$PWD/nvidia-fs.ko $(DESTDIR)/$(MODULE_DESTDIR)
	if [ ! -n "$(DESTDIR)" ]; then $(DEPMOD) -a $(KVER); fi
```

**说明：**
- 将编译好的模块复制到内核模块目录
- 运行 `depmod` 更新模块依赖

### 6.2 卸载模块

```makefile
uninstall:
	/bin/rm -f $(DESTDIR)/$(MODULE_DESTDIR)/nvidia_fs.ko
	if [ ! -n "$(DESTDIR)" ]; then $(DEPMOD) -r -ae $(KVER); fi
```

---

## 7. 清理目标

```makefile
clean:
	rm -rf *.o *.ko* *.mod.* .*.cmd nv.symvers Module.symvers modules.order .tmp_versions/ *~ core .depend TAGS .cache.mk *.o.ur-safe
	rm -f config-host.h
	rm -f nvidia-fs.mod
```

---

## 8. 使用示例

### 8.1 编译模块

```bash
# 基本编译
make

# 指定内核版本
make KVER=5.15.0-generic

# 启用所有可选功能
make CONFIG_NVFS_BATCH_SUPPORT=y CONFIG_NVFS_STATS=y

# 禁用 RDMA 支持
make CONFIG_DISABLE_NVFS_KERN_RDMA_SUPPORT=1
```

### 8.2 安装模块

```bash
# 安装到系统
sudo make install

# 安装到指定目录
make install DESTDIR=/tmp/nvidia-fs

# 加载模块
sudo modprobe nvidia-fs
```

### 8.3 清理构建

```bash
make clean
```

---

## 9. 相关文件

- `configure`：内核功能检测脚本
- `create_nv.symvers.sh`：NVIDIA 符号版本生成脚本
- `dkms.conf`：DKMS（动态内核模块支持）配置
- `GDS_VERSION`：版本号文件
