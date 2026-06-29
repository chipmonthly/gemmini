<p align="center">
  <img width="1000" src="./img/full-logo.svg">
</p>

Gemmini
====================================

Gemmini 项目正在开发一个全系统、全栈的深度神经网络 (DNN) 硬件探索与评估平台。
Gemmini 使架构师能够深入了解系统和软件栈的不同组件（不仅仅是加速器本身）如何相互作用以影响整体 DNN 性能。

Gemmini 是 [Chipyard](https://github.com/ucb-bar/chipyard) 生态系统的一部分，是使用 [Chisel](https://www.chisel-lang.org/) 硬件描述语言开发的。

本文档旨在为想要尝试 Gemmini 的初学者提供信息，并为那些想要开始修改 Gemmini 源代码的更高级的开发者提供深入的信息。

![Gemmini的高层架构](./img/gemmini-system.png)

快速入门
==========

我们在此提供了一个快速指南，介绍如何安装 Gemmini 的依赖项（Chipyard 和 Spike）、构建 Gemmini 硬件和软件，然后在我们的硬件模拟器上运行该软件。

依赖项
---------

在开始之前，请安装 [Chipyard 依赖项](https://chipyard.readthedocs.io/en/latest/Chipyard-Basics/Initial-Repo-Setup.html#default-requirements-installation)。

安装 Chipyard 和 Spike
-----------------------------

运行以下步骤以安装 Chipyard 和 Spike（确保签出正确的 Chipyard 和 Spike 提交，如下所示）：

```shell
git clone https://github.com/ucb-bar/chipyard.git
cd chipyard
./build-setup.sh

source env.sh

cd generators/gemmini
make -C software/libgemmini install
```


构建 Gemmini 软件
-------------------------

运行以下步骤以编译 Gemmini 程序，包括像 ResNet50 这样的大型 DNN 模型，以及小型矩阵乘法测试。

```shell
cd chipyard/generators/gemmini/software/gemmini-rocc-tests
./build.sh
```

之后，您将在 `build/` 中找到适用于“裸机 (baremetal)”环境、“Linux”环境和“代理内核 (proxy-kernel)”环境的 RISC-V 二进制文件。

Linux 二进制文件旨在运行 Linux 的 SoC 上执行。
这些二进制文件是动态链接的，并支持所有系统调用。
通常，我们的用户在 [FireSim](https://fires.im/) 模拟器上运行它们。

裸机 (Baremetal) 二进制文件旨在没有可用操作系统的环境中运行。
它们缺少对大多数系统调用的支持，也不支持虚拟内存。
我们的用户通常在 Verilator 或 VCS 等周期精确模拟器上运行它们。

“代理内核 (Proxy-kernel)”二进制文件旨在名为 [“RISC-V Proxy Kernel”](https://github.com/riscv-software-src/riscv-pk) 的精简版 Linux 上运行。
这些二进制文件支持虚拟内存，通常在 Verilator 等周期精确模拟器上运行。

**警告：** 代理内核二进制文件的堆空间有限，因此某些在裸机或 Linux 环境中正常运行的 Gemmini 程序可能会在代理内核上运行失败。

构建 Gemmini 硬件和周期精确模拟器
-----------------------------------------------

运行以下指令，使用 Verilator 构建周期精确的 Gemmini 模拟器。

```shell
cd chipyard/sims/verilator
make CONFIG=GemminiRocketConfig

# 或者，如果您想要一个可以生成波形的模拟器，请运行以下命令：
make debug CONFIG=GemminiRocketConfig
```

运行此命令后，除了周期精确的模拟器外，您还可以在 `generated-src/` 中找到 SoC 的 Verilog 描述。

使用 Gemmini 功能模拟器
---------------------------

Spike 的运行速度通常比 Verilator 或 VCS 等周期精确模拟器快得多。
然而，Spike 只能验证功能正确性；它无法提供精确的性能指标或剖析 (profiling) 信息。

运行模拟器
---------------

运行以下指令，使用我们上面构建的模拟器来运行之前构建的 Gemmini RISC-V 二进制文件：

```shell
cd chipyard/sims/verilator

# 在功能模拟器中运行大型 DNN 工作负载
spike --extension=gemmini pk ../../generators/gemmini/software/gemmini-rocc-tests/build/imagenet/resnet50-pk

# 在功能模拟器中运行小型 DNN 工作负载
spike --extension=gemmini ../../generators/gemmini/software/gemmini-rocc-tests/build/imagenet/resnet50-baremetal

# 在周期精确模拟器上以裸机模式运行较小的工作负载
make CONFIG=GemminiRocketConfig run-binary BINARY=../../generators/gemmini/software/gemmini-rocc-tests/build/bareMetalC/template-baremetal
```

后续步骤
--------

查看我们的 [MLSys 2022 教程](https://sites.google.com/berkeley.edu/gemmini-tutorial-mlsys-2022)（或我们较早但有些过时的 [IISWC 2021 教程](https://sites.google.com/berkeley.edu/gemminitutorialiiswc2021/)）来学习如何：
* 使用 Gemmini 构建不同类型的多样化加速器。
* 向 Gemmini 添加自定义数据类型。
* 编写您自己的 Gemmini 程序。
* 使用 Gemmini 的性能计数器剖析您的工作负载。

此外，可以考虑了解 [FireSim](fires.im)，这是一个用于 FPGA 加速的周期精确模拟平台。
我们使用 FireSim 运行在 Verilator/VCS 上运行时间过长的端到端 DNN 工作负载。
FireSim 还允许用户检查其 Gemmini 硬件/软件在 Linux 环境中运行时是否能够正常工作。

或者，继续阅读本文档的其余部分，了解 Gemmini 的架构、指令集架构 (ISA) 和配置参数。

硬件架构
================

Gemmini 被实现为一个带有非标准 RISC-V 自定义指令的 RoCC 加速器。
Gemmini 单元使用 Rocket 或 BOOM _tile_ 的 RoCC 端口，并且默认情况下通过系统总线（即直接连接到 L2 缓存）连接到内存系统。

加速器的核心是一个执行矩阵乘法的脉动阵列 (systolic array)。
默认情况下，矩阵乘法同时支持 _输出固定 (output-stationary)_ 和 _权重固定 (weight-stationary)_ 数据流，程序员可以在运行时选择这两种数据流。
不过，数据流也可以在阐释 (elaboration) 阶段固化。

脉动阵列的输入和输出存储在显式管理的暂存器 (scratchpad) 中，暂存器由分块 (banked) SRAM 组成。
DMA 引擎协助在主存（对主机 CPU 可见）和暂存器之间传输数据。

由于权重固定数据流需要在脉动阵列外部使用累加器，因此我们添加了最后一个配备了加法器单元的 SRAM 块 (bank)，它在概念上可以被视为暂存器内存空间的延伸。脉动阵列可以将结果存储到累加器中的任何地址，也可以从累加器中的任何地址读取新的输入。DMA 引擎还可以直接在累加器和主存之间传输数据，这在加载偏置 (bias) 时通常是必需的。

Gemmini 还包括外围电路，以可选地应用激活函数（如 ReLU 或 ReLU6）、按 2 的幂次向下缩放结果以支持量化工作负载，或者在将矩阵输入脉动阵列之前对其进行转置以支持输出固定数据流。

生成器参数
--------------------------

主要关注的参数包括：

* 脉动阵列维度 (``tileRows``, ``tileColumns``, ``meshRows``, ``meshColumns``)：脉动阵列由两级层次结构组成，其中每个 tile（图块）完全是组合逻辑，而 tile 构成的 mesh（网格）在每个 tile 之间都有流水线寄存器。

![Gemmini 的脉动两级层次结构](./img/gemmini-systolic-array.png)

* 数据流参数 (``dataflow``)：决定 Gemmini 中的脉动阵列是输出固定、权重固定，还是同时支持这两种数据流以便程序员在运行时进行选择。

* 暂存器和累加器内存参数 (``sp_banks``, ``sp_capacity``, ``acc_capacity``)：确定 Gemmini 暂存器内存的属性：暂存器或累加器的总容量（以 KiB 为单位），以及暂存器被划分成的 SRAM 块 (bank) 数量。

* 类型参数 (``inputType``, ``outputType``, ``accType``)：
确定流经 Gemmini 加速器不同部分的数据类型。
例如，``inputType`` 可以是 8 位定点数，而决定矩阵乘法中部分和累加类型的 ``accType`` 可以是 32 位整数。
``outputType`` 仅决定两个处理单元 (PE) 之间传递的数据类型；例如， 8 位乘法可能会产生一个 16 位的结果，该结果必须在脉动阵列中的 PE 之间共享。
    - 可选数据类型的示例如下：
        - `SInt(8.W)` 表示带符号的 8 位整数
        - `UInt(32.W)` 表示无符号的 32 位整数
        - `Float(8, 24)` 表示单精度 IEEE 浮点数
    - 如果您的数据类型是浮点数，那么您可能还需要更改 ``pe_latency`` 参数，该参数指定在 PE 内部添加多少个移位寄存器。
如果您的数据类型无法在单个周期内完成乘法累加操作，则这可能是必需的。

* 访存-执行队列参数 (``ld_queue_length``, ``st_queue_length``, ``ex_queue_length``, ``rob_entries``)：为了实现访存与执行解耦，Gemmini 加速器具有加载指令队列、存储指令队列和执行指令队列。这些队列的相对大小决定了访存-执行解耦的程度。Gemmini 还实现了一个重排序缓冲区 (ROB) - ROB 中的条目数决定了可能存在的依赖关系管理限制。

* DMA 参数 (``dma_maxbytes``, ``dma_buswidth``, ``mem_pipeline``)：Gemmini 实现了一个 DMA 用于将数据从主存移动到 Gemmini 暂存器，以及将数据从 Gemmini 累加器移动到主存。这些 DMA 事务的大小由 DMA 参数决定。这些 DMA 参数与 Rocket Chip SoC 系统参数紧密耦合：特别是 ``dma_buswidth`` 与 ``SystemBusKey`` 的 ``beatBytes`` 参数相关联，而 ``dma_maxbytes`` 与 Rocket Chip 参数的 ``cacheblockbytes`` 相关联。

还有一些可选功能，可以在阐释 (elaboration) 阶段在 Gemmini 中启用或不启用。
例如：

* “移入 (move-in)”操作期间的缩放 (``mvin_scale_args``, ``mvin_scale_acc_args``)：
当数据从 DRAM 或主存移入 Gemmini 的本地暂存器内存时，它可以选择乘以一个缩放因子。
这些参数指定缩放因子的数据类型，以及缩放是如何实际执行的。
如果这些参数设置为 ``None``，则该可选功能将在阐释阶段被禁用。
如果暂存器输入和累加器输入都要以相同的方式进行缩放，则可以将 ``mvin_scale_shared`` 参数设置为 ``true``，以便共享乘法器和功能单元。

主要组件
----------------

本小节面向那些希望开始开发/修改 Gemmini RTL 的开发者。
在这里，我们简要介绍 Gemmini 的主要硬件组件以及它们是如何组合在一起的。
如果您对修改 Gemmini 的硬件没有兴趣（除了更改配置参数之外），请随时跳过本节。

### 解耦访存/执行 (Decoupled Access/Execute)

Gemmini 采用访存/执行解耦架构，这意味着“内存访问”和“执行”指令在硬件的不同区域同时进行。
我们将硬件大致分为三个“控制器 (controllers)”：一个用于“执行”指令，另一个用于“加载”指令，第三个用于“存储”指令。
这些控制器都消费来自程序员的直接 ISA 命令，解码这些命令并执行它们，同时共享对暂存器和累加器 SRAM 的访问。

* [ExecuteController](file:///Users/yi/project/gemmini/src/main/scala/gemmini/ExecuteController.scala)：此模块负责执行“执行”类型的 ISA 命令，例如矩阵乘法。
它包含一个用于点积的脉动阵列和一个转置器。

* [LoadController](file:///Users/yi/project/gemmini/src/main/scala/gemmini/LoadController.scala)：此模块负责将数据从主存移入 Gemmini 专用暂存器或累加器的所有指令。

* [StoreController](file:///Users/yi/project/gemmini/src/main/scala/gemmini/StoreController.scala)：此模块负责将数据从 Gemmini 专用 SRAM 移入主存的所有指令。
该模块还负责“最大池化 (max-pooling)”指令，因为当将未池化的数据从专用 SRAM 移动到主存时，Gemmini 会执行池化操作。

### 暂存器和累加器

Gemmini 将脉动阵列的输入和输出存储在一组专用 SRAM 中，我们称之为“暂存器 (scratchpad)”和“累加器 (accumulator)”。
通常，输入存储在暂存器中，而部分和与最终结果存储在累加器中。

暂存器和累加器都在 [Scratchpad.scala](file:///Users/yi/project/gemmini/src/main/scala/gemmini/Scratchpad.scala) 中实例化。
暂存器分块由 [Scratchpad.scala](file:///Users/yi/project/gemmini/src/main/scala/gemmini/Scratchpad.scala) 中的 `ScratchpadBank` 模块实现，累加器分块由 [AccumulatorMem.scala](file:///Users/yi/project/gemmini/src/main/scala/gemmini/AccumulatorMem.scala) 中的 [AccumulatorMem](file:///Users/yi/project/gemmini/src/main/scala/gemmini/AccumulatorMem.scala) 模块实现。

暂存器和累加器 SRAM 的每一行都有 `DIM` 个“元素 (elements)”宽，其中 `DIM` 是沿脉动阵列宽度方向的 PE 数量。
每个“元素”代表 Gemmini 操作的单个标量值。

暂存器中的每个“元素”都是 `inputType` 类型（在默认配置中，这是一个 8 位整数）。
累加器中的每个“元素”都是 `accType` 类型（在默认配置中，这是一个 32 位整数）。

因此，以具有 16x16 脉动阵列的默认配置为例，暂存器分块的行宽为 `16 * bits(inputType) = 128` 位，累加器分块的行宽为 `16 * bits(accType) = 512` 位。

暂存器的输入和输出都必须是 `inputType` 类型。

累加器的输入和输出可以为 `accType` _or_ `inputType` 类型。
如果输入到累加器的值是 `inputType` 类型，它们将被转换为 `accType`。
如果从累加器输出的值是 `inputType` 类型，它们将首先被“缩放 (scaled)”降低到 `inputType` 类型。
具体的“缩放”函数可以根据用户的意愿进行配置，但在默认配置中，缩放函数是乘以一个 `float32` 值，从而将 `int32` 强制转换为 `int8`。

暂存器分块非常简单，主要由一个 SRAM 和一个队列组成。

累加器分块稍微复杂一些：除了底层的 SRAM 外，它们还包括一组加法器以支持就地 (in-place) 累加。
此外，它们还有一组“缩放器 (scalers)”（如上所述）和激活函数单元。
当程序员希望在从累加器中读取数据时将 `accType` 值转换降低为 `inputType` 值时，会应用缩放和激活函数。
这通常用于将一层的部分和输出转化为下一层的低位宽量化输入。

### 脉动阵列和转置器

在 [ExecuteController](file:///Users/yi/project/gemmini/src/main/scala/gemmini/ExecuteController.scala) 内实例化的 [MeshWithDelays](file:///Users/yi/project/gemmini/src/main/scala/gemmini/MeshWithDelays.scala) 包含脉动阵列 ([Mesh](file:///Users/yi/project/gemmini/src/main/scala/gemmini/Mesh.scala))、转置器 ([Transposer](file:///Users/yi/project/gemmini/src/main/scala/gemmini/Transposer.scala)) 以及一组用于对脉动阵列输入进行移位的延迟寄存器。
[MeshWithDelays](file:///Users/yi/project/gemmini/src/main/scala/gemmini/MeshWithDelays.scala) 模块每个周期移入一行三个矩阵（`A`、`B` 和 `D`），并每个周期输出一行结果 `C = A * B + D`。

在权重固定模式下，`B` 值被“预加载”到脉动阵列中，而 `A` 和 `D` 值被穿流通过。
在输出固定模式下，`D` 值被“预加载”到脉动阵列中，而 `A` 和 `B` 值被穿流通过。

`A`、`B` 和 `D` 都是 `inputType` 类型，而 `C` 是 `outputType` 类型。
如果程序员希望将 `C` 写入暂存器，则 `C` 会被转换为 `inputType`。
然而，如果程序员希望将 `C` 写入累加器，则 `C` 会被转换为 `accType`。

请注意，在权重固定模式下，`inputType` 的 `D` 通常没有足够的位宽来精确表示部分和。
因此，在权重固定模式下，`D` 通常只是 0 矩阵，而使用 `accType` 的累加器 SRAM 来累加脉动阵列的部分和输出。

输入（`A`、`B` 和 `D`）必须通过移位寄存器进行延迟，以便来自一个矩阵的每个输入都在完全正确的时间到达正确的 PE，从而与来自另一个矩阵的正确输入进行乘加。
下图显示了一个 2x2 输出固定矩阵乘法（忽略 `D`）的示例，其中在脉动阵列的输入和输出端配有适当的延迟寄存器：

![配有延迟寄存器的脉动阵列](./img/delay-registers.png)

脉动阵列本身（在 [Mesh.scala](file:///Users/yi/project/gemmini/src/main/scala/gemmini/Mesh.scala) 中实现）由 `Tiles` 和 `PEs` 的两级层次结构组成。
`Mesh` 由一组 `Tiles` 组成，它们之间由流水线寄存器分隔。
每个 `Tile` 由组合逻辑的 `PE` 集合组成，其中每个 PE 使用权重固定或输出固定数据流执行单个矩阵乘加操作。

![脉动阵列](./img/gemmini-systolic-array.png)

[MeshWithDelays](file:///Users/yi/project/gemmini/src/main/scala/gemmini/MeshWithDelays.scala) 模块还包括许多计数器和配置寄存器。
[MeshWithDelays](file:///Users/yi/project/gemmini/src/main/scala/gemmini/MeshWithDelays.scala) 假定每个矩阵乘法操作的大小完全是 `DIM x DIM`，其中 `DIM` 是跨越脉动阵列本身宽度的 PE 数量（默认配置中为 16）。
这些计数器计数到 `DIM`，然后从 [MeshWithDelays](file:///Users/yi/project/gemmini/src/main/scala/gemmini/MeshWithDelays.scala) 的输入更新配置寄存器。
这些配置寄存器控制在将 `A` 和 `B` 输入脉动阵列之前是否对它们进行转置。
它们还控制脉动阵列中的预加载值是保留给下一个矩阵乘法，还是被重写并替换。

转置器本身实现为一个非常简单的脉动阵列，它在 `DIM` 个周期内将输入从左向右传输，然后在另外 `DIM` 个周期内将输入从下向上传输。
如下图所示：

![转置器](./img/transposer.png)

请注意，对于输出固定矩阵乘法，即使程序员没有请求转置，也会使用转置器。
这是因为在输出固定模式下，脉动阵列期望来自 `A` 的同一行的输入进入同一个 PE，但是 `A` 单个行中的所有值都存储在同一个暂存器 SRAM 行中。
因此，在从暂存器中读取行之后必须对其进行转置，以便可以将同一行上的元素一个接一个地输入到同一个 PE 中，而不是输入到相邻的 PE 中。

### DMA

Gemmini 包含两个 DMA：一个用于将数据从主存读取到 Gemmini 的专用 SRAM 中，另一个用于将数据从 Gemmini 的专用 SRAM 移动到主存中。
这两个模块均在 [DMA.scala](file:///Users/yi/project/gemmini/src/main/scala/gemmini/DMA.scala) 中实现。

两个 DMA 均在虚拟地址上操作，并共享对 TLB 的访问权限，以将虚拟地址转换为物理主存地址。
如果 TLB 未命中，它会透明地回退到与 Gemmini 的主机 CPU 共享的 PTW。

从 Gemmini 的专用 TLB 获取物理地址后，DMA 会将大型内存请求分解为较小的 [TileLink](https://sifive.cdn.prismic.io/sifive%2Fcab05224-2df1-4af8-adee-8d9cba3378cd_tilelink-spec-1.8.0.pdf) 读写请求。
为了满足 TileLink 协议，每个内存请求必须对齐到从/向主存请求的字节数，并且每个内存请求的大小（以字节为单位）必须是 2 的幂。
DMA 通常会尽量减少 TileLink 请求的数量，即使这需要从主存中读取更大总数据量。
经验表明，过多的 TileLink 请求对性能的限制可能比读取少量额外数据更严重。

将数据从专用 SRAM 写入主存的 DMAWriter 还包含一组 `>` 比较器，用于在内存写入操作期间对数据进行最大池化 (max-pooling)。

### ROB

由于 Gemmini 的解耦访存-执行架构，[LoadController](file:///Users/yi/project/gemmini/src/main/scala/gemmini/LoadController.scala)、[StoreController](file:///Users/yi/project/gemmini/src/main/scala/gemmini/StoreController.scala) 和 [ExecuteController](file:///Users/yi/project/gemmini/src/main/scala/gemmini/ExecuteController.scala) 中的指令可以并发运行，并且相对于其他控制器中的指令而言是乱序执行的。
Gemmini 包含一个重排序缓冲区 (ROB)，旨在检测不同控制器中的指令之间的冲突 (hazards)。
ROB 中的指令只有在与其他控制器中的指令没有依赖关系时，才会下发到各自的控制器中。

请注意，发送给同一个控制器的指令是顺序下发的。
ROB 不会检查同一个控制器内部指令之间的冲突，因为假定每个控制器按程序顺序接收自己的指令，并且有义务在内部处理自己的依赖关系和冲突。

### 矩阵乘法和卷积循环展开器 (Matmul and Conv Loop Unrollers)

Gemmini 的脉动阵列只能在最大为 `DIM x DIM` 个元素的矩阵乘法上运行。
在执行大于此大小的矩阵乘法和卷积时，程序员必须将其矩阵乘法分块为一系列较小的 `DIM x DIM` 矩阵乘法。

然而，由于 CPU 和循环开销，以及软件循环展开和流水线化的难度，高效地对这些操作进行分块对于程序员来说可能很困难。

为了缓解这一困难，Gemmini 的 ISA 包含了高级的 CISC 类型指令，这些指令可自动分块并展开大型矩阵乘法和卷积。
它们在 [LoopMatmul](file:///Users/yi/project/gemmini/src/main/scala/gemmini/LoopMatmul.scala) 和 [LoopConv](file:///Users/yi/project/gemmini/src/main/scala/gemmini/LoopConv.scala) 模块中实现。

这些模块实现为状态机 (FSM)，它们对矩阵乘法/卷积块进行双缓冲以最大化性能，并监控 ROB 中加载/存储/执行指令的比例，以最大化内存访问和点积计算之间的重叠。
例如，如果 ROB 被矩阵乘法指令主导，没有给传入的加载指令留出任何空槽，那么状态机将暂停下发矩阵乘法指令，以便在 Gemmini 的数据通路中为并发加载指令留出更多空间。

软件
==========

Gemmini ISA 在下文的 `ISA` 部分中具体说明。
ISA 包括配置指令、数据移动指令（在主存与 Gemmini 专用内存之间）以及矩阵乘法执行指令。

由于 Gemmini 指令没有通过 GNU binutils 汇编器公开，因此提供了几个 C 语言宏，以构建指令编码来调用这些指令。

Gemmini 生成器包含一个 C 语言库，该库将对自定义 Gemmini 指令的调用封装为常见的 DNN 算子，如矩阵乘法、卷积（带或不带池化）、矩阵加法等。
生成器的 ``software`` 目录包括上述库 and 宏，以及裸机测试，和一些在 Linux 环境中运行测试的 FireMarshal 工作负载。具体而言，该 C 库可以在 [gemmini.h](file:///Users/yi/project/gemmini/software/gemmini-rocc-tests/include/gemmini.h) 文件中找到。

Gemmini 生成器根据生成器参数生成一个 C 头文件。此头文件与 C 库一起编译以微调库的性能。生成的头文件可以在 [gemmini_params.h](file:///Users/yi/project/gemmini/software/gemmini-rocc-tests/include/gemmini_params.h) 下找到。

Gemmini 还可以通过移植微软的 ONNX-Runtime 框架来运行 ONNX 指定的神经网络。该移植版本作为 [onnxruntime-riscv](https://github.com/pranav-prakash/onnxruntime-riscv) 仓库包含在 `software` 目录的子模块中。
要开始使用 ONNX-Runtime，请运行 `git submodule update --init --recursive software/onnxruntime-riscv` 并阅读[此处](https://github.com/pranav-prakash/onnxruntime-riscv/blob/systolic/systolic_runner/docs)的文档。

## 构建并运行 Gemmini 测试

要构建 Gemmini 测试：

```shell
cd software/gemmini-rocc-tests/
./build.sh
```

之后，测试二进制文件将在 `software/gemmini-rocc-tests/build` 中找到。
文件名以 `-baremetal` 结尾的二进制文件旨在在裸机环境中运行，而文件名以 `-linux` 结尾的二进制文件旨在在 Linux 环境中运行。
您可以在周期精确的 RTL 模拟器上运行这些测试，也可以在称为 Spike 的（快得多的）功能 ISA 模拟器上运行。

我们使用 Spike 的一个特殊扩展版本，见[此处](https://github.com/ucb-bar/libgemmini)，它支持 Gemmini 指令。
如果您使用的是 Chipyard，您可以通过在 Chipyard 的根目录下运行 `./scripts/build-toolchains.sh riscv-tools` ，然后在 Gemmini 目录下运行 `make -C software/libgemmini install` 来轻松构建 Spike。
然后，要运行 `mvin_mvout` 测试（该测试只是在将矩阵移出回主存之前将其移入 Gemmini 暂存器），请运行以下命令：

```shell
cd build/bareMetalC
spike --extension=gemmini mvin_mvout-baremetal
```

## 编写您自己的 Gemmini 测试
[template.c](file:///Users/yi/project/gemmini/software/gemmini-rocc-tests/bareMetalC/template.c) 是一个模板 Gemmini 测试，您可以以此为基础编写自己的 Gemmini 测试。要编写您自己的 Gemmini 测试，请运行：

```shell
cd software/gemmini-rocc-tests/
cp bareMetalC/template.c bareMetalC/my_test.c
```

然后，将 `my_test` 添加到 `bareMetalC/Makefile` 顶部的 `tests` 列表中。之后，运行 `./build.sh` 将把 `my_test-baremetal` 安装在 `build/bareMetalC` 中。

## DNN 测试

示例 DNN（如 ResNet50）可以在 `software/gemmini-rocc-tests/imagenet` 和 `software/gemmini-rocc-tests/mlps` 中找到。
这些测试的构建和运行方式与上述其他测试相同，但它们在 VCS 或 Verilator 等软件模拟器中运行通常耗时过长。
我们建议您通过 FPGA 加速模拟平台 [FireSim](https://fires.im/) 运行这些测试，这会将您的运行时间从几天缩短到几分钟。

请注意，DNN 测试依赖于我们的常见 DNN 算子 C 库（位于 [gemmini.h](file:///Users/yi/project/gemmini/software/gemmini-rocc-tests/include/gemmini.h) 中）。
它们极少直接调用 Gemmini ISA 指令，主要调用 C 库中围绕它们的封装函数。

# 内存寻址方案

Gemmini 的专用内存是“按行寻址”的，其中每行有 `DIM` 个元素宽，`DIM` 是脉动阵列宽度方向的 PE 数量（默认配置中为 16）。
这些元素在暂存器中将是 `inputType` 类型，在累加器中将是 `accType` 类型。

每个专用的 Gemmini 内存地址长度均为 32 位。
最高有效 3 位（Most Significant Bits）被保留，并具有特殊含义：
* 如果寻址暂存器，第 31 位（最高位，MSB）为 0；如果寻址累加器，则该位为 1。
* 如果寻址暂存器或从累加器中读取数据，第 30 位被忽略。相反，如果向累加器中写入数据，则如果我们想要覆盖该地址处的数据，第 30 位为 0；如果我们想要在已存在的数据上进行累加，该位为 1。
* 如果寻址暂存器或向累加器中写入数据，第 29 位被忽略。相反，如果从累加器中读取数据，如果我们想要从累加器中读取缩放后的 `inputType` 数据，第 29 位为 0；如果我们想要从累加器中读取 `accType` 数据，该位为 1。
    - 如果对于累加器读取地址而言第 29 位为 1，则我们不对累加器的输出应用激活函数或缩放。

具有 2x2 脉动阵列 of Gemmini 配置的内存寻址方案如下图所示：

![Gemmini 的内存寻址方案](./img/memory-addressing.png)

Gemmini 通过软件可见的虚拟地址访问主存地址（CPU 也可见）。
物理地址转换由 Gemmini 处理，对程序员而言是透明的。

# 指令集架构 (ISA)

本节介绍 Gemmini 的汇编级 ISA，它由自定义 RISC-V 指令组成。

## 数据移动
### `mvin` 将数据从主存移入暂存器
**格式：** `mvin rs1, rs2`
- `rs1` = 要加载到暂存器中的虚拟 DRAM 地址（按字节寻址）
- `rs2[31:0]` = 本地暂存器或累加器地址
- `rs2[47:32]` = 要加载的列数
- `rs2[63:48]` = 要加载的行数。必须小于或等于 `DIM`。
- `funct` = 2

**操作：** Scratchpad[rs2] <= DRAM[Translate[rs1]]
- 将 2D 矩阵从主存加载到 Gemmini 的专用内存中。
- 加载从 rs1/rs2 基地址顺序进行。
- 主存步长 (stride) 必须由 `config_mvin` 命令设置。
- 如果我们加载的列数大于 `DIM`，则将移入多个子矩阵。
这些子矩阵之间的专用内存步长由 `config_mvin` 命令设置。

下图说明了 `mvin` 命令的工作原理：

![Gemmini 的 mvin 命令](./img/mvin.png)

此外，下图说明了移入的列数大于 `DIM` 的特殊情况：

![包含多列的 Gemmini mvin 命令](./img/block-mvin.png)

**注意：**
* Gemmini 中实际上有 **三条** `mvin` 指令：`mvin`、`mvin2` 和 `mvin3`。
`mvin2` and `mvin3` 与 `mvin` 完全相同，只是它们拥有自己独立的配置寄存器组。
当调用 `config_mvin`（如下所述）时，程序员可以选择想要配置哪条 `mvin` 指令。
* 我们有三条 `mvin` 指令的原因是为了让程序员能够重叠 A、B 和 D 矩阵的加载（用于 `A*B+D` 矩阵乘法），其中 A、B 和 D 可能都具有不同的主存步长。

### `mvout` 将数据从暂存器移动到 L2/DRAM
**格式：** `mvout rs1, rs2`
- `rs1` = 要从暂存器写入的虚拟 DRAM 地址（按字节寻址）
- `rs2[31:0]` = 本地暂存器地址
- `rs2[47:32]` = 要存储的列数
- `rs2[63:48]` = 要存储的行数
- `funct` = 3

**操作：** DRAM[Translate[rs1]] <= Scratchpad[rs2]
- 将 2D 矩阵从暂存器存储到主存。
- 存储从 rs1/rs2 基地址顺序进行。步长必须由 `config_mvout` 命令设置。

## 配置
### `config_ex` 配置执行流水线
**格式：** `config_ex rs1 rs2`
- `rs1[1:0]` 必须是 `00`
- `rs1[2]` 决定是输出固定 (0) 还是权重固定 (1)
- `rs1[3]` = 激活函数：relu (1) 或没有激活函数 (0)
- `rs1[8]` = A 是否应该转置？
- `rs1[9]` = B 是否应该转置？
- `rs1[31:16]` = 将 A 的行输入脉动阵列的步长（按暂存器地址计）。
在此上下文中的 "A" 指的是由 A * B = C 表示的矩阵乘法中的左矩阵 A。
如果步长为 1，那么我们将暂存器中从 A 的起始地址开始的连续行作为 A 矩阵输入到脉动阵列中。
如果步长为 2，那么我们改为将每隔一行输入到脉动阵列中。
- `rs1[63:32]` = 从累加器读取时，我们将累加器的 `accType` 输出缩放降为 `inputType` 值的标量值。
    - 在默认配置中，`rs1[63:32]` 是 `float32` 类型
- `rs2[31:0]` = 矩阵乘法的累加结果在离开脉动阵列时右移的位数
    - 此参数仅在输出固定模式下相关，此时部分和必须在脉动阵列内部累加，并在离开脉动阵列并被写入暂存器时进行缩放降低。
- `funct` = 0

**操作：** mode <= rs1(2); shift <= rs2; A_stride <= rs1[31:16]

**注意：**
- 截至目前，除非选择正确的数据流，否则无法执行某些转置选项的组合。
该限制可能会在未来取消。

| 数据流 | 转置 A | 转置 B | 是否允许？ |
| :---: | :---: | :---: | :---: | 
| OS | 否 | 否 | 是 |
| OS | 否 | 是 | 否 |
| OS | 是 | 否 | 是 |
| OS | 是 | 是 | 是 |
| WS | 否 | 否 | 是 |
| WS | 否 | 是 | 是 |
| WS | 是 | 否 | 是 |
| WS | 是 | 是 | 否 |

### `config_mvin` 配置加载流水线
**格式：** `config_mvin rs1 rs2`
- `rs1[1:0]` 必须是 `01`
- `rs1[2]` 如果向累加器的 `mvin` 是 `accType` 类型，则该位为 0；如果是 `inputType` 类型，该位为 1
- `rs1[4:3]` 如果是为 `mvin` 设置步长，则该位为 0；如果为 `mvin2` 设置步长，该位为 1；如果为 `mvin3` 设置步长，该位为 2
- `rs1[31:16]` 是暂存器内存步长（即上面也称为“专用内存步长”）
- `rs1[63:32]` 是数据被移动到暂存器时要乘以的“比例/缩放 (scale)”。如果 Gemmini 没有配置为在 `mvin` 期间具备缩放值的能力，则此项被忽略。
- `rs2` 是以字节为单位的主存步长
- `funct` = 0

**操作：** stride <= rs2; scale <= rs1[63:32]

### `config_mvout` 配置存储流水线
**格式：** `config_mvout rs1 rs2`
- `rs1[1:0]` 必须是 `10`
- `rs2` = 以字节为单位的步长
- `funct` = 0

在 `mvout` 操作期间，Gemmini 还可以执行最大池化。
**这是一项实验性功能，随时可能发生变化。**
该功能假定数据以 NHWC 格式存储在暂存器或累加器中。
控制该功能的参数为：

- `rs1[5:4]` = 最大池化步长。如果为 0，则停用最大池化。
- `rs1[7:6]` = 最大池化窗口大小
- `rs1[9:8]` = 上方零填充 (zero-padding)
- `rs1[11:10]` = 左侧零填充
- `rs1[31:24]` = 池化后图像的输出维度
- `rs1[39:32]` = 要输出的池化行数
- `rs1[47:40]` = 要输出的池化列数
- `rs1[55:48]` = 要池化的未池化行数
- `rs1[63:56]` = 要池化的未池化列数

**操作：** stride <= rs2; max-pooling parameters <= rs1

### `config_norm` 配置归一化命令
**格式：** `config_norm rs1 rs2`

`config_norm` 是一项**实验性**命令，主要是为了支持 Gemmini 上称为 [I-BERT](https://arxiv.org/abs/2101.01321) 的仅整数 BERT 变体。
该命令允许用户设置用于 I-BERT 的 GELU、layernorm 和 softmax 变体的标量常量。

### `flush` 刷新 TLB
**格式：** `flush rs1`
- `rs1` = 如果 `rs1[0]` 为 1，则跳过当前的 TLB 请求（如果其触发了缺页中断并且正在等待中断）。
否则，重复当前的 TLB 请求。

**注意：**

- 此指令在*接收到后立即执行*，无需等待可能排队的其他指令。
如果需要，程序员有责任插入 fence（内存栅栏）指令。

## 核心矩阵乘法序列
每一次矩阵乘法操作都是 `matmul.preload` 和 `matmul.compute` 的组合（由于单条指令长度的限制，它被拆分成了两条指令）。
`matmul.preload` 应该在 `matmul.compute` 之前执行。

示例：
```
//// OS 矩阵乘法示例 ////
// rs1 = InputD
// rs2 = OutputC
// rs3 = InputA
// rs4 = InputB
// matmul InputA InputB OutputC InputD
1. matmul.preload $rs1 $rs2
2. matmul.compute $rs3 $rs4
```
**操作：** Scratchpad[rs2] <= Scratchpad[rs3] \* Scratchpad[rs4] + Scratchpad[rs1]

**寻址注意：**
- 对于 B 或 D，地址可以替换为全高位，以输入一个 0 矩阵。
- 对于 A，地址可以替换为全高位，以输入一个包含未定义垃圾数据的矩阵。

### 预加载 (Preloading)
**格式：** `matmul.preload rs1, rs2`
- `rs1[31:0]` = D 矩阵（输出固定时）或 B 矩阵（权重固定时）的本地暂存器地址
- `rs1[47:32]` = D/B 矩阵的列数
- `rs1[63:48]` = D/B 矩阵的行数
- `rs2[31:0]` = C 矩阵的本地暂存器地址。
如果设置为全高位，则 C 不会被写入暂存器或累加器。
- `rs2[47:32]` = C 矩阵的列数
- `rs2[63:48]` = C 矩阵的行数
- `funct` = 6

**提交行为 (Commit Behavior)：** 此指令在脉动阵列接收到它的后一个周期提交。脉动阵列将保持空闲，直到看到随后的 OS/WS 进行特异指令。

### 计算 (Computing)
#### 显式预加载 (Explicitly Preloaded)
**格式：** `matmul.compute.preloaded rs1, rs2`
- `rs1[31:0]` = A 矩阵的本地暂存器地址（脉动阵列单轴寻址）
- `rs1[47:32]` = A 矩阵的列数
- `rs1[63:48]` = A 矩阵的行数
- `rs2[31:0]` = B 矩阵（输出固定时）或 D 矩阵（权重固定时）的本地暂存器地址（脉动阵列单轴寻址）
- `rs2[47:32]` = B/D 矩阵的列数
- `rs2[63:48]` = B/D 矩阵的行数
- `funct` = 4
- 该指令将对预加载的值（如果输出固定则为 D，如果权重固定则为 B）进行计算

#### 复用先前的预加载 (Re-use Previous Preloads)
**格式：** `matmul.compute.accumulated rs1, rs2`
- `funct` = 5
- `rs1` 和 `rs2` 具有与 `matmul.compute.preloaded` 相同的编码
- 如果为输出固定模式，该指令将在脉动阵列中之前计算出的结果 (C) 的基础上进行计算并在其上进行累加
- 如果为权重固定模式，该指令将在脉动阵列中之前预加载的权重 (B) 的基础上进行计算

## 循环指令

Gemmini 包含 CISC 类型的指令，可以在比 `DIM x DIM` 大得多的数据上执行矩阵乘法和卷积。

这些 CISC 指令能做的事，程序员也可以通过分块并在上述其他 ISA 指令中进行循环来完成；
然而，这些 CISC 指令可以实现比非专家程序员编写的这种分块循环更高的吞吐量。
CISC 指令应该被视为性能增强器；它们没有赋予加速器任何以前没有的新功能。

CISC 指令有太多的操作数，无法放入单个 RISC-V 自定义指令中。
    - 因此，它们被实现为许多 RISC-V 自定义指令的序列，程序员必须连续调用它们。

这些指令可以在 [gemmini.h](file:///Users/yi/project/gemmini/software/gemmini-rocc-tests/include/gemmini.h) 中找到，并附有示例用法。
我们在下面列出它们的参数。

**这些循环指令处于实验阶段，随时可能发生变化。**

### `gemmini_loop_ws` 矩阵乘法循环 (WS 数据流)

此指令计算 `A * B + D = C`，但 `A`、`B`、`D` 和 `C` 都可以大于 `DIM x DIM`。
`A` 和 `B` 必须是 `inputType` 类型，但 `D` 和 `C` 均可以为 `inputType` 或 `accType` 类型。

这些矩阵的大小由 `I`、`J` 和 `K` 表示：

```
A 的暂存器行数 = I * K * DIM
B 的暂存器行数 = K * J * DIM
D 的累加器行数 = I * J * DIM
C 的累加器行数 = I * J * DIM
```

然而，单个 `gemmini_loop_ws` 占用的暂存器行总数必须最多为总暂存器大小的**一半**，因为 Gemmini 在 CISC 指令期间会执行双缓冲。
要计算更大的矩阵乘法，循环指令也必须在外层循环内进行分块。

为了支持 `gemmini_loop_ws` 指令的外层分块，我们引入了一个名为 `ex_accumulate` 的参数，该参数确定是否在累加器中已存在的偏部分和之上执行矩阵乘法（来自同一外层循环中先前对 `gemmini_loop_ws` 的调用）。

### `gemmini_loop_conv_ws` 卷积循环 (WS 数据流)

Gemmini 还包含一个用于卷积的 CISC 指令，其实现类似于矩阵乘法 CISC 指令。
`gemmini_loop_conv_ws` 将使用 WS 数据流执行卷积，还支持诸如最大池化、转置卷积以及对权重和输入数据进行各种预处理转换等功能。

与 `gemmini_loop_ws` 类似，单个 `gemmini_loop_conv_ws` 调用的输入必须容纳在 Gemmini 专用内存的一半以内，以支持双缓冲。
如果程序员希望执行更大的卷积，他们必须将 `gemmini_loop_conv_ws` 进行分块并包装在外层循环中。

# 引用 Gemmini
如果 Gemmini 对您的学术研究有所帮助，鼓励您引用我们的论文。以下是 BibTeX 示例：
```
@INPROCEEDINGS{gemmini-dac,
  author={Genc, Hasan and Kim, Seah and Amid, Alon and Haj-Ali, Ameer and Iyer, Vighnesh and Prakash, Pranav and Zhao, Jerry and Grubb, Daniel and Liew, Harrison and Mao, Howard and Ou, Albert and Schmidt, Colin and Steffl, Samuel and Wright, John and Stoica, Ion and Ragan-Kelley, Jonathan and Asanovic, Krste and Nikolic, Borivoje and Shao, Yakun Sophia},
  booktitle={Proceedings of the 58th Annual Design Automation Conference (DAC)}, 
  title={Gemmini: Enabling Systematic Deep-Learning Architecture Evaluation via Full-Stack Integration}, 
  year={2021},
  volume={},
  number={},
  pages={}
}
```

# 致谢

- 本项目由美国政府在 DARPA RTML 项目下提供部分资助（合同号：FA8650-20-2-7006）。本文档中包含的观点和结论属于作者个人，不应被解释为代表美国官方政策的明示或暗示。
- Gemmini [logo](./img/full-logo.svg) 由 Dima Nikiforov 设计（[@CobbledSteel](https://github.com/CobbledSteel)）。
