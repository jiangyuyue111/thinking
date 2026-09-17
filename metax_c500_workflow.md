# 沐曦 C500 算子开发实战营 —— 开机、任务与知识点

> 姊妹篇：`metax_c500_env_setup.md`（讲**环境怎么搭**、以及怎么识别一份 AI 编的假搭建指南）
> 这篇讲**搭好之后怎么用**：每次开机做什么、四份作业要交什么、这轮新踩的坑。
> 更新时间：2026-09-18

## 结论（TL;DR）

1. **开机就两步**：控制台开机 → `ssh metax`。环境变量**自动就位**，不用再 `source`。
2. **只有 `/data` 和容器可写层扛重启**。「关机/开机」IP 不变；「销毁重建」会换 IP、端口、用户名。
3. **免密登录本平台做不到**——服务端的 SSH 网关是 `SSHPiper`，只开密码认证。**别再试公钥了。**
4. 四份作业按「环境认知 → 基础算子 → 复杂算子 → AI 协作」递进，**10/18 截止**。

---

## 一、每次开机：三步

### ① 控制台开机

模力方舟 → 算力市场 → 找到你租的那台 C500 → **开机**。

> ⚠️ **只关机，别销毁。** 关机再开，IP/端口/用户名都不变；销毁后重新租，三者全变，
> 本地 ssh 配置得改一次。`/data` 是挂载卷能活下来，但没必要找这个麻烦。

### ② 本地连接

本地 `~/.ssh/config` 里配好别名后，连接就是三个词：

```sshconfig
Host metax
    HostName <实例IP>          # 销毁重建后会变，改这里
    Port <端口>                 # 同上
    User root+vm-xxxxxxxx       # 同上。这串是网关的**路由键**，不是普通用户名
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

```powershell
ssh metax
```

**会问密码**，输进去。（为什么不能免密 → 见第三节第 1 条）

### ③ 直接开始工作

进去会先打印一段 banner：

```
TileLang environment configured:
  source: /app/tilelang-metax
  native: /app/tilelang-metax/build/lib/libtilelang.so
  python: /opt/conda/bin/python
