# verilog-axi 系统学习路线

> 本文面向有基本数字电路和 Verilog 知识、每周可投入 6～8 小时、AXI 与 cocotb 经验有限的学习者。时间是工作量估计，不是学会的承诺。路线以当前 checkout 为准，当前提交为 `516bd5d`。

## 0. 先读这份说明

### 项目状态与调查边界

这是一个可复用的 AXI4/AXI4-Lite RTL 组件库，不是带 CPU、操作系统或板级工程的完整 SoC。README 明确声明仓库已被 `fpganinja/taxi` 取代，后续新功能和修复不会在本仓库继续维护。因此本仓库很适合学习成熟的 Verilog 组件、握手和验证方法；若用于新产品，应另外评估 `taxi` 的接口、许可证、验证状态和迁移成本，不能把本仓库当作有持续支持的产品依赖。

本路线核对了 `README.md`、`rtl/` 的模块入口和主要参数、`tb/` 的 cocotb/pytest 测试目录、顶层旧测试、`syn/vivado/` 的约束脚本、`tox.ini`、`tb/Makefile`、代表性模块 Makefile 和 GitHub Actions 配置。没有逐行审查所有 RTL、所有测试和所有综合脚本；未核实的行为在文中标为“待核实”，不能由模块名推断功能。

### 全局地图

| 目录/文件 | 实际用途 | 学习重点 |
|---|---|---|
| [`rtl/`](rtl) | 可综合 Verilog-2001 组件；大量模块按读、写方向拆分，再由 wrapper 合并 | 接口、参数、握手、状态和数据通路 |
| [`tb/`](tb) | 每个模块的 cocotb 测试、pytest/cocotb-test 入口、旧的顶层测试 | 激励、随机停顿、背压、断言式结果检查 |
| [`syn/`](syn) | 当前可见内容主要是 Vivado 的 CDC 时序约束脚本 | CDC 同步器的实现边界与 STA 约束 |
| [`tox.ini`](tox.ini) | Python 3.10、pytest、cocotb、cocotbext-axi 等版本和 pytest 收集配置 | 可重复运行测试的基线 |
| [`.github/workflows/`](.github/workflows) | Ubuntu 22.04 CI，安装 Icarus 后按 10 组拆分 pytest | CI 与本地命令的差异 |

README 列出的模块与实际 RTL 文件大体对应，但 README 的“Testing”只给出依赖概述；实际 pytest 配置 `addopts` 使用 `--ignore-glob=tb/test_*.py`，因此默认 pytest 主要收集 `tb/<模块目录>/test_*.py`。顶层 `tb/test_*.py` 仍在仓库中，但不能假设它们会被默认命令执行。

### 模块分层与优先级

**必须精读（主线）：**

1. [`rtl/axil_ram.v`](rtl/axil_ram.v) 与 [`tb/axil_ram/test_axil_ram.py`](tb/axil_ram/test_axil_ram.py)：最小 AXI-Lite 存储目标，先建立读写握手、WSTRB 和响应概念。
2. [`rtl/axi_register.v`](rtl/axi_register.v)、[`rtl/axi_register_rd.v`](rtl/axi_register_rd.v)、[`rtl/axi_register_wr.v`](rtl/axi_register_wr.v) 与 [`tb/axi_register/test_axi_register.py`](tb/axi_register/test_axi_register.py)：wrapper、读写拆分、simple/skid buffer。
3. [`rtl/axi_ram.v`](rtl/axi_ram.v)、`axi_ram_*_if.v` 与 [`tb/axi_ram/test_axi_ram.py`](tb/axi_ram/test_axi_ram.py)：AXI4 burst、窄传输和内存接口。
4. [`rtl/axi_adapter.v`](rtl/axi_adapter.v)、`axi_adapter_rd.v`、`axi_adapter_wr.v` 与 [`tb/axi_adapter/test_axi_adapter.py`](tb/axi_adapter/test_axi_adapter.py)：宽度转换及读写通路。
5. [`rtl/axi_interconnect.v`](rtl/axi_interconnect.v)、[`rtl/axi_crossbar.v`](rtl/axi_crossbar.v) 及其 `*_rd.v`、`*_wr.v`、`*_addr.v`，配合 [`tb/axi_interconnect/test_axi_interconnect.py`](tb/axi_interconnect/test_axi_interconnect.py) 和 [`tb/axi_crossbar/test_axi_crossbar.py`](tb/axi_crossbar/test_axi_crossbar.py)：译码、仲裁、并发与顺序。
6. [`rtl/axi_dma.v`](rtl/axi_dma.v)、`axi_dma_rd.v`、`axi_dma_wr.v` 与 [`tb/axi_dma/test_axi_dma.py`](tb/axi_dma/test_axi_dma.py)：内存与 AXI-Stream 之间的搬运。

