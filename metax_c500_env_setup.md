# 沐曦 C500 算子开发环境搭建（附：怎么识别一份假的搭建指南）

> 目的：记录在模力方舟上搭建 沐曦曦云 C500 + TileLang + 九齿(NineToothed) 开发环境的过程。
> 这次真正花时间的不是配环境，而是**发现手上那份"参考资料"是 AI 编的**。
> 所以这篇一半讲怎么搭，一半讲怎么在动手之前把假信息筛掉。

## 结论（TL;DR）

1. **一次都没打开过目标仓库的"环境搭建指南"，可以直接判定不可信。** 本次那份资料虚构了镜像全称、工具链命令和整个目录树，而正确答案一直躺在仓库 README 第 4 行。
2. 幻觉有一个稳定指纹：**拿熟悉技术栈的心智模型去套陌生技术栈**——把 CUDA 的 `nvcc` 套成 `jiuci`，把 `xxx --version` 当万能验证手段，把目录树编成 `tutorials/lessonNN-topic/`。
3. 证伪成本极低：`git ls-remote` 一条命令，30 秒。
4. 搭建本身的坑不多，但**每一个都是"装坏了只能重建实例"级别的**，所以值得写成可反复跑的体检脚本。

---

## 一、那份假资料错在哪（幻觉指纹）

| 资料的说法 | 真实情况 |
|---|---|
| 仓库 `ODTCAIInfra/go-llama-go` | ❌ `remote: Not found`。真实：**`ccf-ai-infra/gollamago`** |
| 镜像「沐曦 MXMACA 多范式算子开发环境（算子开发实战营专用）」 | ❌ 无此镜像。真名在**「基础镜像」**分类下 |
| 按关键词 `MXMACA`/`九齿`/`算子开发` 搜镜像 | ❌ 搜不到，镜像名里根本没这三个词 |
| `jiuci --version` | ❌ 无此命令。"九齿"= **NineToothed**，`pip install ninetoothed` |
| `jiuci build vector_add.jiuci -o vector_add_kernel.o` | ❌ 不存在这种编译流程 |
| `tilelang --version` | ❌ TileLang 是 Python 包，靠 `source ./setup_env.sh` 挂载，无 CLI |
| `tutorials/lesson01-env-check/`、`examples/01-vector-add/`、`final-project/` | ❌ 全不存在。真实为 `assignment/task1~4/` + `operators/` + `course/` |
| `source /opt/mxmaca/env.sh` | ❌ 路径是 `/opt/maca`，脚本是仓库自带的 `./setup_env.sh` |

### 提炼出的四类幻觉指纹

| 指纹 | 本次例子 | 为什么会这样 |
|---|---|---|
| **把熟悉栈的 CLI 形状硬套** | `jiuci --version`、`jiuci build x.jiuci -o x.o` | 模型见过无数遍 `nvcc x.cu -o x`，遇到没见过的"九齿"就照形状填了个名字进去 |
| **编造带修饰语的产品全称** | 「沐曦 MXMACA 多范式算子开发环境（算子开发实战营专用）」 | 写产品名时习惯性加括号补语，读起来很正式，实际没有出处 |
| **目录树过于整齐** | `tutorials/lesson01-env-check/lesson02-jiuci-basic/01-vector-add/` | 按"教程应该长什么样"生成，而不是按"这个仓库实际长什么样" |
| **给出未经验证的检索建议** | "镜像搜索框输入 MXMACA/九齿/算子开发" | 关键词从概念推导而来，不是从真实结果里看来的 |

**共同点：描述里没有任何一个能对上真实物件的具体细节。** 全是形状正确的空壳。

---

## 二、验证真伪的四步（成本递增，前两步就够）

1. **`git ls-remote <url>`** —— 30 秒。返回 `remote: Not found` 即证伪。比打开浏览器快得多。
2. **读目标仓库 README 的前几行** —— 本次镜像全名就在 README **第 4 行**，作者写得明明白白。
3. **翻仓库里的截图** —— 作者自己截的界面图是**最强证据**。本次 `image.png` 直接拍到了镜像选择框：`TileLang / 0.1.9 / Python 3.12 / maca 3.3.0.4`。
4. **读源码** —— 用来验证"作业里留了 TODO"这类断言。本次确认 `solution.py:16` 和 `ninetoothed_add.py:20,24` 确实是 `raise NotImplementedError`，所以"初次 pytest 会失败"是真的、且是预期的。

### 信息源可信度排序（本次实测有效）