```

**看到这个就说明环境全自动就位了**，直接 `cd` 到作业目录跑测试：

```bash
cd /data/gollamago/assignment/task2
python -m pytest -q test_add.py
```

**不用再 `source ./setup_env.sh`。** 快速自检：

```bash
echo $TILELANG_ROOT      # 应打 /app/tilelang-metax
```

### 传送文件

```powershell
scp C:\path\to\file metax:/data/
```

有了 config 别名，**不用再写 `-P 端口`**。

---

## 二、我要做什么：四份作业

递进关系：环境认知 → 基础算子 → 复杂算子 → AI 协作。

| 任务 | 主题 | 交付物 | 截止 |
| :---: | :--- | :--- | :---: |
| 1 | 沐曦算力与 MXMACA 入门 | 环境报告（两张截图） | — |
| 2 | TileLang Add + 九齿 Vector Add | 两个 kernel + benchmark 截图 + 3~5 句记录 | — |
| 3 | TileLang Softmax + 九齿 GEMM | 两个 kernel + benchmark 截图 | — |
| 4 | AI Agent 辅助开发 | `prompts.md` + `reflection.md` | — |
| — | **结业项目** | 任选 2 种后端实现 ≥2 个算子 | **10/18** |

### 任务一：环境认知（最简单，先交这个）

```bash
mx-smi                                        # 截图 1：硬件识别
pip list | grep -iE "torch|maca|metax"        # 截图 2：软件栈识别
```

要会的东西：`mx-smi`（对标 `nvidia-smi`）的常用参数——
`mx-smi`（概览）、`-l 1000`（1 秒刷新）、`--show-usage/-memory/-temperature/-process`。

### 任务二：基础算子

**TileLang Add**——`solution.py` 里的 `tl_add_1d(A, B, BLOCK_N)`：
1D `float16` 加法，`1 ≤ N ≤ 1M`，**必须处理尾块**（N 不整除 `BLOCK_N`）。
题目给了 5 步提示：`T.ceildiv` 算块数 → `T.Kernel` 取 pid 算 base_idx → `T.Parallel`
遍历 tile → 判断越界保护尾块 → 写回。

```bash
cd /data/gollamago/assignment/task2
python -m pytest -q test_add.py          # N=1、127、1024、100003、1048576
python benchmark_add.py                  # 交这个的截图（对比 ≥2 种 BLOCK_N）
```

**九齿 Add**——`ninetoothed_add.py` 里的 `arrangement()` + `application()`，默认 `BLOCK_SIZE = 1024`。

```bash
python -m pytest -q test_ninetoothed_add.py    # size=98432
python benchmark_ninetoothed_add.py            # 2^18 ~ 2^27
```

> ⚠️ **先写 `arrangement()` 再写 `application()`。** 见第三节第 6 条。

### 任务三：复杂算子

**TileLang Softmax**——`tl_softmax(A, BLOCK_N, BLOCK_M)`：二维 `[N,M] float32` 按行 Softmax，
用**数值稳定**公式 `exp(x-max)/sum(exp(x-max))`。benchmark 五组配置：`(1,1)、(3,7)、(16,256)、(17,513)、(64,4096)`。

**九齿 GEMM**——`ninetoothed_gemm.py`：`C = A @ B`，输入输出 `float16`、**中间 `float32` 累加**，
默认 block `64×64×64`，用 `ntl.dot` 沿 K 方向累加。测试 `M=N=K=512`。

```bash
cd /data/gollamago/assignment/task3
python -m pytest -q test_softmax.py test_ninetoothed_gemm.py
python benchmark_softmax.py
python benchmark_ninetoothed_gemm.py
```

### 任务四：AI Agent 辅助

**不是"让 AI 帮我写"，是"记录 AI 怎么参与的"。** 四个阶段必须都有：

1. **需求分析**——给 Agent 算子公式、shape/dtype、TileLang 版本，要它列算法步骤、边界条件、风险
2. **代码/测试生成**——生成 kernel 初稿或测试用例
3. **验证与调试**——跑 pytest/benchmark，失败日志发回 Agent 要修复方案
4. **性能优化**——要它基于**实测数据**提 ≥2 个优化假设，**逐一实测**并记录是否有效

每轮 Prompt 必须含：**算子契约 + 目标硬件 + 当前代码/错误 + 验收标准**。
禁止只写"帮我优化"。从 `prompts.md` 模板开始，最后交 `prompts.md` + `reflection.md`。

### 结业项目（10/18）

在 gollamago 里用 **九齿 / TileLang / MACA C++ 任选 2 种**，实现并优化 **≥2 个算子**。
框架已预置 `rms_norm` 和 `rope` 的调用/注册/测试入口。

> ⚠️ 没实现的槽位会**自动 fallback 到 PyTorch 并打警告**——所以"能跑通"不代表你接上了。
> 检查 `registered_operators` 里值为 `torch_fallback` 的，那些就是还没接的。
> **这种状态不能拿来做性能结论。**

---

## 三、知识点：这轮真正值钱的几条

### 1. 免密登录为什么做不到——`SSHPiper` 网关

**现象**：公钥装进了实例的 `~/.ssh/authorized_keys`，实例内 `sshd -T` 也显示
`pubkeyauthentication yes`，但客户端永远 `Permission denied`。

**根因**：服务端 banner 是 **`SSH-2.0-SSHPiper`** —— 根本**不是 OpenSSH**，而是
[sshpiper](https://github.com/tg123/sshpiper)，一个**按用户名路由的 SSH 反向代理**（docker 术语里的
downstream = 客户端，upstream = 真服务器）。平台拿它当网关。

一处一处对上：

| 观察 | 解释 |
|---|---|
| 用户名 `root+vm-3S9lrtEHcRqkOMov` 这种怪格式 | sshpiper **按用户名路由**。`root` = 目标账户，`vm-xxx` = 路由到哪台 |
| `Authentications that can continue: password` | 网关的**插件配置**没开公钥 |
| 实例里 `sshd -T` 说 pubkey 开着 | 那是**内层** sshd。网关在前，内层配置**管不到客户端** |
| 公钥塞进实例 `authorized_keys` 没用 | sshpiper 要用自己的 **mapping key** 重签 session_id。实例里那份是给**它的** upstream 连接用的 |

> **准确表述**：不是"sshpiper 不支持公钥"（它支持，能在上下游间做认证方式映射），
> 而是**平台没为这个账号开，而开它要改网关侧的 mapping key，实例内部够不着。**

> **可复用教训**：**查认证配置，要查"正在服务那个端口的东西"，不是查默认配置文件。**
> `sshd -T` 不带参数读的是 `/etc/ssh/sshd_config`，而 32222 端口根本不是那个 sshd 在服务。
> 通用形式：**先确认"谁在服务这个端口"（`ss -tlnp` / banner / 进程命令行），再查它的配置。**
> 推而广之，`nginx -T`、`git config --list --show-origin` 都是同一个思路：
> **查最终生效值，别查"我写了什么"。**

### 2. `pkill -HUP -x sshd` 会把自己踢下线

**现象**：改完 sshd 配置跑重载，命令刚执行就被 `Connection closed by remote host` 断开。

**根因**：`-x` 是**进程名完全匹配**，而**每个会话的子进程进程名也是 `sshd`**
（argv[0] 形如 `sshd: root@pts/0`）。所以这一下把所有 sshd 全发了 HUP，包括正在服务你这条连接的。

- 给 **master** 发 HUP → 重载配置，现有连接不受影响 ✅
- 给**会话子进程**发 HUP → **它直接退出** ❌

**正确写法**：

```bash
kill -HUP "$(cat /var/run/sshd.pid)"     # 首选，认 pidfile
pkill -HUP -o -x sshd                    # 兜底，-o = 只选最早启动的（master 起得最早）
```

> **可复用教训**：**给守护进程发信号前，先分清"哪个是 master"。**
> `pkill -x <name>` 在"一个 master + N 个同名 worker/会话进程"的模型下必然误伤。
> 这类进程（sshd、nginx worker、gunicorn、supervisor）都有同样的坑。
> 顺带记住：**改远程服务的配置，先另开一个连接放着当保险**——恢复成本最低。

### 3. 登录自动 source 是怎么打通的（login shell 链）

**目标**：不用每次登录手动 `source ./setup_env.sh`。

**做法**：往 `~/.bashrc` **末尾**追加：

```bash
[ -f /data/gollamago/setup_env.sh ] && source /data/gollamago/setup_env.sh
```

**为什么这样就够**——`ssh` 起的是 **login shell**，读文件的顺序是：

```
~/.bash_profile  →  ~/.bash_login  →  ~/.profile     （取第一个存在的）
```

**不是直接读 `.bashrc`。** Ubuntu 默认靠 `~/.profile` 里的 `if [ -n "$BASH_VERSION" ]` 那一段
转手去 source `.bashrc`。**如果镜像里有 `.bash_profile` 但没写转手代码，链就断了。**
本机只有 `~/.profile`（887 字节，Ubuntu 默认版），所以链是通的。

**为什么必须放末尾**——`.bashrc` 开头有：

```bash
case $- in
    *i*) ;;
      *) return;;