**对照阅读：**

- `axil_register*`、`axil_adapter*`、`axil_interconnect*`、`axil_crossbar*`：将 AXI4 主线中的概念映射到 AXI4-Lite；不要重复逐行阅读。
- `axi_cdma*` 与 [`tb/axi_cdma/test_axi_cdma.py`](tb/axi_cdma/test_axi_cdma.py)：与 DMA 的描述符和方向差异。
- `axi_fifo*` 与 [`tb/axi_fifo/test_axi_fifo.py`](tb/axi_fifo/test_axi_fifo.py)：事务缓冲和地址延迟。
- `axil_reg_if*`、`axil_dp_ram*`、`axi_dp_ram*`：在理解基础握手后阅读。

**专题/按需深入：**

- [`rtl/axil_cdc.v`](rtl/axil_cdc.v) 与 [`tb/axil_cdc/test_axil_cdc.py`](tb/axil_cdc/test_axil_cdc.py)：跨时钟域；再读 [`syn/vivado/axil_cdc.tcl`](syn/vivado/axil_cdc.tcl)。
- `axi_vfifo.v`、`axi_vfifo_raw.v`、`axi_vfifo_enc.v`、`axi_vfifo_dec.v` 与 `tb/axi_vfifo*`：虚拟 FIFO 的分段和编码，依赖较多。
- `arbiter.v`、`priority_encoder.v` 和 `*_wrap.py`：抽象基础与 wrapper 生成，不是第一周主线。

真实依赖要以 RTL 实例化为准。例如 crossbar 的读写子模块会实例化地址译码、寄存器等组件；这说明源码依赖关系，但不等于建议的学习顺序。建议顺序是按概念复杂度安排的。

## 1. 入门自测与先修知识

### 30 分钟自测

- 能否写出 `valid && ready` 发生一次传输，并说明发送端在 `ready=0` 时数据和控制信号必须保持稳定？
- 能否解释 AXI4-Lite 写地址和写数据通道可以独立到达？
- 能否区分组合逻辑、时序逻辑、非阻塞赋值和同步复位？
- 能否读懂 Verilog 参数、`generate`、位切片和一个两状态 FSM？
- 能否用 Python 创建虚拟环境、运行 pytest，并定位一个 cocotb 测试的 DUT 和 HDL 源文件？

全部能回答可从阶段 1 开始；只会 Verilog 但不懂握手，先完成阶段 0；不会 Python/pytest 的读者只补工具用法，不要跳过协议和波形任务。

### 必要知识与项目锚点

| 知识 | 最低掌握目标 | 在本项目中的落点 |
|---|---|---|
| Verilog 时序基础 | 能区分 `always @*` 与 `always @(posedge clk)`，理解寄存器保持 | `axil_ram.v` 的 ready/valid 状态寄存器 |
| 参数化/generate | 能跟踪 DATA/ADDR/STRB/ID 宽度和条件生成 | `axi_register_rd.v` 的 `*_REG_TYPE` |
| ready/valid | 能按周期判断传输、背压和稳定性 | 所有 AXI 通道 |
| AXI4-Lite | 读写地址、数据、响应，WSTRB | `axil_ram.v`、`axil_register*` |
| AXI4 | burst、LEN/SIZE/LAST、ID、outstanding、顺序 | `axi_ram.v`、crossbar、DMA |
| AXI-Stream | `tvalid/tready/tdata/tkeep/tlast` 及背压 | `axi_dma_*` |
| Python/cocotb | coroutine、Clock、RisingEdge、随机激励 | `tb/axil_ram/test_axil_ram.py` |
| 波形 | 能从 `valid/ready` 和状态寄存器解释一个事务 | Icarus FST/VCD 或其他仿真波形 |

还要理解 AXI4 支持的协议空间不等于某个模块实现的全部空间。比如 README 说某些组件支持特定 burst 或窄传输，仍要以参数、状态机和对应测试核实。

## 2. 环境、命令与首个实验

### 环境事实

