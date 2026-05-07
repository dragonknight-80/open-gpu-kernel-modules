# Open GPU Kernel Modules 项目分析框架（细化到 GPU IP Core）

> 目标：给出一个可执行、可扩展的分析方法，用于系统化理解 `open-gpu-kernel-modules` 代码库，并把软件模块映射到 GPU 硬件 IP Core。

## 1. 总览：项目分层与分析入口

按仓库结构可先分三层：

1. **Linux 内核接口层（kernel-open/）**
   - 负责与 Linux 内核 API 对接（PCI、DRM、MM、中断、同步、字符设备等）。
2. **OS-agnostic 层（src/）**
   - 与具体 Linux 版本弱耦合，承载核心 GPU 管理逻辑（RM、显示模式、UVM 相关逻辑）。
3. **固件/生态对接层（nouveau/ + GSP firmware 相关）**
   - 固件提取、布局、与 Nouveau 协同。

建议把每个分析对象都放入统一模板：

- 模块名
- 所属目录与编译产物
- 核心职责
- 关键数据结构
- 关键入口/回调
- 依赖的内核子系统
- 对应 GPU IP Core
- 初始化/异常/恢复路径
- 调试手段（日志、tracepoint、ioctl、参数）

## 2. 先建立“模块→IP Core”映射表（核心框架）

> 注意：开源内核模块不等于公开全部硬件微架构细节。这里是“驱动职责视角”的 IP 映射。

### 2.1 GPC / TPC / SM（图形与计算执行核心）- 深入版

#### 2.1.1 这个 IP 在驱动中的“可见边界”

- GPC/TPC/SM 本身是执行阵列，内核模块里**更多体现为“通道、上下文、虚拟地址空间、故障恢复”**，而不是直接暴露“某个 SM 寄存器逐条编程”。
- 在本仓库里，你看到的是：
  - 设备文件/ioctl 入口 -> 资源对象管理；
  - 命令提交通道和同步对象；
  - GPU fault / MMU fault 的回传与恢复；
  - 与 UVM 的地址空间协同。

#### 2.1.2 建议先看的文件（函数级分析前的最小集合）

- 内核接口总入口与设备管理：
  - `kernel-open/nvidia/nv.c`
  - `kernel-open/nvidia/nv-linux.c`
  - `kernel-open/nvidia/nv-frontend.c`
- ioctl 与对象/控制路径：
  - `kernel-open/nvidia/nv-ioctl.c`
  - `kernel-open/nvidia/nv-mmap.c`
- OS-agnostic RM 主体（执行核心的资源抽象主要在这里）：
  - `src/nvidia/src/kernel/rmapi/`
  - `src/nvidia/src/kernel/gpu/`
  - `src/nvidia/src/kernel/mem_mgr/`
- UVM 协同：
  - `kernel-open/nvidia-uvm/uvm.c`
  - `kernel-open/nvidia-uvm/uvm_va_space.c`
  - `kernel-open/nvidia-uvm/uvm_gpu.c`

#### 2.1.3 控制流（GPC/TPC/SM 视角）

可按下面链路追踪：

1. 用户态 CUDA/OpenGL/Vulkan 触发提交。
2. 通过字符设备/ioctl 进入 `kernel-open/nvidia/`。
3. RM 将提交映射到“通道/上下文/地址空间/同步对象”。
4. 命令由前端与调度路径下发到 GPU 执行阵列（GPC/TPC/SM）。
5. 完成/异常通过中断、事件、fence 回传。
6. 若 fault：走 UVM / RM fault handler，决定重试、迁移、回收或 reset。

#### 2.1.4 数据流（GPC/TPC/SM 视角）

1. 用户态 buffer/command buffer 建立。
2. 内核 pin + map（CPU VA -> GPU VA）。
3. GPC/TPC/SM 通过统一地址空间取指/取数。
4. 结果写回显存或系统内存。
5. 同步对象（fence/semaphore/event）通知用户态。

#### 2.1.5 重点故障模型（建议逐条验证）

- **地址翻译相关**：GPU page fault、invalid PTE、权限位冲突。
- **上下文相关**：上下文切换超时、通道卡死、抢占失败。
- **同步相关**：fence 不完成、semaphore 失配。
- **恢复相关**：engine reset 之后对象一致性丢失。

#### 2.1.6 GPC/TPC/SM 深挖执行清单（可直接开工）