esac
```

**非交互 shell 会提前 return**。放末尾 = 只对交互式登录生效，**不会污染 `scp`/`sftp` 的会话**。
放开头则可能让 `scp` 也去执行 `setup_env.sh`，拖慢甚至搞坏非交互传输。

**验证方式**（不用真的重连，忠实模拟一次登录）：

```bash
bash -lic 'echo TILELANG_ROOT=$TILELANG_ROOT'
# -l = login shell，-i = interactive，跟 sshd 起 shell 的方式一致
```

> **可复用教训**：**改 shell 启动配置前，先搞清楚"谁会读这个文件"。**
> login shell ≠ interactive shell ≠ non-interactive shell，三者读的文件不同。
> 放错位置的代价是"改了但没生效"或者"污染了 scp"——**两种都很难查**。

### 4. `@triton.jit` 不能从 heredoc / stdin 里跑

**现象**：`python - <<'PY'` 喂一段带 `@triton.jit` 的代码进去，报
`OSError: could not get source code`。

**根因**：`@triton.jit` **在装饰时**就调 `inspect.getsourcelines(fn)`
（`triton/runtime/jit.py:704`）。heredoc / 管道 / `exec` 进来的代码**没有源文件**，自然取不到。

**正确做法**：先写文件再跑。

```bash
cat > /tmp/triton_smoke.py <<'PY'
...
PY
python /tmp/triton_smoke.py
```

> **可复用教训**：**依赖 `inspect.getsource` 的东西都不能从 stdin 喂。**
> 除了 triton，还有 `torch.compile`、`dataclass`、`pydantic`、`functools.wraps` 相关的调试路径。
> 这是个**测试写法的坑，不是环境问题**——别看到报错就去怀疑环境。

同时记住**怎么验证 triton 认的是 MACA 后端**：

```python
print(triton.runtime.driver.active.get_current_target())
# GPUTarget(backend='maca', arch=80, warp_size=64)   ← backend 必须是 'maca'
```

报成 `cuda`/`hip` 才是真出问题。⚠️ 但**别用 `triton.__version__` 判断是不是 MACA 版**：
它读包内 `version.py` 的硬编码串（输出 `3.0.0`），`+metax` 后缀只在 wheel 的 dist-info 元数据里
（`pip show triton` 才有）。两者都对，不是装坏了。

### 5. 判断"是不是切片卡"看 torch，不看 `mx-smi`

```
mx-smi:  Memory-Usage: 826/65536 MiB     ← 物理卡容量（64GB）
torch:   mem free/total = 30.5/30.8 GiB  ← 你的配额（32GB）
```

**两张表说的是两回事。** 更要命的是：**`mx-smi` 的 `Sliced GPU` 表会变空**（重启后就空了），
但配额**照旧生效**（torch 还是只看得见 30.8 GiB）。

> **教训**：**别用"某张表有没有渲染出来"判断配额**。看真正受约束的那一方（torch 报的数）。

顺带两条：
- **镜像标签骗人**：标签写 `maca 3.3.0.4`，实跑 `MACA 3.5.3.20`，所有包后缀 `+metax3.5.3.9`。
  **以 `mx-smi` / `pip show` 为准，别信镜像名。**
- **性能数据别跨重启对比**：重启会换物理卡（Bus-id `38:00.0` → `0e:00.0`）。

### 6. 九齿的 `ninetoothed.make()` 在 import 时就 trace

`ninetoothed_add.py` 里 `_KERNEL = ninetoothed.make(arrangement, application, ...)` 是**模块级**的，
所以 **import 时就会 trace `arrangement()`**。

**后果**：`arrangement()` 没实现 → 报 **collection error**（不是 test failure），
**你连 `application()` 的报错都看不到**。

> **顺序：先写完 `arrangement()`，再写 `application()`。**
> 对比 TileLang：`@tilelang.jit` 是**惰性**的，调用时才报，没这个问题。
> **通用教训**：**模块级的元编程（make/trace/compile）会把错误提前到 import 期**，
> 让"缺一个函数"表现成"整个文件收集失败"，报错位置和真实原因会脱节。

### 7. 装包前必须 dry-run

`pip install -r requirements.txt` 里的 `torch` **没有版本号**。正常情况下 pip 认为已装的
`+metax` 版满足要求、不会动它——**但这是 pip 的实现细节，不是承诺**。

一旦被换成 PyPI 原版 `+cu121`，MACA 就废了，**且不可逆**（只能重建实例）。

```bash
# 装之前
python -m pip install --dry-run ninetoothed   # 确认 triton 那行是 "already satisfied"
pip show torch | grep -i ^Version             # 记下来
# 装之后
pip show torch | grep -i ^Version             # 对比，后缀必须还是 +metax
```

本次实测：九齿只要求松约束 `triton>=3.0.0`，MACA 版 triton（`3.0.0+metax3.5.3.9`）满足，
**没有被顶掉**。

> **通用原则**：**对"做坏了只能重来"的操作，装前装后各记一次状态。**
> 代价两行代码，收益是一个不会漏掉的告警。

---

## 四、出问题怎么查

| 症状 | 先查什么 |
|---|---|
| `ssh metax` 连不上 | 控制台看实例是否在运行；IP/端口是否变了（销毁重建会变） |
| 登录后 `$TILELANG_ROOT` 是空的 | `tail -4 ~/.bashrc` 看那三行还在不在；在就 `source /data/gollamago/setup_env.sh` |
| `import tilelang` 失败 | 必须先 source（现在自动了）；`ls /app/tilelang-metax/build/lib/libtilelang.so` |
| `import torch` 报错 / 后缀不是 `+metax` | 环境被污染了，看第四节第 7 条 |
| 作业 pytest 全红且是 `NotImplementedError` | **正常**，那是要你填的 TODO |
| 九齿测试 collection error | 多半是 `arrangement()` 还没写（第三节第 6 条） |
| 结果对不上但没报错 | 检查是不是 fallback 到 PyTorch 了（看 `registered_operators`） |

**一键体检**：`bash /data/restore.sh`（幂等，可反复跑）。

---

## 五、环境信息

| 项 | 值 |
|---|---|
| 课程 | 沐曦「多范式算子开发实战营」（9/13 开营，10/18 项目截止） |
| 仓库 | <https://www.gitlink.org.cn/ccf-ai-infra/gollamago> |
| 镜像 | 模力方舟 → 预装镜像 → **基础镜像** → `TileLang / 0.1.9 / Python 3.12 / maca 3.3.0.4` |
| 实际运行时 | **MACA 3.5.3.20**（镜像标签写的 3.3.0.4 是旧的）、驱动 3.8.30、`mx-smi` 2.2.12 |
| 硬件 | 曦云 C500 **切片卡**：50% 算力 + 32GB 配额（torch 报 30.8 GiB） |
| 软件栈 | `torch 2.8.0+metax3.5.3.9`、`triton 3.0.0+metax3.5.3.9`、`ninetoothed 0.26.0`、`tilelang 0.1.9+cuda.gitf1ca0fb9` |
| 官方工作目录 | `/data/gollamago` |
| TileLang 源码树 | `/app/tilelang-metax`（靠 `PYTHONPATH` 挂载，不是 pip wheel） |
| 四个后端 | `torch`(基线) / `tilelang` / `ninetoothed` / `maca_cpp` |
| target | `auto` / `cuda` / `maca`（**MACA 下 `--device` 仍写 `cuda`**，是刻意兼容不是笔误） |
| SSH 网关 | `SSHPiper`，**只支持密码登录** |

### 选镜像的一个坑

**选镜像前先把 GPU 型号选对。** 型号选错会把列表筛成**另一套体系**：
`Flagos / 2.1 / PyTorch-Agent / {vllm:0.24.0 | megatron 0.17.1 | verl 0.7.0} / Python3.12 / maca3.7.0.38`。
那套三层组合里**没有任何一层是 TileLang**，极易误判成"课程镜像已下架"。
**正确的 TileLang 镜像只在选对型号后才出现。**

---

# 附录 A：0 基础版——从零学会连上实例

> 这一章假设你**完全没接触过命令行**。有基础的话看第一节就够了。

## A.1 先搞懂三个词

| 词 | 人话解释 |
|---|---|
| **终端** | 一个能打字的黑框框窗口。你在里面敲命令，电脑照着做。Windows 上叫 **PowerShell** |
| **SSH** | 一种"远程登录"的方式。用它在**自己电脑上**操作**远处那台 GPU 机器**，就像坐在它面前 |
| **提示符** | 那个等着你打字的符号。**看清楚它长什么样，就知道你现在在哪儿** |

**提示符是本附录最重要的概念**，因为它一眼告诉你"我现在在本地还是云端"：

```
PS C:\Users\蒋玉月>              ← 本地（你自己的电脑）
(base) root@0107137682f9:/data#   ← 云端（那台 GPU 机器）
```

**记住这个区别**，能避免 90% 的新手错误（比如在本地窗口敲云端才有的命令——你前面就踩过一次）。

## A.2 完整步骤

### 第 1 步：打开一个终端

按键盘上的 **`Win + R`**（`Win` 是左下角那个窗户图标键）。

弹出一个小框，输入：

```
powershell
```

按回车。出来一个窗口，最后面是一个像 `PS C:\Users\蒋玉月>` 的东西在闪。

**这个窗口现在是"本地窗口"。**

### 第 2 步：敲连接命令

在闪的地方输入这四个字，然后回车：

```
ssh metax
```

> `ssh` = 我要远程登录。`metax` = 我给那台机器起的**外号**，
> 背后对应着一长串真实地址（IP、端口、用户名），写在配置文件里了。
> 所以**你只需要记 `ssh metax` 两个词**。

### 第 3 步：第一次会问一句"信不信它"

```
The authenticity of host '[140.207.205.81]:32222' can't be established.
ED25519 key fingerprint is SHA256:iTSE3EM4e6+mxZnVfRlxOIP4+p25qYOQUyFV3oer06w.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