`tox.ini` 固定的起点包括 Python 3.10、pytest 7.2.1、pytest-xdist 3.1.0、pytest-split 0.8.0、cocotb 1.7.2、cocotb-bus 0.2.1、cocotb-test 0.2.4、cocotbext-axi 0.1.24、Jinja2 3.1.2；CI 使用 Ubuntu 22.04 和 Icarus Verilog。当前任务没有安装或运行这些依赖，因此下面是配置推导的命令，不是已验证通过的结果。

推荐在 WSL/Linux 中运行 Makefile 和 Icarus 命令；PowerShell 可以调用 Python/pytest，但仓库 Makefile 含 Unix shell 语法，不能直接把 `make`、`rm`、`$(...)` 当作 PowerShell 语法。Windows 原生环境是否能无修改运行，取决于已安装的 make、Icarus 和 cocotb 配置，待核实。

### 首次运行建议（WSL/Linux）

```bash
cd /mnt/c/Users/Administrator/Desktop/verilog-axi/verilog-axi
python3.10 -m venv .venv
source .venv/bin/activate
python -m pip install -r <(python - <<'PY'
print('pytest==7.2.1')
print('pytest-xdist==3.1.0')
print('pytest-split==0.8.0')
print('cocotb==1.7.2')
print('cocotb-bus==0.2.1')
print('cocotb-test==0.2.4')
print('cocotbext-axi==0.1.24')
print('jinja2==3.1.2')
PY
)
pytest tb/axil_ram/test_axil_ram.py -q
```

上面的临时 requirements 写法是方便复制的示例；也可以按 `tox.ini` 手动安装。Icarus 必须由系统包管理器安装，版本应与 cocotb 兼容。更接近仓库配置的方式是 `tox`，但第一次学习不建议先跑全量 CI 分组。

默认 pytest 会忽略 `tb/test_*.py`，所以应显式指定 `tb/axil_ram/test_axil_ram.py` 或先运行 `pytest --collect-only -q` 查看收集结果。要运行单个测试参数，可先看文件末尾的 `@pytest.mark.parametrize`，例如 axil RAM 的 `data_width`：

```bash
pytest tb/axil_ram/test_axil_ram.py -q -k 'data_width_32'
pytest tb/axil_ram/test_axil_ram.py --collect-only -q
```

具体 node id 由 pytest 生成，应以 `--collect-only` 输出为准，不要猜测名称。Makefile 方式适合在对应目录运行，例如 `make -C tb/axil_ram`；参数、仿真器和波形开关要以该目录 Makefile 为准。代表性 Makefile 使用 `SIM=icarus`、`WAVES=1` 和 `iverilog_dump.v` 生成 FST，但当前未实际执行，波形文件名和查看器兼容性待核实。

故障定位顺序：确认 Python 包版本 → 确认 `iverilog` 和 `cocotb-config` 在 PATH → `pytest --collect-only` → 看 cocotb-test 生成的 sim build 日志 → 单独编译最小 DUT → 打开波形核对时钟、复位、valid/ready → 再检查参数和路径。

## 3. 主线阶段

每阶段都要留下一个可检查的产物。若验收失败，回到表中对应的前置阶段，而不是只增加阅读量。

### 阶段 0：协议与工具基础（6～10 小时，约 1～2 周）

**目标/前置：**能用周期表解释一个 AXI-Lite 读写，理解 ready/valid 和背压；需要基本 Verilog。

**阅读顺序：**README 的 common signals/parameters → `tb/axil_ram/test_axil_ram.py` 的 TB、Clock、master → `rtl/axil_ram.v` 的端口、参数、读写两个 `always` 块。

**追踪：**`s_axil_awvalid/ready`、`wvalid/ready`、`bvalid/bready`、`arvalid/ready`、`rvalid/rready`、`mem_wr_en`、`mem_rd_en`、`PIPELINE_OUTPUT`、`WSTRB`。

**实验：**画出一次 AW/W 同周期写、AW 先到、读响应被 backpressure 的周期表；在测试中开启 idle/backpressure generator，记录每次实际传输。

**波形：**复位释放、地址和数据保持、`valid && ready` 的传输周期、响应保持周期、WSTRB 对应字节。

**产物与验收：**一页协议时序图、一份 10 行事务解释。随机暂停时能指出每个传输周期，并能说明为何 `valid` 不能因 `ready=0` 而撤销。否则回补 ready/valid 和 Verilog 时序。

