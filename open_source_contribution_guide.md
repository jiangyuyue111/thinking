# 内核与计算机视觉：开源贡献项目指南

> 整理日期：2026-09-09  
> 目标：找到能够实际参与、形成可验证成果，并能用于简历和面试的开源项目。  
> 默认基础：会一些 C/C++ 或 Python，首次参与开源。本文精选 16 个项目，覆盖 GitHub、GitLab、Gitee、AtomGit，以及 kernel.org 邮件协作体系。

**建议先选 1 个主项目，连续投入 6—8 周。** 第一阶段争取做出“问题复现 → 原因分析 → 修复或测试 → 社区评审”的完整成果，再逐步深入。项目名气可以帮助说明背景，真正能支撑面试的是你解决了什么问题、如何验证，以及哪些成果进入了上游。

本文的项目定位和贡献流程附有官方来源；难度、优先级、任务切入点和时间安排是面向初次贡献者的分析建议，不是项目官方评级。Issue 状态依据本次检索返回的页面，页面可能存在缓存；开始前需要重新检查状态、负责人和关联 PR。本文没有逐仓运行代码，也不承诺维护者响应时间或补丁一定合并。

## 目录

- [1. 先选适合自己的路线](#路线选择)
- [2. 平台与提交方式](#平台入口)
- [3. 内核与系统方向：8 个项目](#内核项目)
- [4. 计算机视觉与推理方向：8 个项目](#视觉项目)
- [5. 具体任务线索与筛选结果](#任务线索)
- [6. 如何找到值得投入的 Issue](#筛选方法)
- [7. 8 周执行路线](#执行路线)
- [8. 如何沉淀为项目经历和简历](#简历成果)
- [9. 导师计划与长期入口](#导师计划)
- [10. 可复制的贡献记录模板](#记录模板)

<a id="路线选择"></a>

## 1. 先选适合自己的路线

| 你的基础或目标 | 优先选择 | 第一项成果建议 | 后续深入方向 |
| --- | --- | --- | --- |
| 会 C，希望进入嵌入式或操作系统方向 | RT-Thread、Zephyr | 一个组件的缺陷复现、回归测试和修复 | 驱动、同步原语、调度、内存管理 |
| 会 Rust，学过操作系统课程 | ArceOS、Asterinas | 一个库函数或系统调用的边界行为测试 | 文件系统、进程、虚拟内存、设备抽象 |
| 目标是 Linux 内核、服务器系统岗位 | Linux、openEuler Kernel | 一个 KUnit/selftests 测试或明确的小缺陷 | 固定一个子系统持续提交补丁 |
| 会 Python、NumPy，深度学习基础较少 | scikit-image | 图像处理函数的边界用例与修复 | 数值稳定性、图像处理算法、性能分析 |
| 会 PyTorch，希望做视觉算法工程 | Kornia、Torchvision | 数据增强、几何变换或数据集接口的回归测试 | 可微算子、数据管线、数值与梯度正确性 |
| 想做 OCR、目标检测等可展示的应用 | PaddleOCR、Ultralytics | 从真实样本发现并修复输入、后处理或导出问题 | 训练评估、识别质量、部署一致性 |
| C++ 较好，希望结合底层与 AI 部署 | ncnn；随后考虑 ONNX Runtime | 算子精度对齐、模型转换复现和性能基线 | SIMD、量化、内存优化、推理后端 |

**如果还没有明确偏好：内核方向先比较 RT-Thread 与 Zephyr；视觉方向先比较 Kornia 与 PaddleOCR。** 喜欢数学和图像处理、又没有 GPU，可以从 scikit-image 开始。已经学过 Rust 操作系统实验，可以直接考虑 ArceOS。

“内核”在本文主要指操作系统内核。ncnn、ONNX Runtime 中的计算算子 kernel 属于推理执行代码；QEMU 属于模拟器和虚拟化基础设施。它们与底层开发有关，但简历里应准确区分。

<a id="平台入口"></a>

## 2. 平台与提交方式

| 平台或协作体系 | 本文代表项目 | 从哪里进入 | 提交时注意什么 |
| --- | --- | --- | --- |
| GitHub | Zephyr、RT-Thread、NuttX、Asterinas、ArceOS，以及本文视觉项目 | 项目仓库的 Issues、Pull requests、贡献指南 | 阅读目标分支规则，先搜索重复问题和已有 PR |
| GitLab | QEMU | [代码仓库](https://gitlab.com/qemu-project/qemu/)、[问题列表](https://gitlab.com/qemu-project/qemu/-/issues) | QEMU 官方要求通过邮件提交补丁，不接受用 GitLab MR 代替该流程 |
| Gitee | RT-Thread | [RT-Thread Gitee 仓库](https://gitee.com/rtthread/rt-thread) | 可阅读代码；本指南按 RT-Thread 官方贡献文档指向的 GitHub PR 流程组织，不在两个平台重复提交 |
| AtomGit | openEuler Kernel | [内核仓库](https://atomgit.com/openeuler/kernel)、[社区贡献攻略](https://www.openeuler.org/zh/community/contribution/detail.html) | 旧教程里的 Gitee 地址可能已经不是当前协作入口，按当前仓库与 SIG 指引确认分支和流程 |
| kernel.org 与邮件列表 | Linux | [主线代码](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/)、[邮件归档](https://lore.kernel.org/) | 通过维护者和子系统邮件列表提交补丁，GitHub 上的 torvalds/linux 不是普通外部 PR 入口 |

流程依据：[Linux 补丁指南](https://docs.kernel.org/process/submitting-patches.html)、[QEMU 官方贡献入口](https://www.qemu.org/contribute/)、[RT-Thread 贡献指南](https://github.com/RT-Thread/rt-thread/blob/master/.github/CONTRIBUTING.md)。openEuler 的平台变化可查阅[官方迁移通知](https://www.openeuler.org/zh/blog/20251217-openeulerMigrate/20151217-openeulerMigrate)。

选项目时，先从项目官网或官方组织找到仓库，确认是源码上游、镜像还是个人 Fork；平台搜索排名本身不能说明仓库接受贡献。

<a id="内核项目"></a>

## 3. 内核与系统方向：8 个项目

以下难度主要针对“带验证的小型代码贡献”。入门表示任务可以比较局部；中等表示需要理解一个模块；较难表示涉及复杂系统行为、兼容性或较重的验证要求。文档与测试工作也需要真正跑通相关功能。

| 项目 | 定位 | 核心语言 | 建议难度 | 对应求职方向 |
| --- | --- | --- | --- | --- |
| RT-Thread | 嵌入式实时操作系统 | C | 入门—中等 | 嵌入式、驱动、RTOS |
| Zephyr | 嵌入式实时操作系统生态 | C | 入门—中等 | 嵌入式、物联网、系统测试 |
| Apache NuttX | 重视标准接口的实时操作系统 | C | 中等 | 嵌入式系统、POSIX、驱动 |
| ArceOS | Rust 实验性模块化操作系统 | Rust | 中等 | Rust 系统开发、操作系统研究 |
| Asterinas | 面向 Linux 兼容的 Rust 操作系统内核 | Rust | 中等—较难 | 内核、Rust、系统软件 |
| Linux | 通用操作系统内核 | C、少量 Rust/汇编 | 中等—较难 | Linux 内核、驱动、基础软件 |
| openEuler Kernel | 发行版维护的 Linux 内核 | C | 中等—较难 | 服务器操作系统、内核维护 |
| QEMU | 机器模拟与虚拟化基础设施 | C 等 | 中等—较难 | 虚拟化、模拟器、系统软件 |

### K1. RT-Thread：适合从 C 和嵌入式经验起步

入口：[GitHub 仓库](https://github.com/RT-Thread/rt-thread) · [Gitee 仓库](https://gitee.com/rtthread/rt-thread) · [贡献指南](https://github.com/RT-Thread/rt-thread/blob/master/.github/CONTRIBUTING.md) · [Issues](https://github.com/RT-Thread/rt-thread/issues) · [PR 与评审记录](https://github.com/RT-Thread/rt-thread/pulls)

适合会指针、结构体、链表和基本调试，想接触线程、信号量、设备驱动的人。项目提供实时操作系统及其组件生态，官方贡献流程使用 GitHub PR。

- **建议切入：** 环形缓冲区、消息队列或组件接口的边界测试；能够复现的配置、构建问题；已有 BSP 的明确缺陷。BSP 是板级支持包。
- **验证与设备：** 可以先找已有模拟器或 QEMU 目标上的任务；修改真实外设驱动时，需要对应板卡或与持有硬件的维护者合作验证。
- **简历可沉淀：** 线程同步问题的复现方法、缓冲区边界条件分析、驱动错误处理，以及合入上游的测试与修复。
- **第一步：** 跑通一个现有示例与对应测试，选一个自己能复现的问题，再阅读所在模块最近合并的 PR。

### K2. Zephyr：适合学习完整的嵌入式开发与测试流程

入口：[仓库](https://github.com/zephyrproject-rtos/zephyr) · [贡献入口](https://docs.zephyrproject.org/latest/contribute/index.html) · [详细贡献规则](https://docs.zephyrproject.org/latest/contribute/guidelines.html) · [新手任务页](https://github.com/zephyrproject-rtos/zephyr/contribute) · [Issues](https://github.com/zephyrproject-rtos/zephyr/issues)

适合愿意学习 CMake、配置系统、测试框架与设备抽象的人。官方强调小而独立的改动，并要求提供足够的说明和测试。

- **建议切入：** 一个 POSIX 接口的测试、示例的实际运行问题、Kconfig 配置说明、已有模块的小缺陷。
- **验证与设备：** 优先选择支持主机或模拟目标的任务。蓝牙、传感器、真实驱动问题可能依赖开发板及外设。
- **简历可沉淀：** 自动化测试、跨配置验证、接口兼容性、嵌入式构建与调试。
- **第一步：** 看新手任务页，再打开具体 Issue 检查负责人和关联 PR；新手标签不能替代这个检查。

### K3. Apache NuttX：适合对 POSIX 与嵌入式系统接口感兴趣的人

入口：[内核仓库](https://github.com/apache/nuttx) · [应用与测试配套仓库](https://github.com/apache/nuttx-apps) · [贡献指南](https://github.com/apache/nuttx/blob/master/CONTRIBUTING.md) · [官方贡献文档](https://nuttx.apache.org/docs/latest/contributing/index.html) · [Issues](https://github.com/apache/nuttx/issues)

可以围绕系统接口、文件系统和驱动积累经验。官方对兼容性与提交质量要求较细，涉及内核及配套应用时，要检查两个仓库是否需要配合修改。

- **建议切入：** 现有系统接口的错误路径测试、可复现的小型驱动问题、特定配置下的构建修复。
- **验证与设备：** 部分任务可借助模拟目标；官方对破坏兼容性的改动有更严格的真实硬件测试要求，QEMU 不能替代该类验证。
- **简历可沉淀：** POSIX 行为、跨仓库协同、回归测试、嵌入式兼容性分析。
- **第一步：** 先阅读贡献指南与项目原则，选择单一配置、单一功能的问题，避免首次贡献就调整公共接口。

### K4. ArceOS：适合 Rust 操作系统课程之后继续深入

入口：[仓库与构建说明](https://github.com/arceos-org/arceos) · [组织与组件仓库](https://github.com/arceos-org) · [最小应用示例](https://github.com/arceos-org/app-helloworld) · [Issues](https://github.com/arceos-org/arceos/issues) · [PR](https://github.com/arceos-org/arceos/pulls)

项目定位是实验性模块化操作系统。模块化结构适合围绕一个组件学习；它并不意味着整个内核对初学者都简单。部分功能位于独立 crate 仓库，修改前要确认代码实际归属。

- **建议切入：** axlibc 的边界行为与兼容性、分配器测试、同步组件测试、启动示例或构建说明中的实际问题。
- **验证与设备：** 官方最小应用支持多种 QEMU 架构，可先固定一种架构完成复现。
- **简历可沉淀：** Rust 系统编程、C 接口兼容、内存分配、调度与 QEMU 调试。
- **第一步：** 跑通最小应用，找一个范围小的 Issue。第 5 节列出了时间转换函数相关的具体线索。

### K5. Asterinas：适合长期积累 Rust 内核经验

入口：[仓库](https://github.com/asterinas/asterinas) · [项目手册](https://asterinas.github.io/book/index.html) · [贡献入口](https://asterinas.github.io/book/to-contribute/) · [编码规范](https://asterinas.github.io/book/to-contribute/coding-guidelines/index.html) · [Issues](https://github.com/asterinas/asterinas/issues)

Asterinas 以 Rust 和内存安全为核心，目标是构建面向生产的 Linux 替代方案。这里描述的是项目目标，不代表已经具备 Linux 的全部成熟度或兼容性。

- **建议切入：** 一个系统调用的兼容性测试、文件操作错误路径、已有内核组件的回归测试。
- **验证与设备：** 先按手册准备开发环境并跑通测试；所选模块能否在你的宿主架构和虚拟化环境验证，需要具体确认。
- **简历可沉淀：** 系统调用语义、Rust 安全抽象、文件系统、内核测试与问题定位。
- **第一步：** 读手册中的贡献章节和相关模块，再在 Issue 中讨论一个明确的小任务。本次查看的通用新手任务页未列出任务，因此不把它描述成“随时有新手票可领”。

### K6. Linux：适合明确走内核路线、愿意持续投入的人

入口：[主线源码](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/) · [开发流程](https://docs.kernel.org/process/index.html) · [补丁提交指南](https://docs.kernel.org/process/submitting-patches.html) · [KUnit](https://docs.kernel.org/dev-tools/kunit/index.html) · [kselftest](https://docs.kernel.org/dev-tools/kselftest.html) · [邮件归档](https://lore.kernel.org/)

不需要先读完整个 Linux 才能贡献，但必须限定范围，例如一个测试目录、一类接口或一个已有驱动。

- **建议切入：** 为已有行为补 KUnit/selftests；复现并缩小一个回归问题；修复自己能验证的错误处理路径。不要把无上下文的批量格式修改作为主要目标。
- **验证与设备：** 纯逻辑单元测试与部分系统测试可使用虚拟环境；驱动和架构特定问题可能需要真实硬件。测试环境应能启动你修改后的内核。
- **简历可沉淀：** 子系统知识、回归定位、测试设计、邮件评审、上游提交记录。
- **第一步：** 选一个测试目录，运行现有测试；按 `MAINTAINERS` 和补丁指南找到目标维护者、代码树与邮件列表。

写简历时区分“提交到邮件列表”“维护者分支接收”和“进入主线”，三者不是同一状态。

### K7. openEuler Kernel：适合服务器操作系统和内核维护方向

入口：[AtomGit 内核仓库](https://atomgit.com/openeuler/kernel) · [当前贡献攻略](https://www.openeuler.org/zh/community/contribution/detail.html) · [平台迁移通知](https://www.openeuler.org/zh/blog/20251217-openeulerMigrate/20151217-openeulerMigrate) · [历史 Kernel 文档仓库](https://gitee.com/openeuler/kernel-docs)

这是 openEuler 社区维护的 Linux 内核，适合了解发行版如何维护内核分支、集成修复和验证兼容性。历史文档有参考价值，具体账号、平台和分支规则以当前入口为准。

- **建议切入：** 在指定分支复现问题、补充测试、验证已讨论的上游修复是否适用于目标分支。
- **验证与设备：** 使用与目标分支匹配的系统环境；先固定架构和配置，再扩展测试。
- **简历可沉淀：** 发行版内核维护、补丁回移分析、测试矩阵和兼容性验证。
- **第一步：** 通过社区贡献入口找到 Kernel SIG，确认任务和分支。不要把“上游有这个补丁”直接等同于“目标分支应该接收”。

### K8. QEMU：适合虚拟化与系统软件方向

入口：[GitLab 仓库](https://gitlab.com/qemu-project/qemu/) · [官方贡献入口](https://www.qemu.org/contribute/) · [Issues](https://gitlab.com/qemu-project/qemu/-/issues) · [邮件补丁指南](https://qemu-project.gitlab.io/qemu/devel/submitting-a-patch.html)

QEMU 是模拟器与虚拟化基础设施，适合学习设备模型、机器架构与系统软件调试。它不是操作系统内核。

- **建议切入：** 一个现有设备模型的错误路径测试、构建兼容性问题、文档中能够复现的使用缺陷。
- **验证与设备：** 设备模拟任务可以先用软件环境；涉及 KVM 或特定加速功能的任务，要检查宿主系统和硬件支持。
- **简历可沉淀：** 设备模型、虚拟化、系统测试、跨架构行为分析。
- **第一步：** 构建目标模拟器并复现一个小问题；按官方指南发送邮件补丁，不能直接用 GitLab MR 替代。

<a id="视觉项目"></a>

## 4. 计算机视觉与推理方向：8 个项目

“视觉识别”既包括 OCR、分类、检测等应用，也包括训练与识别流程依赖的图像处理、几何变换、数据管线和推理引擎。下表区分这些定位，方便按岗位选择。

| 项目 | 定位 | 核心技术 | 建议难度 | 起步是否需要 GPU |
| --- | --- | --- | --- | --- |
| scikit-image | 图像处理基础库 | Python、NumPy、Cython | 入门—中等 | 通常不需要 |
| Kornia | 可微视觉与几何算子库 | Python、PyTorch | 入门—中等 | CPU 能做部分任务；GPU 问题需 GPU 验证 |
| PaddleOCR | OCR 与文档图像识别 | Python、PaddlePaddle | 入门—中等 | 小样本及部分 CPU 推理任务可起步 |
| Ultralytics | 检测、分割、分类、姿态等 | Python、PyTorch | 中等 | 数据与接口任务可用 CPU；训练与 GPU 后端另计 |
| OpenCV | 视觉算法与工程基础库 | C++、Python 接口 | 中等 | 普通图像处理任务不需要 |
| Torchvision | PyTorch 视觉数据与模型组件 | Python、C++ | 中等 | 数据与部分变换测试可用 CPU |
| ncnn | 移动端与边缘推理引擎 | C++、SIMD、Vulkan | 中等—较难 | CPU 算子任务不需要；目标后端需对应设备 |
| ONNX Runtime | 跨平台推理运行时 | C++、Python 等 | 中等—较难 | CPU 路径可起步；其他后端按设备验证 |

表中的硬件判断是选题建议，不表示可以只测 CPU 就完成所有贡献。没有 GPU 时，应主动选取不依赖 GPU 的问题。

### V1. scikit-image：适合第一次做图像处理库贡献

入口：[仓库](https://github.com/scikit-image/scikit-image) · [官方贡献指南](https://scikit-image.org/docs/stable/development/contribute.html) · [Issues](https://github.com/scikit-image/scikit-image/issues) · [PR](https://github.com/scikit-image/scikit-image/pulls)

适合熟悉 Python、数组操作，愿意理解图像算法输入输出的人。官方贡献文档覆盖测试、示例和性能基准。

- **建议切入：** 空输入、不同数据类型、图像尺寸边界；已有算法示例的实际错误；性能变化的可重复基准。
- **可形成的成果：** 一组说明问题的图像或数组、失败测试、修复与算法解释。
- **第一步：** 选一个自己理解的模块，例如颜色转换、形态学或分割，跑通测试后再挑问题。
- **注意：** 它偏图像处理基础库。面试算法岗位时，应解释贡献如何影响识别前处理或后处理。本次通用新手页没有列出任务，可直接从 Issues 中筛选局部问题。

### V2. Kornia：适合把 PyTorch 与视觉数学结合起来

入口：[仓库](https://github.com/kornia/kornia) · [贡献指南](https://github.com/kornia/kornia/blob/main/CONTRIBUTING.md) · [路线图](https://github.com/kornia/kornia/blob/main/ROADMAP.md) · [Issues](https://github.com/kornia/kornia/issues) · [PR](https://github.com/kornia/kornia/pulls)

项目面向几何视觉和可微视觉操作。适合做张量形状、图像变换、数值精度和梯度方面的贡献。

- **建议切入：** 空批次、不同 dtype/device 的行为差异；几何变换与参考实现的对齐；公开 API 文档与实际接口不一致的问题。
- **可形成的成果：** 数值或梯度正确性测试、与参考实现的误差比较、明确输入范围的性能基准。
- **第一步：** 从你能解释数学含义的单个函数开始。新公共 API 或明显行为变化先讨论设计。
- **注意：** 本次检索发现新手聚合页与具体 Issue 状态可能不一致，具体例子见第 5 节。

### V3. PaddleOCR：适合文字识别与文档处理项目经历

入口：[仓库](https://github.com/PaddlePaddle/PaddleOCR) · [社区贡献说明](https://github.com/PaddlePaddle/PaddleOCR/blob/main/docs/community/community_contribution.md) · [Issues](https://github.com/PaddlePaddle/PaddleOCR/issues) · [PR](https://github.com/PaddlePaddle/PaddleOCR/pulls)

适合从扫描件、图片文字、复杂文档等实际输入出发，追踪识别流程中的工程问题。中文贡献材料便于起步。

- **建议切入：** 图片读取、预处理、结果结构、后处理与推理示例中的可复现问题；公开样本上的模型部署差异。
- **可形成的成果：** 样本集、错误类型分析、修复前后识别表现、推理日志和回归测试。
- **第一步：** 固定版本并跑通一个识别流程，整理少量具有代表性的失败样本，先区分模型能力问题与代码缺陷。
- **注意：** 不同版本目录与 API 可能差异很大；个人应用里的优化不一定适合并入通用工具库。训练或大模型验证的资源需求应单独评估。

### V4. Ultralytics：适合目标检测等应用与工程贡献

入口：[仓库](https://github.com/ultralytics/ultralytics) · [官方贡献指南](https://docs.ultralytics.com/help/contributing) · [Issues](https://github.com/ultralytics/ultralytics/issues) · [PR](https://github.com/ultralytics/ultralytics/pulls)

覆盖检测、分割、分类、姿态和跟踪等视觉任务，适合从完整识别流程中发现数据处理与部署问题。

- **建议切入：** 数据集读取边界、标注转换、结果可视化、导出路径与最小推理示例中的问题。
- **可形成的成果：** 一个可展示的检测项目，加上确实解决通用问题的上游贡献；评估中同时记录准确性和速度。
- **第一步：** 用已有模型和小样本数据复现问题，尽量避免先花大量算力训练、却没有确定缺陷。
- **项目规则：** 官方指南列明 AGPL-3.0 及贡献 CLA 流程，参与前阅读对应文档；项目采用何种许可不要仅凭“YOLO”这个名称判断。

### V5. OpenCV：适合 C++ 视觉工程与基础算法

入口：[主仓库](https://github.com/opencv/opencv) · [扩展模块](https://github.com/opencv/opencv_contrib) · [贡献指南](https://github.com/opencv/opencv/wiki/how_to_contribute) · [Issues](https://github.com/opencv/opencv/issues) · [测试数据仓库](https://github.com/opencv/opencv_extra)

适合愿意从源码构建 C++ 库、理解图像数据布局和算法边界的人。官方指南要求确认问题在受支持开发分支仍然存在。

- **建议切入：** 图像读写、颜色转换、尺寸边界、文档示例的实际错误；熟悉 DNN 后再考虑模型导入与执行问题。
- **可形成的成果：** 精简复现、C++ 回归测试、内存或性能分析、跨平台验证。
- **第一步：** 只构建与目标问题相关的模块和测试，按指南配置测试数据。
- **注意：** 新算法通常先考虑 opencv_contrib，并提前沟通范围；本次新手任务页未列出任务，不能因为项目大就预期有很多容易合并的票。

### V6. Torchvision：适合深入视觉数据与训练管线

入口：[仓库](https://github.com/pytorch/vision) · [贡献指南](https://github.com/pytorch/vision/blob/main/CONTRIBUTING.md) · [Issues](https://github.com/pytorch/vision/issues) · [PR](https://github.com/pytorch/vision/pulls)

适合已经会用 PyTorch、想进一步理解数据集、图像变换和视觉组件的人。

- **建议切入：** 数据集加载异常、图像变换的形状与类型处理、边界条件、文档与实现不一致的问题。
- **可形成的成果：** 可重复的数据管线测试、不同输入形态的一致性验证、训练前处理问题的定位。
- **第一步：** 从你实际使用过的一个 Dataset 或 Transform 开始，阅读现有测试和近期评审。
- **注意：** PyTorch 与 Torchvision 的版本组合会影响复现；新增模型或大型功能先讨论，不能默认项目会接收。

### V7. ncnn：适合 C++、边缘视觉部署与性能优化

入口：[仓库](https://github.com/Tencent/ncnn) · [中文贡献指南](https://github.com/Tencent/ncnn/wiki/how-to-be-a-contributor.zh) · [开发 Wiki](https://github.com/Tencent/ncnn/wiki) · [Issues](https://github.com/Tencent/ncnn/issues) · [PR](https://github.com/Tencent/ncnn/pulls)

项目面向移动端和边缘推理，涉及 CPU 优化、Vulkan、模型转换及量化。对于同时喜欢底层和视觉的人，这是值得重点考虑的交叉方向。

- **建议切入：** 算子测试、模型转换后的输出差异、示例程序问题；先保证正确性，再考虑局部性能优化。
- **可形成的成果：** 模型与算子级误差比较、目标设备延迟和内存基线、优化补丁及回归测试。
- **第一步：** 跑通一个小模型和相关测试，记录硬件、线程数、输入尺寸、精度及构建参数。
- **注意：** x86 测试不能直接证明 ARM 或 Vulkan 后端正确；声称优化了某个后端，就需要对应验证。仓库的 `CONTRIBUTING.md` 主要是致谢信息，实际入门应看上面的中文 Wiki。

### V8. ONNX Runtime：适合推理部署与运行时工程

入口：[仓库](https://github.com/microsoft/onnxruntime) · [贡献指南](https://github.com/microsoft/onnxruntime/blob/main/CONTRIBUTING.md) · [开发 Wiki](https://github.com/microsoft/onnxruntime/wiki) · [Issues](https://github.com/microsoft/onnxruntime/issues) · [标签说明](https://github.com/microsoft/onnxruntime/labels)

这是跨平台推理运行时，视觉模型是其应用场景之一。适合对模型执行、算子、图优化和后端感兴趣的人。

- **建议切入：** 小型 ONNX 模型复现、CPU 算子测试、明确的形状或类型问题、样例修复。
- **可形成的成果：** 最小模型、运行时错误定位、数值对齐测试、推理性能分析。
- **第一步：** 从 CPU 路径和小模型开始，筛选 `contributions welcome` 等标签，并阅读相关模块的评审记录。
- **注意：** 新公共 API 与较大改动需要先讨论；源码构建和多后端验证成本较高。模型导出器与运行时是不同环节，要先定位问题属于哪个仓库。

<a id="任务线索"></a>

## 5. 具体任务线索与筛选结果

这些链接帮助你开始调查，**不是已经认领或保证无人处理的任务**。难度是本文估计，问题描述仍需在当前代码上复现。核验日期为 2026-09-09；检索页面的抓取时间可能早于该日期。

### 值得打开继续确认的线索

| 线索 | 本次查到的状态与核验范围 | 建议起步动作 | 预计技术门槛 |
| --- | --- | --- | --- |
| [ArceOS #357：较大的 tm_year 导致 mktime 溢出 panic](https://github.com/arceos-org/arceos/issues/357) | 具体页显示 Open，未分配负责人，也未显示关联 PR；提供问题说明与复现材料 | 在当前分支复现，分析可表示范围与错误处理，补边界测试后讨论修复 | 中等：Rust、C 时间接口、整数溢出 |
| [ArceOS #324：tm_mon 越界引起 mktime panic](https://github.com/arceos-org/arceos/issues/324) | 具体页显示 Open，包含复现代码；与 #357 属于相近模块 | 先研究月份归一化等接口语义，避免简单拒绝所有范围外输入；确认相关修复是否已覆盖 | 中等：时间接口语义、边界测试 |
| [Kornia #4275：部分公开名称缺失于文档或导出列表](https://github.com/kornia/kornia/issues/4275) | 官方 Issues 列表显示 Open，创建于 2026-09-05；具体页本次抓取失败，负责人和关联 PR 未核实 | 先打开详情确认范围，核对 API 可导入性和文档构建；适合从一个接口开始 | 入门—中等：Python 包结构、文档工具 |
| [Kornia #4281：部分相机几何接口拒绝空批次](https://github.com/kornia/kornia/issues/4281) | 官方 Issues 列表显示 Open，创建于 2026-09-05；具体页本次抓取失败，负责人和关联 PR 未核实 | 确认设计预期，再为形状 B=0 写最小测试，检查输出形状与正常批次行为 | 中等：PyTorch 张量与几何接口 |

Kornia 两条线索的状态依据是[官方 Issues 列表](https://github.com/kornia/kornia/issues)，不是对缺陷真实性的独立验证。ArceOS 两个问题最好作为同一模块的一组候选调查，避免同时做相互重叠的补丁。

### 本次筛掉的“看似可以直接领取”的任务

- [Kornia #2601：移除旧测试标记](https://github.com/kornia/kornia/issues/2601)：曾出现在检索返回的新手聚合页，但打开具体页显示 **Closed**。仅适合参考历史处理方式，不应继续按开放任务认领。
- [Zephyr #20654：补 flash simulator 文档](https://github.com/zephyrproject-rtos/zephyr/issues/20654)：具体页显示 **Open，但已有负责人**，页面还出现相关编号。可以学习或询问是否需要协助，不建议直接重复实现。

这也是本文主要提供持续更新的 Issues 入口，而不是堆积一长串旧“good first issue”编号的原因。

<a id="筛选方法"></a>

## 6. 如何找到值得投入的 Issue

### 6.1 先看最近的合并与评审，再决定投入

在候选仓库的 Pull requests 页面中筛选已合并 PR，重点看与你目标模块相关的 3—5 个小改动：维护者是否仍在评审、测试怎么写、提交描述要求是什么。如果只有自动更新和机器人活动，不能据此判断有人会评审你的代码。

先做一个可独立验证的小问题。持续没有相关维护者响应时，可以保留复现成果，转向备选项目；不要为了等合并而停下所有学习。

### 6.2 GitHub 搜索示例

以下是可以复制到 GitHub Issues 搜索框的查询，仓库名可以自行替换：

```text
repo:kornia/kornia is:issue is:open label:"good first issue" no:assignee
repo:kornia/kornia is:issue is:open label:"help wanted" sort:updated-desc
repo:zephyrproject-rtos/zephyr is:issue is:open label:"Good first issue" no:assignee
repo:microsoft/onnxruntime is:issue is:open label:"contributions welcome"
repo:opencv/opencv is:issue is:open sort:updated-desc
repo:RT-Thread/rt-thread is:pr is:merged sort:updated-desc
```

标签拼写由仓库决定；没有对应标签或没有搜索结果，不代表不能贡献。`no:assignee` 也不保证无人开发，还应检查评论和关联 PR。

### 6.3 优先选择的任务特征

| 值得优先做 | 暂时降低优先级 |
| --- | --- |
| 有输入、期望结果、实际结果和环境信息 | 只有“无法运行”，没有可复现材料 |
| 自己能够运行相关测试 | 依赖自己无法获得的硬件或数据 |
| 范围集中在一个模块或接口 | 跨多个子系统的大型重构 |
| 维护者认可问题或明确欢迎帮助 | 需求尚未形成共识，或已经有人负责 |
| 可以说明修复前失败、修复后通过 | 只能得到“好像快了”“看起来准确了” |

### 6.4 第一次沟通可以这样写

仅在你确实完成对应动作后，替换下面的占位符：

```text
Hi, I reproduced this issue on <commit/version> with <environment>.
The minimal reproducer is <steps or link>.
I would like to work on a focused fix and a regression test.
My initial approach is <brief explanation>.
Is anyone already working on this, and does this approach fit the project?
```

第一次贡献不必追求很大的功能，但应保留你理解并验证代码的过程。各项目的贡献规则、AI 使用与披露要求可能不同，提交时阅读目标仓库当前指南即可。

<a id="执行路线"></a>

## 7. 8 周执行路线

以下按每周约 8—12 小时估计，不包含维护者等待时间。时间较少时延长周期，保留每个阶段的交付物。

| 周次 | 要做的事 | 阶段交付物 |
| --- | --- | --- |
| 第 1 周 | 选 1 个主项目和 1 个备选；读指南、跑示例和相关测试 | 环境说明、版本或 commit、运行日志 |
| 第 2 周 | 阅读一个模块；筛选并复现 1—2 个问题 | 最小复现、预期行为、初步原因分析 |
| 第 3 周 | 在社区确认范围；先实现能够检测缺陷的测试 | 修复前失败的测试与说明 |
| 第 4 周 | 完成局部修复，补验证和必要文档 | 一份可以评审的 PR 或邮件补丁 |
| 第 5—6 周 | 响应评审，完善测试；阅读相邻代码 | 评审修改记录、更新后的补丁 |
| 第 7 周 | 在同一模块做第二项独立改进，或完善性能与兼容性验证 | 更完整的测试矩阵或第二项贡献 |
| 第 8 周 | 整理技术复盘、演示和简历表述 | 成果主页、复现说明、真实状态的简历条目 |

### 内核路线的最小成果包

选择一个方向即可：RT-Thread/Zephyr 的组件测试与修复，或 ArceOS/Asterinas 的接口兼容性工作；已有 Linux 基础则选择一个 selftests/KUnit 范围。

成果包括：固定版本、内核配置或目标板配置、构建方法、复现程序、修复前后日志、补丁与评审链接。Windows 用户可以在适合的 Linux 虚拟机或其他项目支持环境中开发；需要启动自编译内核或使用特定加速功能时，应先确认测试环境支持。

### 视觉路线的最小成果包

选择 Kornia/scikit-image 的一个算子或边界问题，或 PaddleOCR/Ultralytics 的一条实际识别流程。

成果包括：固定依赖版本、少量可公开复现的样本、最小脚本、测试、修复前后结果。算法变化记录准确性指标；性能变化记录输入尺寸、精度、批量大小、线程数、设备、预热方式和测量范围。

### 同时连接两个兴趣的项目选题

**边缘设备上的视觉推理性能分析与上游修复。** 用 ncnn 或 ONNX Runtime 跑一个公开视觉模型，将图像读取、预处理、模型推理、后处理分别计时，查明真正的瓶颈，再决定是修改应用、推理引擎还是系统相关代码。

这个选题可以展示系统分析与视觉部署能力。只有真正修改并向操作系统内核提交的部分，才写成“内核贡献”；普通应用性能优化应按实际工作命名。

<a id="简历成果"></a>

## 8. 如何沉淀为项目经历和简历

### 8.1 把证据整理好

| 成果 | 应保留的证据 | 能说明什么 |
| --- | --- | --- |
| 高质量缺陷报告 | Issue、最小复现、运行环境 | 发现与表达问题的能力 |
| 回归测试 | 修复前失败、修复后通过的日志 | 理解行为并验证修复的能力 |
| 已合并补丁 | 上游 PR/commit 或邮件归档 | 协作与交付能力 |
| 性能优化 | 基线、测量脚本、硬件配置、误差范围 | 实验设计和工程分析能力 |
| 持续贡献 | 同一模块的一组独立成果与复盘 | 对代码的持续理解，而非一次偶然修改 |

只 Fork 仓库、运行官方 Demo，或者在个人项目里调用一个库，可以作为学习或应用开发经历；要写成“向该项目贡献”，应有相应上游记录。尚未合并的补丁可以写“已提交，评审中”，不必隐去真实状态。

### 8.2 简历条目模板

以下均是占位模板，不能把其中的数量、提升幅度或合并状态直接当作已经取得的成绩。

**内核与系统方向：**

> 参与 `<项目/模块>` 开源开发，复现并定位 `<触发条件>` 下的 `<故障>`，通过 `<分析方法>` 确认根因；补充 `<N>` 个回归用例并完成 `<修复内容>`，在 `<架构/配置>` 上验证。补丁状态：`<已合并/评审中>`，附 `<PR/commit/邮件链接>`。

**视觉算法工程方向：**

> 为 `<项目>` 的 `<算子/数据流程>` 修复 `<边界或一致性问题>`，建立覆盖 `<dtype/shape/device>` 的测试，并与 `<参考实现或数学性质>` 对齐；在 `<固定评估条件>` 下取得 `<真实结果>`。附 `<贡献与实验链接>`。

**视觉部署方向：**

> 基于 `<ncnn/ONNX Runtime>` 完成 `<模型>` 的部署与性能分析，定位 `<具体瓶颈>` 并实现 `<改动>`；在 `<设备、输入尺寸、精度、batch>` 下，延迟由 `<A>` 降至 `<B>`，准确性或数值误差为 `<实测结果>`。上游贡献状态：`<真实状态>`。

没有速度提升也可以写有价值的贡献，例如避免崩溃、修正输出、覆盖错误路径或提高兼容性。关键是结果可复现，自己能解释。

### 8.3 面试前准备回答这五个问题

1. 问题在什么输入和环境下出现，影响了谁？
2. 为什么会出现，你排除了哪些可能原因？
3. 为什么选择这个修复，其他方案有什么代价？
4. 如何证明修好了，并且没有破坏原有行为？
5. 维护者提出过什么意见，你如何修改和回应？

<a id="导师计划"></a>

## 9. 导师计划与长期入口

| 入口 | 可以怎样使用 |
| --- | --- |
| [开源之夏 OSPP](https://summer.ospp.ac.cn/) | 查看当期社区、项目与导师要求，搜索操作系统、Rust、视觉或推理相关主题 |
| [Google Summer of Code](https://summerofcode.withgoogle.com/) | 查看当年组织与项目方向，提前通过小型贡献了解社区 |
| [LFX Mentorship](https://mentorship.lfx.linuxfoundation.org/) | 查询 Linux 基金会相关导师项目及其技能要求 |

这些是持续关注入口，不代表当前仍可报名，也不代表本文每个项目都参加本年度计划；资格、项目列表与时间以当期官方页面为准。无需等活动开始才做日常贡献。

<a id="记录模板"></a>

## 10. 可复制的贡献记录模板

```markdown
### 贡献：<项目名 / 问题标题>

- 仓库：
- Issue：
- PR / 邮件补丁：
- 当前状态：调查中 / 开发中 / 已提交 / 评审中 / 已合并
- 上游 commit 或接收分支：
- 我的工作范围：
- 开始日期与投入时间：

#### 环境

- 操作系统、CPU / GPU / 开发板：
- 项目版本或 commit：
- 编译器、依赖版本、构建参数：

#### 问题与修复

- 最小复现：
- 预期行为与实际行为：
- 根因：
- 修复方案与取舍：
- 修复前失败的测试：
- 修复后验证结果：
- 尚未验证的范围：

#### 协作与成果

- 维护者反馈：
- 根据反馈做出的修改：
- 实验或演示链接：
- 可写入简历的一句话：
```

**今天可以完成的第一步：** 从路线表选择一个项目，阅读贡献指南，跑通一个示例和一组相关测试，然后记录版本、环境和结果。之后再从第 5 节的线索或项目 Issues 中选题。