这是问你"这台机器我以前没见过，要不要信任"。

**输入 `yes`，回车。**（只问这一次，以后不再问。）

> 💡 顺带学会一件事：`SHA256:iTSE3EM4...` 这串是这台机器的**指纹**。
> 正经的 IT 流程里，你应该拿它跟官方给的指纹**对一下**，
> 一致才输 yes——防的是"有人冒充服务器骗你密码"。
> 这一步叫**验证主机指纹**。

### 第 4 步：输密码（最反直觉的一步）

```
root+vm-3S9lrtEHcRqkOMov@140.207.205.81's password:
```

**打字的时候，屏幕上什么都不会出现**——没有圆点、没有星号、光标也不动。

**这不是卡住了，也不是键盘坏了，是故意的**（防止旁边的人看你密码有几位）。

你就照常盲打密码，打完按回车。

> 打错了怎么办？没关系，它会让你重试。**但别连按回车**——空密码也算一次失败。

### 第 5 步：确认连上了

看到这些，就是成功了：

```
TileLang environment configured:
  source: /app/tilelang-metax
  ...
(base) root@0107137682f9:/data#
```

**判断标准：提示符变成了 `root@一串字符` 开头。** 看到它就说明你现在在**云端**了。

### 第 6 步：开始干活

```bash
cd /data/gollamago/assignment/task2
python -m pytest -q test_add.py
```