**误区：**把 `valid` 当成请求脉冲；认为 AW 和 W 必须同周期；把 `bvalid` 拉高当成响应已经被接收。

### 阶段 1：AXI-Lite RAM、register 与寄存器接口（8～12 小时，约 2 周）

**目标/前置：**能读懂简单存储目标和流水缓冲；完成阶段 0。

**阅读顺序：**`axil_ram.v` → `axil_register_rd.v`/`wr.v` → `axil_register.v` wrapper → `axil_reg_if.v`、`axil_reg_if_rd.v`、`wr.v`；测试依次看 `tb/axil_ram`、`tb/axil_register`、`tb/axil_reg_if`。

**追踪：**地址到内存索引的换算、WSTRB byte lane、`PIPELINE_OUTPUT`、`*_REG_TYPE`、寄存器占用/释放条件、读写独立状态。

**设计问题：**为什么 AXI-Lite RAM 可等待 AW 和 W 同时有效？skid buffer 如何避免有气泡？输出寄存器增加了什么延迟和容量？

**实验：**用 8/16/32 位参数分别做全字写和部分 WSTRB 写；对 register 的 bypass/simple/skid 配置运行相同事务，统计首拍延迟和连续吞吐。已有测试覆盖的组合以测试参数为准，新增组合需自行标注。

**波形/产物：**地址、WSTRB、内存写使能、缓冲有效位和响应；提交参数对照表、波形截图或信号记录、测试结果。

**验收：**能解释部分写为何只更新目标 byte lane；能从波形指出 buffer 满/空和输入输出握手；不能解释就回补阶段 0 和 `axi_register_rd.v` 的 generate 分支。

### 阶段 2：AXI4 RAM 与 burst（10～16 小时，约 2～3 周）

**目标/前置：**从单拍 AXI-Lite 过渡到 AXI4 的 burst、窄传输和 `LAST`；完成阶段 1。

**阅读顺序：**`axi_ram.v` → `axi_ram_rd_if.v`/`axi_ram_wr_if.v`/`axi_ram_wr_rd_if.v` → `tb/axi_ram/test_axi_ram.py`；再对照 `axi_dp_ram.v` 与 `tb/axi_dp_ram`。

**追踪：**`awlen/arlen`、`awsize/arsize`、`awburst/arburst`、`wlast/rlast`、地址递增、WSTRB、ID、读写 outstanding 和内存接口握手。

**设计问题：**burst 长度是如何转换为 beat 数的？窄传输的有效 byte lane 如何变化？读数据和写响应如何保持 ID/顺序？模块实际是否支持所有 README 声称的 burst？用 RTL 和测试证据回答。

**实验：**单拍、INCR burst、不同 SIZE、非对齐地址和跨 4KB 边界分别测试；把支持范围写成“已测试/源码可见但未测试/待核实”。不把 AXI 规范允许的场景自动当成模块支持。

**波形/产物：**地址 beat、`wstrb`、`wlast`、`rvalid/rready/rlast`、内部计数器和响应 ID；提交 burst 周期表和失败场景记录。

**验收：**能按任意一个 burst 逐 beat 计算地址和有效字节；能指出 4KB 边界是否由该模块主动处理。无法做到则回补地址/size/len 和阶段 1 的 byte lane 实验。

### 阶段 3：宽度转换与 AXI/AXI-Lite 适配（8～14 小时，约 2 周）

**目标/前置：**能解释不同数据宽度和协议转换导致的拆分、合并、响应映射。

**阅读顺序：**`axi_adapter.v` → `axi_adapter_rd.v`/`wr.v` → `axil_adapter.v` 及方向文件 → `axi_axil_adapter.v` 及方向文件；测试看 `tb/axi_adapter`、`tb/axi_axil_adapter`、`tb/axil_adapter`，并核对顶层旧测试是否被默认 pytest 忽略。

**追踪：**源/目标 DATA_WIDTH、STRB_WIDTH、地址低位、beat 计数、响应聚合、窄 burst 和非对齐路径。

**实验：**32→16、16→32、32→32 各跑全字/部分写/读 burst；在两端分别加入 idle/backpressure，证明转换器不会丢 beat 或提前结束。

**验收：**能画出一个源 beat 被拆成多个目标 beat 的时序，并解释 `LAST` 和 response 何时产生。不能解释时回补阶段 2。

### 阶段 4：interconnect、crossbar、仲裁与顺序（12～20 小时，约 2～3 周）

