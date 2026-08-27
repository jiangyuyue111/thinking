# ESP32-P4 手机网页遥控小车：从零到跑通 + 踩坑复盘

> 目标：给 TinyNav（ESP32-P4-WIFI6-M 模组 + DRV8833 电机 + A010 深度相机）加「手机网页遥控」，从"能不能做"到"全功能跑通"。
> 结论：✅ 网页遥控（前进/后退/转向 + 速度滑块）跑通；❌ 遗留「直接电池上电时主控复位异常」硬件问题（见文末）。

---

## 1. 最终架构（方案二：esp_hosted WiFi）

ESP32-P4 **芯片本身没有 WiFi 射频**，`ESP32-P4-WIFI6-M` 模组 = P4 + 外接 ESP32-C6，靠 SDIO 连。要让 P4 上网，必须走 `esp_wifi_remote` 组件（把标准 `esp_wifi` API 透明转成 RPC 发给 C6）。

**关键选型**：后端用 **esp_hosted**，不是 ePPP。

| 项 | 值 |
|---|---|
| WiFi 后端 | `espressif/esp_wifi_remote 1.6.3` + `espressif/esp_hosted >=2.12` |
| C6 出厂固件 | **ESP-Hosted slave**（微雪板出厂已烧好，不用自己烧 C6） |
| SDIO 引脚 | CMD=52 CLK=51 D0-D3=47~50，reset=9（P4 芯片固定，不用飞线） |
| 网页 | `web_control.c` 标准 SoftAP + HTTP，热点 `TinyNav-Car`，`192.168.4.1` |
| 电机 PWM | **软件 PWM**（esp_timer 1kHz），因为 P4 硬件 PWM 外设失效 |

---

## 2. 五个关键坑（按踩坑顺序，每个都能复用）

### 坑1：ESP32-P4 没有 WiFi，标准 esp_wifi 编译都过不了
- **现象**：`soc_caps.h` 里没有 `SOC_WIFI_SUPPORTED`（对比 esp32s3 有 `=1`），P4 只有 `SOC_WIRELESS_HOST_SUPPORTED`。
- **根因**：P4 芯片设计上无 WiFi/BT 射频，靠外接 C6。
- **解法**：走 `esp_wifi_remote` 组件，标准 `esp_wifi` API 透明可用，业务代码不用改。

### 坑2：ePPP 后端连不上 C6，换 esp_hosted 秒通
- **现象**：ePPP 方案 `sdmmc_card_init failed`，C6 没响应。
- **根因**：微雪板 C6 **出厂固件是 ESP-Hosted slave**，不是 ePPP slave。两者协议不兼容。
- **解法**：改 `esp_hosted` 后端，一次通过。
- **复用**：遇到"外接协处理器连不上"，先查**对方出厂固件是什么协议**，别急着猜。

### 坑3：ESP_IDF_VERSION 环境变量必须是 "6.0"（不带 patch）
- **现象**：编译链接报 `WIFI_RMT_STATIC_RX_BUFFER_NUM undeclared`。
- **根因**：设成 `6.0.0` 导致 Kconfig 去加载不存在的 `Kconfig.idf_v6.0.0.in`，`WIFI_RMT_*` 配置全没生成。
- **解法**：`idf_tools.py` 里 `ESP_IDF_VERSION = major.minor`，即 `"6.0"`。
- **复用**：组件里出现 `orsource "Kconfig.idf_v$ESP_IDF_VERSION.in"` 这类，先确认版本号的**精确格式**。

### 坑4：esp-idf v6.0 不支持中文路径（cmake 栈溢出崩溃）
- **现象**：`cmake` 一跑就崩（exit 0xC0000409，无任何输出），但最小项目能过。
- **根因**：esp-idf v6.0 的 cmake/project.cmake 在含中文路径（`C:\Users\蒋玉月`）下崩溃，任何 cmake 版本都崩。
- **解法**：项目移到 `D:\TinyNav`（无中文），原位置建 junction 软链保持旧路径可访问。
- **复用**：Windows 上 esp-idf 项目**路径必须纯 ASCII**。

### 坑5：P4 的 MCPWM 和 LEDC 外设输出都失效
- **现象**：配置完全正确、duty 寄存器能写进去（`ledc_get_duty` 读回正确），但引脚就是没 PWM。GPIO 直接 `set_level` 能转，证明硬件链通。
- **根因**：IDF v6.0 在 P4 上，PWM 外设（MCPWM/LEDC）的 GPIO 输出不生效（疑似 matrix 路由 bug，未深挖）。
- **解法**：**软件 PWM**——esp_timer 50µs 回调翻转 GPIO，1kHz、5% 分辨率，电机调速完全够用。
- **复用**：P4 上要 PWM，**优先考虑软件 PWM 或 RMT**，别在 MCPWM/LEDC 上耗时间。

---

## 3. 环境搭建的可复用方法（国内网络）

| 需求 | 方案 |
|---|---|
| 拉 esp-idf v6.0 源码 | gitee 镜像 `gitee.com/mirrors/esp-idf`（有 v6.0 tag） |
| 拉 submodule | 改 `.gitmodules` 指向 github，再用 `gh-proxy.com` 加速（2MB/s），个别失败就 `git submodule update --init <path>` 单独拉 |
| 工具链下载 | `gh-proxy.com` 下 riscv32-esp-elf/cmake（espressif 官方源只有 563B/s，慢得没法用） |
| pip 源 | 阿里云镜像 `mirrors.aliyun.com/pypi/simple` |
| 组件（esp_wifi_remote 等） | `components.espressif.com` 直连可达 |

**烧录/编译环境变量**（Windows PowerShell）：
```
IDF_PATH / IDF_TOOLS_PATH / IDF_PYTHON_ENV_PATH
ESP_IDF_VERSION="6.0"  (注意！)
ESP_ROM_ELF_DIR=...\esp-rom-elfs\20241011\
PATH += riscv32-esp-elf\bin; cmake\bin; ninja
```

---

## 4. 遗留问题：直接电池上电时主控复位异常（未解决）

- **现象**：插 USB（串口）上电 → 主控正常、热点起来、一切正常，之后断电重启也 OK。但**一开始就直接电池上电** → 主控没启动（无热点），电机间歇性乱跑。
- **定位过程**（有复用价值）：
  1. 先排除软件：回退到之前确认能用的固件，问题依旧 → 不是固件。
  2. 量电压：降压模块空载 5.1V，**带载（接板子）跌到 4.7V** → 带载压降 0.4V。
  3. 推断：主控启动瞬间电流大，电压进一步跌 → 跌破工作阈值 → 反复复位 → 无热点 + GPIO 乱 → 电机间歇性跑。
- **根因方向**：供电带载能力/上电瞬间稳定性不足（不是电压数值，是瞬态跌落）。
- **待验证的解法**：降压模块 5V 输出并联 1000µF 电解电容储能；换粗电源线；电池充满。

---

## 5. 一句话方法论

> 硬件联调遇到"接 USB 正常、独立供电异常"，先怀疑**上电瞬间的供电稳定性**（带载压降、复位时序），别先怀疑代码。
