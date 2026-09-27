# 从向量加法理解 GPU 算子：TileLang、NineToothed 与性能验证

> 前置文档：[`metax_c500_env_setup.md`](./metax_c500_env_setup.md) 讲环境和软件栈；[`metax_c500_workflow.md`](./metax_c500_workflow.md) 讲开机、连接和作业流程。这篇只回答一个问题：**一个可信的 GPU 算子，是怎样从数学定义变成可验证的并行程序的？**

## 结论（TL;DR）

1. 向量加法的难点不是 `A + B`，而是把任意长度 `N` 完整、不越界地映射到 GPU 并行程序。
2. TileLang 暴露的是“块数→块号→局部偏移→全局索引”；NineToothed 则把它拆成“怎么分块”和“块内算什么”。
3. `ceildiv` 和 `index < N` 是一对：前者保证尾块被启动，后者保证尾块不越界。少任何一个都不是正确实现。
4. 性能数字只有在**先校验正确性、再预热、同口径多次计时**后才有意义；一个耗时不能自动等价为“显存带宽”。
5. 本次实测中，`BLOCK_N=1024` 在 1 Mi 元素上比 `256` 快约 `1.53×`，但小规模下差距很小；优化参数不存在脱离 shape 的永久最优值。

## 一、先建立一张全局图

本项目的完整闭环不是“写出一段代码”，而是：

```text
硬件/驱动可见
    ↓
框架能在 GPU 创建张量
    ↓
算子契约（shape、dtype、公式、边界）
    ↓
并行映射（block、pid、index、tail mask）
    ↓
参考实现对拍
    ↓
预热 + 多次计时 + 解释限制
    ↓
代码、数据、截图、结论一起进 Git
```

其中任何一层缺失，结论都可能是假的：

| 缺失的环节 | 常见假象 |
| --- | --- |
| 只看 `mx-smi` | 认为看到物理卡 64 GiB 就拥有 64 GiB 配额 |
| 只看程序“能跑” | 实际落到 PyTorch fallback，自己的 kernel 没被调用 |
| 只测正数 shape | 尾块越界在测试中永远不出现 |
| 只跑一次计时 | 把 JIT 编译、冷启动和抖动当成 kernel 耗时 |
| 只比一个 shape | 把某个特定规模的结果误写成普遍结论 |

## 二、环境的核心不是版本号，而是调用链

实测环境：沐曦 C500 切片卡（50% 算力、32000 MiB 显存配额），PyTorch `2.8.0+metax3.5.3.9`，Triton `3.0.0+metax3.5.3.9`，NineToothed `0.26.0`，TileLang 源码树在 `/app/tilelang-metax`。

真正应该验证的是下面这条链：

```text
Python 程序
  → PyTorch / TileLang / NineToothed
  → Triton 或 TileLang JIT
  → MACA 后端
  → MetaX 驱动
  → C500 / sGPU 配额
```

两个容易误判的点：

- MACA 版 PyTorch 仍使用 `device="cuda"`。这是兼容接口，不代表程序跑到了 NVIDIA GPU。
- `mx-smi` 显示物理卡总显存，`torch.cuda.mem_get_info()` 更接近当前进程真正受到的配额。看“谁在受限”，不看哪张表更显眼。

## 三、向量加法的算子契约

数学定义只有一行：

```text
C[i] = A[i] + B[i],  0 ≤ i < N
```

可执行的算子契约还必须写清：

| 维度 | 本项目的定义 |
| --- | --- |
| shape | `A`/`B`/`C` 均为 1-D，长度 `N` |
| dtype | `float16` |
| 输出 | 新建 `C`，不原地修改输入 |
| 边界 | `1 ≤ N ≤ 1M`，必须支持 `N % BLOCK_N != 0` |
| 参考实现 | `torch.add(A, B)` |
| 误差 | `atol=1e-2, rtol=1e-2` |

“尾块”不是额外功能，而是契约的一部分。只会处理 `N` 刚好整除块大小的 kernel，只能算 demo。

## 四、TileLang：显式写出索引映射

核心实现：

```python
num_blocks = T.ceildiv(N, BLOCK_N)

with T.Kernel(num_blocks, threads=256) as pid:
    base_idx = pid * BLOCK_N

    for offset in T.Parallel(BLOCK_N):
        index = base_idx + offset
        if index < N:
            C[index] = A[index] + B[index]
```

### 1. 四层索引关系

```text
num_blocks = ceil(N / BLOCK_N)    需要几个数据块
pid                               当前是第几个块
base_idx = pid * BLOCK_N          这个块从哪个元素开始
index = base_idx + offset         当前局部位置对应的全局元素
```

例如 `N=100003, BLOCK_N=1024`：

- `ceildiv(100003, 1024) = 98`，所以必须启动 98 个块；
- 前 97 块处理 99328 个元素；
- 最后一块只有 675 个有效元素；
- 因此最后一块必须执行 `index < N`。