**目标/前置：**理解共享互连与 nonblocking crossbar 的吞吐、译码和顺序保护差异。

**阅读顺序：**`arbiter.v`/`priority_encoder.v` → `axi_interconnect.v` 及 `*_rd/*_wr`（如存在）→ `axi_crossbar_addr.v` → `axi_crossbar_rd.v`/`wr.v` → `axi_crossbar.v` wrapper；测试看 `tb/axi_interconnect`、`tb/axi_crossbar`，再对照 AXI-Lite 版本。

**追踪：**地址范围参数、master/slave 选择、grant、 admission control、decode error、ID ordering、读写独立路径和各端口寄存器。

**实验：**两 master 访问两个 slave 的并发读写；制造同一 ID 的顺序压力、不同 ID 的并发、无匹配地址和目标 backpressure。分别记录 shared interconnect 与 crossbar 的等待关系。

**波形/产物：**译码结果、仲裁 grant、各端口 valid/ready、ID FIFO/排序状态、错误响应；提交地址映射图和事务时序图。

**验收：**能说明“共享互连不支持并发”与“crossbar nonblocking”的证据，能定位 decode error 来源。不能说明就回补阶段 2 的 ID/顺序，再读 `*_addr.v`。

### 阶段 5：FIFO、DMA/CDMA 与 AXI-Stream（14～24 小时，约 3～4 周）

**目标/前置：**能从描述符或控制请求追踪到 AXI burst 与 Stream 数据，理解吞吐和背压。

**阅读顺序：**`axi_fifo.v`、`axi_fifo_rd.v`、`axi_fifo_wr.v` → `axi_dma_rd.v`/`wr.v` → `axi_dma.v` → `axi_dma_desc_mux.v` → `axi_cdma*`；测试看 `tb/axi_fifo`、`tb/axi_dma_rd`、`tb/axi_dma_wr`、`tb/axi_dma`、`tb/axi_cdma`。

**追踪：**最大 burst、地址/长度、未对齐 offset、AXI `LAST`、Stream `tkeep/tlast`、描述符 ready/valid、输出 FIFO 和错误/完成状态。

**实验：**对齐和非对齐内存搬运；Stream 端插入随机 backpressure；比较 FIFO 延迟选项；构造短包、长 burst、边界长度和中途复位（仅在测试语义允许时）。

**验收：**能用波形解释一个描述符如何变成若干 burst 和 Stream beat，能区分数据正确性与吞吐问题。没有综合报告时不得声称资源或频率收益。

### 阶段 6：CDC 与 virtual FIFO 专题（10～18 小时，约 2～3 周）

**目标/前置：**在已掌握握手、FIFO 和 DMA 后，分析跨时钟域与分段存储。

**阅读顺序：**`axil_cdc.v`、`axil_cdc_rd.v`、`axil_cdc_wr.v` → `tb/axil_cdc` → `syn/vivado/axil_cdc.tcl`；再读 `axi_vfifo_raw*`、`axi_vfifo_enc/dec.v`、`axi_vfifo.v` 和对应 `tb/axi_vfifo*`。

**追踪：**各时钟域的同步标志、握手保持、复位、异步 FIFO/分段地址、编码器/解码器状态和边界条件。

**实验：**使用不同源/目的时钟比率运行 CDC 测试；改变复位相位并记录可重复现象。CDC 仿真不能证明亚稳态安全、MTBF 或真实 STA，必须把这些结论留给同步器结构、Vivado 约束和硬件验证。

**验收：**能解释哪些信号跨域、如何同步、为什么 `ASYNC_REG` 和 timing constraint 必要；能指出仿真无法证明的事项。不能做到则回补阶段 5 的 FIFO/背压。

## 4. 递进实验与受控调试

按以下顺序扩展同一个最小测试，而不是一次写一个大 testbench：

1. 单次读写、部分 WSTRB 写。
2. AW/W 分离到达、随机 idle、响应和读数据 backpressure。
3. register bypass/simple/skid 的延迟与吞吐对比。
4. AXI burst、窄传输、非对齐和 4KB 边界；每项先确认模块实际支持范围。
5. 多端口译码、仲裁、并发访问、无匹配地址错误响应。
6. DMA/CDMA 描述符、搬运、`tkeep/tlast`、完成响应。
7. CDC 独立时钟和复位；标出仿真不能证明的 CDC 属性。

每次实验记录：参数、随机种子、期望事务、实际握手周期、关键波形、结果和未覆盖条件。已有测试和新增测试分开标记。