```
仓库内截图  ≈  仓库源码  >  仓库 README  >  官方活动页  >  搜索引擎  >  别人转述的"参考资料"
```

判据只有一条：**转述者的每句话，能不能对上仓库里的一个具体物件**（一行代码、一张图、一个文件名）。对不上就是编的。

---

## 三、环境本体：沐曦软件栈知识点

### MACA / MXMACA

- 沐曦的 CUDA 对标栈，安装路径 `/opt/maca`
- 目录：`lib/`（运行时）、`mxgpu_llvm/`（编译器与 bin）
- **兼容 CUDA 命名是刻意设计，不是笔误**：MACA 版 PyTorch 仍然写 `--device cuda`，靠 `--target` 区分 `cuda`/`maca`。这是国产栈降低迁移成本的普遍策略。
  → **看到国产栈里出现 `cuda` 字样，先别急着当成错误改掉。**
- `mx-smi` 对标 `nvidia-smi`：`mx-smi`（概览）、`-l 1000`（1 秒刷新）、`--show-usage/-memory/-temperature/-process`
- `mxcc` 对标 `nvcc`（这个是真的存在的），但只影响编译 `.maca` 内核的后端

### `source` 来的环境脚本（TileLang）

TileLang 在镜像里是**源码树形态**部署的，不是 pip 装的 wheel：

```
/app/tilelang-metax/
├── tilelang/__init__.py
└── build/lib/libtilelang.so      # 已编译好的 native 库
```

`setup_env.sh` 干三件事（所以叫"开发环境"而不是"运行时"）：

| 变量 | 值 |
|---|---|
| `PYTHONPATH` 前置 | `$TILELANG_ROOT`（默认探测 `/app/tilelang-metax`、`/data/tilelang-metax`） |
| `LD_LIBRARY_PATH` 前置 | `/opt/maca/lib`、`/opt/maca/mxgpu_llvm/lib` |
| `PATH` 前置 | `/opt/maca/mxgpu_llvm/bin` |

三个必记的推论：

1. **必须在 `import tilelang` 之前 source**，顺序反了 Python 找不到源码树。
2. **必须 `source`，不能 `bash`** —— 脚本第一件事就是自检：
   ```bash
   if [[ "${BASH_SOURCE[0]}" == "$0" ]]; then
       echo "Run this script with: source ./setup_env.sh" >&2
       exit 2
   fi
   ```
   这是个很值得学的惯用法：**让脚本自己拒绝被错误地执行**，比写十行文档管用。
3. **不持久** —— 新开 shell、重连 SSH 都要重新 source。写进 `~/.bashrc`：
   ```bash
   echo '[ -f /data/gollamago/setup_env.sh ] && source /data/gollamago/setup_env.sh' >> ~/.bashrc
   ```

另外一个副作用：TileLang 走 JIT 编译，**第一次调用的耗时包含编译时间，不能拿来评价性能**。

### 九齿 = NineToothed

- Python 库（InfiniTensor 出品），`pip install ninetoothed`，底层跑在 Triton 上
- **没有独立的编译器或 CLI**，这是那份资料最大的破绽
- 作业形态是填两个函数：`arrangement()` 负责按 block 对齐输入输出，`application()` 负责 tile 内的逐元素计算

---

## 四、搭建脚本的设计模式（比脚本内容更值得记）

这次写的 `bootstrap.sh` 有六个刻意设计，都是可以搬到下一个项目里的：

### 1. 诊断脚本不要 `set -e`

```bash
set -uo pipefail    # 注意：没有 -e
```

`-e` 会让脚本在**第一个**错误处退出。而诊断脚本的目标恰恰相反：**一次跑完，把所有问题都收集出来**。
否则你会陷入"修一个 → 跑一次 → 又发现下一个"的循环，而这些问题本来可以一次性看完。

> 判断依据：脚本的产出是**报告**还是**结果**？报告类脚本不该 `set -e`。

### 2. 三态输出，而不是二态

```bash
ok()   { echo "  [OK]   $*"; PASS=$((PASS+1)); }
bad()  { echo "  [FAIL] $*"; FAIL=$((FAIL+1)); }
note() { echo "  [..]   $*"; }      # 不是错误，但你需要知道
```

`note` 这一态是关键。例如 "`mxcc` 不在 PATH" —— 它**不是错误**（TileLang 和九齿都不受影响），但影响 `maca_cpp` 后端。
二态输出会把它报成 FAIL，让人去修一个根本不用修的东西。

### 3. 每个 FAIL 后面跟一句"这说明什么"