### 2. `BLOCK_N` 不等于 `threads`

- `BLOCK_N`：一个程序块要覆盖多少数据元素。
- `threads=256`：一个 GPU 线程块使用多少执行线程。
- `T.Parallel(BLOCK_N)`：由编译器把 `BLOCK_N` 个独立迭代映射到线程上。

因此 `BLOCK_N=1024, threads=256` 不是“用 256 个线程只算 256 个元素”。平均看，每个线程可以承担多个独立元素，具体映射由编译器决定。

### 3. 为什么必须连续访存

相邻执行单元访问相邻地址，硬件才能合并显存事务。`index = base_idx + offset` 是连续访存；如果改成大 stride 跳着读，有效带宽通常会下降。

这是向量加法的性能本质：**计算只有一次加法，更多时间花在读两个输入、写一个输出。**

## 五、NineToothed：把布局和计算分开

NineToothed 实现同一个算子时，把两类问题拆开：

```python
def arrangement(lhs, rhs, output):
    return (
        lhs.tile((BLOCK_SIZE,)),
        rhs.tile((BLOCK_SIZE,)),
        output.tile((BLOCK_SIZE,)),
    )

def application(lhs, rhs, output):
    output = lhs + rhs
```

| 函数 | 回答的问题 |
| --- | --- |
| `arrangement()` | 输入和输出如何按 `BLOCK_SIZE` 分 tile |
| `application()` | 每个 tile 内部做什么计算 |

这种抽象让算子表达很短，但不代表底层没有分块、索引和尾块处理；只是 DSL 帮开发者承担了部分工作。

### import 期 trace 是最容易看错的报错

```python
_KERNEL = ninetoothed.make(arrangement, application, ...)
```

这句位于模块顶层，所以 `import ninetoothed_add` 时就会 trace `arrangement()`。如果 `arrangement()` 仍是 `NotImplementedError`，pytest 表现为 **collection error**，而不是普通的 test failure。

可复用判断：**错误发生在测试收集阶段时，先查 import 期执行的 trace / compile / registry，不要先怀疑测试数据。**

## 六、正确性：测试的价值在于覆盖失败方式

TileLang 测试了 `N=1、127、1024、100003、1048576`：

- `1`：最小规模，暴露空迭代和启动配置问题；
- `127`：小于块大小，整个 kernel 几乎都是尾块；
- `1024`：刚好整除，验证常规路径；
- `100003`：非 2 的幂且不整除，专门攻击边界逻辑；
- `1048576`：大规模，同时覆盖性能路径。

NineToothed 的 `98432 = 96 × 1024 + 128`，最后一块只有 128 个有效元素，因此也是一个尾块测试，不是随机选的数。

校验模式：

```python
reference = torch.add(lhs, rhs)
torch.testing.assert_close(output, reference, atol=1e-2, rtol=1e-2)
```

`float16` 不适合直接用 `==`。绝对误差处理接近 0 的值，相对误差处理数值尺度不同的值，两者要同时写清。

## 七、性能测量：先把“测到了什么”说清楚

### 1. 为什么需要预热和同步

GPU 调用通常是异步的：CPU 提交任务后可能立即返回，GPU 还在工作。如果直接用 CPU 时钟夹住一次调用，容易只测到提交耗时。

有效测量至少需要：

1. 先执行多次预热，排除 JIT 编译和冷缓存；
2. 使用 GPU Event，或在 CPU 计时边界调用 `torch.cuda.synchronize()`；
3. 重复多次，使用平均值或中位数；
4. 对比项使用相同的预热、重复次数和输出分配口径。

`triton.testing.do_bench(lambda: ...)` 帮忙做了预热和多次测量，但本项目的 `nt_add_1d()` 每次还会执行 `torch.empty_like()`，所以这个结果包含输出分配，不是纯 kernel 时间。

### 2. 向量加法的有效带宽

`float16` 每个元素 2 字节。理想化地看，每个元素需要：

```text
读 A：2 B + 读 B：2 B + 写 C：2 B = 6 B
```

因此：

```text
effective_GB/s = (6 × N) / (time_ms × 10^6)
```

它是**按算子语义折算的有效吞吐**，不等于物理 HBM 瞬时带宽。缓存命中、输出分配、启动开销、计时方法和数据重用都会影响数字。

### 3. 本次实测能证明什么

TileLang，`N=1048576`：

| `BLOCK_N` | 耗时 | 有效带宽 |
| ---: | ---: | ---: |
| 256 | 19.39 μs | 324.45 GB/s |
| 1024 | 12.68 μs | 496.19 GB/s |

在这个 shape 上，`1024` 耗时是 `256` 的约 `65.4%`，即快约 `1.53×`。但在 `N=1、127、1024` 时，两者都约 13 μs，因为固定启动开销盖过了分块差异。

NineToothed，`N=134217728`：