1. 建一张“ioctl -> RM 对象 -> 提交通道 -> 中断回调”时序图。
2. 建一张“UVM fault -> 迁移/映射更新 -> 重放提交”时序图。
3. 按 GPU 架构代际（Turing/Ampere/Hopper/Blackwell）比对同类路径差异。
4. 整理 10 个常见错误日志模式及定位手册。


#### 2.1.7 函数级时序版（第一版，可直接对照源码）

> 说明：由于 GSP/RM 分层，GPC/TPC/SM 的“硬件执行”不会在开源代码中完整展开；下面时序聚焦你可直接跟踪到的内核函数链。

**A) 设备与 VA 空间准备（为 GPC/TPC/SM 执行做前置）**

1. `uvm_open()` 打开 `/dev/nvidia-uvm`，建立 UVM file 上下文。  
2. `uvm_api_mm_initialize()` 绑定 UVM FD 与 MM 关联关系。  
3. `uvm_gpu_init()` / `init_gpu()` 完成 GPU 级初始化。  
4. `alloc_and_init_address_space()` 与 `configure_address_space()` 创建并配置 GPU address space。  
5. `uvm_gpu_init_va_space()` 将 VA space 与 GPU 关联，后续提交与 fault 才有上下文。

**B) CPU 访问触发的 managed memory fault（与 SM 执行强相关）**

1. CPU 访问 managed 区域触发 VMA fault。  
2. `uvm_vm_fault()` -> `uvm_va_space_cpu_fault_managed()` 进入 UVM fault 处理。  
3. UVM 在 `uvm_va_space`/`uvm_va_range`/`uvm_va_block` 路径中更新映射、迁移页、建立一致性。  
4. fault 处理返回后，CPU/GPU 在同一 VA 视图继续协同执行。

**C) GPU fault 与中断底半部（直接影响 GPC/TPC/SM 连续执行）**

1. GPU 运行中出现 replayable/non-replayable fault。  
2. ISR 统计与处理路径由 `uvm_parent_gpu_init_isr()` 初始化的结构接管。  
3. fault service 通过 bottom-half 执行（可在 `uvm_gpu.c` 看到 replayable/non-replayable 统计与锁）。  
4. 通过 tracker（如 replay tracker / clear_faulted tracker）等待与收敛，决定 replay、clear 或恢复。  
5. 必要时走 `uvm_parent_gpu_disable_isr()` / `uvm_parent_gpu_flush_bottom_halves()` 进入收敛或回收流程。

**D) 进程退出/映射回收（防止遗留状态污染下一次提交）**

1. `uvm_release()` 分派到 `uvm_release_va_space()` 或 `uvm_release_mm()`。  
2. `uvm_vm_close_managed()` / `uvm_destroy_vma_managed()` 销毁 managed ranges。  
3. `uvm_va_space_destroy()` 清理 VA 空间及关联资源。

**E) nvidia.ko -> RMAPI 主链（补充 GPC/TPC/SM 提交入口）**

1. `/dev/nvidia*` 请求经 `kernel-open/nvidia/nv.c` 与 `nv-mmap.c` 进入内核接口层。  
2. 资源对象/控制请求下沉到 `src/nvidia/src/kernel/rmapi/entry_points.c`、`rmapi.c`、`alloc_free.c`。  
3. GPU 对象与内存对象在 `src/nvidia/src/kernel/gpu/`、`src/nvidia/src/kernel/mem_mgr/` 协同建模。  
4. 提交流程最终由 RM + firmware/GSP 驱动硬件执行阵列（GPC/TPC/SM）。

**建议打点（trace/log）顺序**

- 第 1 层：`uvm_open` / `uvm_api_mm_initialize` / `uvm_gpu_init` 是否成功。  
- 第 2 层：`uvm_vm_fault` 调用频率、返回码、关联 `uvm_va_space_cpu_fault_managed` 行为。  
- 第 3 层：`uvm_gpu.c` 中 replayable/non-replayable fault 统计项与 bottom-half 计数。  
- 第 4 层：回收路径是否完整走到 `uvm_release` -> `uvm_va_space_destroy`。

### 2.2 Copy Engine (CE)

- 典型职责：DMA copy、异步搬运、P2P 数据路径。
- 代码关注点：
  - `kernel-open/nvidia-peermem/`（RDMA / peer memory 对接）
  - `nvidia.ko` 中 DMA/映射相关路径。