## A.3 概念：`cd` 和"当前目录"

命令行里**你永远站在某个文件夹里**，这个位置叫"当前目录"。

- `pwd` —— 问"我现在在哪"（print working directory）
- `cd /data/gollamago` —— "带我去这个文件夹"（change directory）
- `ls` —— "列一下这个文件夹里有啥"（list）

**所以 `cd` 到作业目录再跑测试**，不是仪式，是因为 pytest 得在当前目录找到 `solution.py`。
（这也是任务二为什么必须 `cd assignment/task2` 才能跑——仓库没配 pytest 的 rootdir。）

## A.4 怎么退出

```
exit
```

或者按 **`Ctrl + D`**。回到 `PS C:\Users\蒋玉月>` 就是回到本地了。

**注意**：直接点窗口右上角的 ❌ 关掉也行，但**如果云端正在跑任务，可能会被中断**。
养成 `exit` 的习惯。

## A.5 常见报错对照表

| 屏幕上写什么 | 人话 | 怎么办 |
|---|---|---|
| `Are you sure you want to continue connecting` | 第一次连，问你信不信 | 输 `yes` |
| 输密码时**什么都不显示** | **正常现象**，不是坏了 | 盲打完回车 |
| `Permission denied (password)` | 密码不对 | 重新 `ssh metax`，慢点打，注意大小写 |
| `Connection closed by remote host` | 连接被服务端断开 | 可能密码错太多次被临时挡了，等几分钟再试 |
| `Connection timed out` | 网络不通 | 去控制台看**实例是不是关机了** |
| `Could not resolve hostname` | 名字写错了 | 确认敲的是 `ssh metax` |
| `command not found` | 没这个命令 | **先看提示符**：如果还是 `PS C:\...`，说明你在本地窗口敲了云端的命令 |
| `No such file or directory` | 路径不对 | `pwd` 看自己在哪，`ls` 看有什么 |

## A.6 每次开机的口诀

```
控制台开机  →  ssh metax  →  输密码  →  看到 root@ 就是进去了
```

**四个动作，没了。**

进去之后如果想知道环境好没好，敲一句：

```bash
echo $TILELANG_ROOT
```

打出 `/app/tilelang-metax` 就一切正常。

---

# 附录 B：为什么这个平台不能免密登录（写给以后的自己）

试过公钥登录、失败了、**别再试**。原因见第三节第 1 条，一句话版：

> 平台的 SSH 网关是 `SSHPiper`，它**只开密码认证**，而开公钥需要平台运营方在**网关侧**配置
> mapping key——**实例内部改不了**。

两个曾经误导过我们的假象，**再看到别上当**：

1. 往实例 `~/.ssh/authorized_keys` 里塞公钥 → **没用**（那是给 sshpiper 的 upstream 连接用的）
2. 实例里 `sshd -T` 显示 `pubkeyauthentication yes` → **没有参考价值**（那是内层 sshd，网关在先）

**所以：看到密码提示框，正常输密码就行。这是这个平台的设计，不是没配好。**