**受控故障注入：**选 `axil_ram` 或 `axi_register`，先预测“把 ready 条件提前一拍”或“屏蔽一个 WSTRB lane”会造成的行为；复制测试后只改一个 RTL 条件，运行使其失败，使用波形定位第一个错误周期，再恢复原逻辑并记录根因。此路线不在当前任务中修改 RTL。

## 5. 复杂 RTL 阅读模板

对每个模块复制以下记录，不要只逐行翻译：

```text
模块/版本：
用途与不支持范围（证据文件）：
上游/下游接口：
参数及其影响：
读路径：输入握手 -> 地址/数据寄存器 -> 输出握手
写路径：地址/数据接收 -> 内存/下游 -> 响应
关键状态/计数器：
背压如何向上游传播：
ID、LAST、WSTRB 或 tkeep 如何保持：
正常场景波形：
边界/错误场景波形：
对应测试与未覆盖项：
我能解释的行为：
仍待核实的问题：
```

阅读 `axi_crossbar_rd.v` 时先画端口和 `axi_crossbar_addr` 的关系，再跟踪一次 AR 从输入到输出、一次 R 返回；阅读 `axi_dma_rd.v` 时先标出描述符/地址/beat 计数器，再跟踪一条 AXI R 到 AXI-Stream 输出。这样能把接口、数据通路、控制通路、状态更新和背压联系起来。

## 6. 综合项目与最终验收

### 主线项目：可验证的 AXI 外设子系统

使用现有模块搭建一个可仿真的小系统：一个 AXI-Lite 控制寄存器块（`axil_reg_if` 或 `axil_ram`）、一个 AXI RAM/DP RAM 数据区、一个 register 或 FIFO、一个 interconnect 或小型 crossbar。自定但固定地址映射，例如控制区 `0x0000_0000`、数据区 `0x0001_0000`；实际映射必须与参数和译码实现一致。

实施步骤：先单独验证每个 slave → 写地址译码和错误响应 → 加 register/FIFO 并验证背压 → 为数据区加入 burst → 用 cocotb 产生正常、部分写、随机停顿和并发访问 → 生成波形并写出至少三条事务解释。提交顶层 RTL、测试、README、地址映射表、参数表和已知限制。

### 进阶选做：DMA 数据搬运链

把 AXI RAM、`axi_dma` 和 AXI-Stream sink/source 接起来，实现一个描述符驱动的固定长度搬运；加入非对齐和 Stream backpressure 场景。只使用源码和测试实际支持的参数，未验证的 4KB/错误恢复能力明确列为待核实。

### 四层验收

| 层次 | 通过标准 |
|---|---|
| 能使用 | 能按项目配置运行一个小测试，选择参数并解释失败日志；未验证环境问题要明确记录 |
| 能解释 | 能从波形逐周期说明一次读、写、burst、背压和错误响应 |
| 能修改 | 能提出一个小功能/故障修复，先写预期，再用测试和波形证明影响 |
| 能集成与排错 | 能连接两个以上现有组件，处理地址映射、参数、响应和背压，并定位一次故障 |

未通过时，按现象回补：握手错误回阶段 0；byte lane/burst 错误回阶段 1～2；转换错误回阶段 3；并发/顺序错误回阶段 4；数据搬运错误回阶段 5；跨域问题回阶段 6。不要用“再读一遍所有 RTL”替代针对性回补。

## 7. 专题索引与外部资料

完成主线后再按需要深入：

- AXI-Lite 控制路径：`axil_reg_if*`、`axil_register*`、`axil_crossbar*`。
- AXI 内存与并发：`axi_ram*`、`axi_dp_ram.v`、`axi_interconnect*`、`axi_crossbar*`。
- 数据搬运：`axi_dma*`、`axi_cdma*`、descriptor mux。
- 缓冲与吞吐：`axi_fifo*`、register/skid buffer。
- CDC：`axil_cdc*` 与 `syn/vivado/axil_cdc.tcl`。
- 生成与工程化：`rtl/*_wrap.py`、Makefile、GitHub Actions。

协议细节优先查 ARM AMBA AXI/ACE 协议规范，工具行为查 cocotb、cocotbext-axi、Icarus 和 pytest 官方文档；使用的版本应与 `tox.ini` 对齐。当前任务没有执行安装、编译或仿真，因此所有运行结果仍待在目标环境中核实。