```
[FAIL] mx-smi 不存在 —— 镜像大概率选错了，回去确认是不是 maca 镜像
```

不是 `command not found` 的复读，而是**指向下一步动作**。同样的信息量，一个要你再想三步，一个可以直接照做。

### 4. 不可逆操作的前后快照

```bash
TORCH_BEFORE="$(pip show torch | grep -i '^Version')"
python -m pip install -r requirements.txt
TORCH_AFTER="$(pip show torch | grep -i '^Version')"
[ "${TORCH_BEFORE}" != "${TORCH_AFTER}" ] && bad "torch 版本被改动了！"
```

`requirements.txt` 里 `torch` 没写版本号 → pip 认为已装的 `+metax` 版满足要求、不会动它。
但这**是 pip 的实现细节，不是承诺**。一旦被换成 PyPI 的 `+cu121`，MACA 就废了，且**不可逆**（只能重建实例）。

> 通用原则：**对"做坏了只能重来"的操作，装前装后各记一次状态。** 代价两行代码，收益是一个不会漏掉的告警。

### 5. 幂等

```bash
[ -d "${WORKDIR}/.git" ] && git pull --ff-only || git clone "${REPO_URL}" "${WORKDIR}"
```

能不能反复跑，是"脚本"和"体检工具"的分界线。幂等的脚本可以随时跑一遍确认环境还好着，不必担心它把什么搞坏。

### 6. 结尾要诚实

```
注意：上面 pytest 报 'failed' 很可能是正常的 ——
作业入口文件里留着 TODO，本来就还没实现。
```

脚本最后主动说明**哪些红是预期的**。否则第一次跑的人会拿着这个红去修一个不存在的环境问题。
这和"报告结果要如实"是一回事：**不能只报结果，不报怎么解读这个结果**。

---

## 五、零碎但会再遇到的坑

- **`scp` 的端口参数是大写 `-P`**。小写 `-p` 是"保留文件时间戳"，参数写错不会报错，只会连不上。
- **Windows 上写 shell 脚本必须确认 LF 行尾**。用字节计数验证比肉眼看可靠：
  ```powershell
  $b = [IO.File]::ReadAllBytes($p)
  ($b | Where-Object { $_ -eq 13 }).Count    # CR 的个数，必须是 0
  ```
- **PowerShell 用 `Get-Content` 读 UTF-8 中文文件会按 GBK 解码成乱码**。这**不代表文件坏了**——只是 PowerShell 的解码问题。要判断内容别用它下结论。
- **GitLink 的 `git clone` 偶发 `curl 56 ... early EOF`**（本次撞了两次），直接重试即可，不是配置问题。
- **pytest 没有配置 rootdir**（`pytest.ini` 里只有 marker 定义），必须 `cd` 进作业目录再跑，否则 `from solution import ...` 直接 import 失败。

---

## 六、一句话方法论

> **"能编译 ≠ 能跑"（裸机版）的姊妹版是：**
> **"文档说能跑 ≠ 环境里有那个命令"。**

两者同源，都是**把假设的运行时环境当成了实际的运行时环境**：

| | 假设方 | 现实 | 后果 | 验证手段 |
|---|---|---|---|---|
| 裸机启动 | 编译器假设有 ELF loader 和栈 | 都没有 | 编译通过，跑死了 | `objdump -d` 看真实编译产物 |
| 环境搭建 | 文档假设有 `jiuci` 命令 | 命令不存在 | 照做报 `command not found`，然后开始怀疑环境、重装、折腾 | `git ls-remote` 看仓库到底存不存在 |

共同点是：**问题在第一步之前就该被发现，只是没人去验证那个前提。**

---

## 环境信息备忘

| 项 | 值 |
|---|---|
| 课程 | 沐曦「多范式算子开发实战营」（9/13 开营，10/18 项目截止） |
| 仓库 | <https://www.gitlink.org.cn/ccf-ai-infra/gollamago> |
| 镜像 | 模力方舟 → 预装镜像 → **基础镜像** → `TileLang / 0.1.9 / Python 3.12 / maca 3.3.0.4` |
| 硬件 | 沐曦 曦云 C500（32G ≈ ¥2.5/h，64G ≈ ¥4/h），算力券抵扣 |
| 平台 | 模力方舟算力市场 <https://moark.com/compute> |
| 官方工作目录 | `/data/gollamago` |
| 四个后端 | `torch`(基线) / `tilelang` / `ninetoothed` / `maca_cpp` |
| 后端 target | `auto` / `cuda` / `maca`（MACA 下 `--device` 仍写 `cuda`） |