- 重点分析问题：
  - pin/unpin 生命周期；
  - IOMMU/ATS 场景下地址转换与一致性。

### 2.3 Memory Subsystem（FB、BAR、MMU、页表）

- 典型职责：显存管理、CPU-GPU 映射、地址空间隔离。
- 代码关注点：
  - `kernel-open/nvidia/` 内存映射与 ioctl 入口；
  - `kernel-open/nvidia-uvm/` 的 unified memory fault/migrate。
- 重点分析问题：
  - 哪些路径由 CPU page fault 触发，哪些由 GPU fault 触发；
  - NUMA、THP、HMM 兼容性。

### 2.4 Display Pipeline（Display / Head / Window / Cursor / LUT / Flip）

- 典型职责：模式设置、扫描输出、vblank、flip、色彩管理、VRR。
- 代码关注点：
  - `src/nvidia-modeset/`（OS-agnostic modeset 核心）
  - `kernel-open/nvidia-modeset/`（Linux glue）
  - `kernel-open/nvidia-drm/`（DRM KMS 集成）
- 重点分析问题：
  - atomic commit 到硬件编程链路；
  - vblank / flip fence 的时序；
  - HDR、LUT、颜色空间设置链路。

### 2.5 Display I/O PHY（HDMI / DP / AUX / link training）

- 典型职责：链路训练、带宽协商、热插拔、EDID/DP AUX。
- 代码关注点：
  - `src/nvidia-modeset/src/dp/` 和 `nvkms-hdmi.c`
  - `kernel-open/nvidia-drm/nvidia-drm-connector*.c`
- 重点分析问题：
  - HPD 中断处理与去抖；
  - DP link retrain 失败恢复路径。

### 2.6 Video Codec IP（NVDEC / NVENC）

- 典型职责：视频编解码硬件会话管理。
- 代码关注点：
  - 多数策略在用户态组件 + 固件，本仓库以内核资源与通道安全隔离为主。
- 重点分析问题：
  - 内核可见的会话资源对象有哪些？
  - 错误传播到用户态的路径（ioctl errno / event）。

### 2.7 PMU / Power / Thermals / Clocks

- 典型职责：功耗状态切换、时钟域管理、温度/风扇策略（部分封闭）。
- 代码关注点：
  - `src/nvidia-modeset/src/nvkms-pow.c` 等功耗相关文件；
  - `nvidia.ko` 中设备电源管理回调。
- 重点分析问题：
  - suspend/resume 的顺序依赖；
  - runtime PM 与显示活动之间的互锁。

### 2.8 GSP（GPU System Processor）

- 典型职责：把一部分传统内核 RM 职责下沉到固件执行。
- 代码关注点：
  - README 中已明确模块需匹配对应 GSP firmware 版本。
  - `nouveau/` 工具展示了固件镜像抽取与布局信息。
- 重点分析问题：
  - host<->GSP 消息边界在哪里；
  - 哪些异常由 GSP 检测并回传。

### 2.9 Security / Isolation（Falcon、安全启动链相关）

- 典型职责：微控制器加载、安全策略、访问隔离。
- 代码关注点：
  - 本仓库公开层多为接口和控制路径，具体安全实现可能在固件/闭源组件。
- 重点分析问题：
  - 哪些敏感操作在内核模块中做权限门控；
  - ioctl capability 检查和对象句柄校验。

## 3. 模块到 IP Core 的初始映射表（含文件路径）

> 这是第一版“可执行地图”：用于快速确定你该去哪一层、哪个目录追踪某个 IP 行为。