| 配置 | NineToothed | PyTorch |
| --- | ---: | ---: |
| 固定 `BLOCK_SIZE=1024` | 0.7820 ms | 0.5620 ms |
| 开启自动调优 | 0.7676 ms | 0.5619 ms |

自动调优在最大规模上提升约 `1.8%`，但整体仍略慢于 PyTorch。这个结论完全合格：作业要求是正确实现和同口径对比，不是必须打赢成熟库。

### 4. 本次实测不能证明什么

- 不能仅凭 496.19 GB/s 宣布已经到达 C500 物理显存带宽；
- 不能凭一次 `1024 > 256` 宣布 1024 对所有 shape 都最优；
- 不能把跨重启、跨物理卡的微小差异当成优化效果；
- 不能在正确性校验之前谈性能。算错了的 kernel 再快也没有意义。

## 八、TileLang 与 NineToothed 的抽象差异

| 维度 | TileLang | NineToothed |
| --- | --- | --- |
| 核心表达 | 显式 kernel 和索引映射 | arrangement + application |
| 分块 | `T.ceildiv` + `T.Kernel` | `Tensor.tile()` |
| 块内并行 | `T.Parallel` | DSL 从 tile 表达推导 |
| 尾块 | 显式 `index < N` | DSL 负责边界处理 |
| 错误时机 | `@tilelang.jit` 通常在调用/编译时暴露 | `ninetoothed.make()` 可在 import 期暴露 |
| 适合学习 | 线程、块、索引、mask | 数据布局与计算解耦 |

两者不是“哪个语法更短”的比较。TileLang 训练的是如何显式控制并行映射；NineToothed 训练的是如何把布局与计算解耦。理解两种抽象，比背两段代码更有价值。

## 九、报错时的最短路径

| 现象 | 先分类 | 最先检查 |
| --- | --- | --- |
| `import tilelang` 失败 | 环境问题 | `$TILELANG_ROOT`、`PYTHONPATH`、`libtilelang.so` |
| pytest 报 `NotImplementedError` | 作业尚未实现 | 报错行是否仍是 TODO |
| NineToothed collection error | import 期 trace | `arrangement()` 是否已完整返回三个 tiled tensor |
| 只在非整除 shape 失败 | 尾块问题 | `ceildiv` 和 `index < N` 是否同时存在 |
| 结果错但不崩溃 | 计算/映射问题 | 先打印小 shape，再与 PyTorch 逐元素对拍 |
| 首次运行特别慢 | JIT 编译 | 预热后重测，不要用首次数据 |
| 小 shape 下 GB/s 接近 0 | 启动开销主导 | 扩大 shape，不要怀疑公式 |
| 性能比较波动 | 测量口径/环境 | 同步、预热、重复次数、物理卡是否变化 |

排错顺序应该是：

```text
先分层（环境 / import / 编译 / 正确性 / 性能）
→ 做最小复现
→ 与参考实现对拍
→ 一次只改一个变量
→ 记录变更前后数据
```

## 十、Git 不只是上传工具，而是实验记录系统

这个项目采用两个远程：

```text
upstream = 官方 GitLink 仓库（只取课程更新）
origin   = 个人 GitHub 仓库（提交自己的实现和证据）
```

本次完整的提交单元包含：

- 算子源码；
- benchmark 脚本；
- 原始文本结果；
- 终端截图；
- 3～5 句的结论与限制。

它们合在一起才能回答“代码是什么、有没有测过、数据是什么、作者如何解释”。

Git 四个阶段：

```text
修改文件 → git add → git commit → git push
工作区      暂存区      本地历史       GitHub
```

PAT 只是 HTTPS 推送时的密码替代品。它必须使用最小权限（指定仓库 + Contents read/write），不得进入聊天、截图、shell 历史或仓库。已暴露的 Token 没有“藏回去”，只有立即删除/撤销才是有效处置。

## 十一、从向量加法迁移到后续算子

向量加法已经建立了四个基础：分块、索引、尾块、基准测量。后续算子只是逐步加入新问题：

| 算子 | 在向量加法之上新增的难点 |
| --- | --- |
| Softmax | 行级归约、数值稳定、`max`/`sum` 两次 reduction |
| GEMM | M/N/K 三维分块、数据复用、float32 累加 |
| LayerNorm | 均值/方差归约、精度与多路读取 |
| RMSNorm | reduction + 逐元素缩放，以及框架注册/fallback 验证 |
| RoPE | 索引变换、两路旋转、布局和向量化 |

下一阶段不应继续机械扩大向量长度，而应学会：

1. 在写代码前预测瓶颈是计算、带宽、启动还是同步；
2. 用 profiler 验证预测，而不是仅凭耗时猜测；
3. 引入 shared memory / local buffer 后，计算数据复用带来的收益和同步成本；
4. 每次只改一个参数，保留可复现的对照组。

## 一句话方法论

> **GPU 算子开发不是把公式翻译成 DSL，而是把数据、并行映射、边界、测量口径和证据一起变成可验证的系统。**