| 模块 / 产物 | 主要目录 | 对应 IP Core | 你先看的文件（最小集合） | 备注 |
|---|---|---|---|---|
| `nvidia.ko` 内核接口层 | `kernel-open/nvidia/` | GPC/TPC/SM、Memory/MMU、中断/恢复、安全门控 | `nv.c`, `nv-linux.c`, `nv-frontend.c`, `nv-ioctl.c`, `nv-mmap.c` | 计算/图形核心入口最关键 |
| `nvidia.ko` OS-agnostic 层 | `src/nvidia/` | GPC/TPC/SM 抽象、RM 对象模型、内存管理 | `src/kernel/rmapi/`, `src/kernel/gpu/`, `src/kernel/mem_mgr/` | 核心逻辑浓度最高 |
| `nvidia-uvm.ko` | `kernel-open/nvidia-uvm/` | Unified Memory、GPU fault/migration、VA 协同 | `uvm.c`, `uvm_gpu.c`, `uvm_va_space.c`, `uvm_migrate.c` | AI/HPC 重点路径 |
| `nvidia-modeset.ko` 内核接口层 | `kernel-open/nvidia-modeset/` | Display pipeline（head/window/flip/vblank） | `nvidia-modeset-linux.c`, `nvkms-ioctl.h` | Linux 侧 glue |
| `nvidia-modeset.ko` OS-agnostic 层 | `src/nvidia-modeset/` | Display 核心状态机、DP/HDMI、VRR、颜色管理 | `src/nvkms-modeset.c`, `src/nvkms-flip.c`, `src/nvkms-dpy.c`, `src/dp/` | 显示分析主战场 |
| `nvidia-drm.ko` | `kernel-open/nvidia-drm/` | DRM/KMS 接口、原子提交、connector/encoder/crtc | `nvidia-drm-drv.c`, `nvidia-drm-modeset.c`, `nvidia-drm-connector.c`, `nvidia-drm-crtc.c` | 与 Linux 图形栈对接核心 |
| `nvidia-peermem.ko` | `kernel-open/nvidia-peermem/` | CE 数据搬运相关、P2P/RDMA 映射 | `nvidia-peermem.c`, `nv-p2p.h` | 高性能网络与 GPU 互联关键 |
| 固件抽取工具 | `nouveau/` | GSP firmware 边界、固件布局 | `extract-firmware-nouveau.py`, `extract-firmware-nouveau.txt` | 用于理解 host/firmware 边界 |

## 4. 以“数据流 + 控制流”双轴分析每个 IP Core

对每个 IP core 都画两张图：

1. **数据流图**：用户态 buffer → 内核映射 → IOMMU/GPU VA → IP Core 访问。
2. **控制流图**：ioctl/atomic commit → RM/KMS 调度 → 中断/回调 → fence/event。

最小闭环（建议统一模板）：

- init path
- steady-state path
- fault path
- reset/recover path
- teardown path

## 5. 建议的“源码阅读顺序”（高效）

1. `README.md`：明确模块边界与构建产物。
2. `Makefile`、`kernel-open/*/*.Kbuild`：看目标模块与对象文件关系。
3. `kernel-open/nvidia-drm/`：从标准 DRM/KMS 入口理解显示链路。
4. `kernel-open/nvidia-modeset/` + `src/nvidia-modeset/`：深入显示 IP（Head/Window/DP/HDMI/VRR/Flip）。
5. `kernel-open/nvidia-uvm/`：统一内存与 fault/migration。
6. `kernel-open/nvidia/` + `src/nvidia/`：RM 主体，覆盖通道/对象/内存/中断。
7. `kernel-open/nvidia-peermem/`：CE/P2P/RDMA 专项。

## 6. 面向落地的“IP Core 分析清单”（你可以直接照抄用）

每个 IP core 建一个 markdown（例如 `analysis/ip-display.md`），按以下小节：

1. IP 功能边界
2. 关键源码文件与入口函数
3. 与其他 IP 的依赖图
4. 上电初始化时序
5. 正常命令执行时序
6. 中断与错误码
7. reset/recovery 与超时机制
8. 性能计数器/可观测性
9. 常见 bug 模式
10. 复现实验与日志抓取命令

## 7. 建议优先深挖的 IP（按收益排序）

1. **Display（含 DP/HDMI）**：代码公开度高，最容易建立端到端认知。
2. **Memory + UVM**：对 AI/HPC 场景价值最高。
3. **CE + peermem**：和 RDMA/P2P 直接相关。
4. **GSP 边界**：决定“哪些事情在内核，哪些在固件”。

## 8. 你可以如何使用这个框架（两周计划）

- **第 1-2 天**：完成模块总图（nvidia / modeset / drm / uvm / peermem）。
- **第 3-5 天**：完成 Display IP（含 DP link training 与 flip/vblank）闭环。
- **第 6-8 天**：完成 UVM + Memory fault/migrate 闭环。
- **第 9-10 天**：完成 CE/peermem 闭环。
- **第 11-14 天**：补齐异常恢复、性能与调试手册。
