# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目简介

ESP32-P4 开发板（+ ESP32-C6 无线模块，经 ESP-Hosted/SDIO 驱动）的 Arduino 固件，使用 3 个原生 TWAI CAN 控制器，通过 CAN 总线为特斯拉 Model 3/Y/YL 解锁功能：防提示音（nag-killer）、EU R79 解锁、召唤解锁（Summon）、持续 AP、ISA 提示音抑制、TLSSC 激活、国家/地区伪装、S3XY 按钮 BLE 支持、驾驶员监控捕获等。板载 Wi-Fi AP（192.168.4.1）提供 Web 仪表盘和 OTA 升级。

## 构建

目标板设置（Arduino-ESP32 3.3.12 / ESP-IDF 5.5.x）：

```text
esp32:esp32:esp32p4:ChipVariant=postv3,FlashSize=32M,PartitionScheme=app5M_fat24M_32MB,PSRAM=enabled,USBMode=hwcdc,CDCOnBoot=cdc,FlashMode=qio
```

- `python tools/build_dashboard.py` — 从 `dashboard_source.html` 重新生成 `index_html.h`（确定性 gzip，`mtime=0`）。修改仪表盘后必须重新生成；绝不要手工编辑 `index_html.h`。
- `python tools/build_p4.py` — 完整 OTA 构建；依赖外部工作区发布器（上级工作区目录中的 `tools/build_ota.py`，本包需位于 `firmware/universal/` 下）。产物为 `P4-CAN-build/compile.log` 和 `manifest.json`（含源码/二进制 SHA-256 哈希）。
- `build_opt.h` 添加 `-fno-exceptions`；保留。
- 板卡引脚变体：使用 X179 扩展板时在编译期定义 `-DP4CAN_BOARD_X179`（见 `pin_config.h`；总线按总线名而非丝印编号映射）。

## 测试

仅为主机（PC）测试 —— 本仓库不会刷写硬件。在仓库根目录运行：

```sh
# 仪表盘结构/一致性检查（同时强制全仓库禁用规则，见下）；需要 PATH 中有 node
python -X utf8 tests/test_dashboard.py

# 在模拟 DOM 上执行真实的仪表盘 JS 函数
node tests/test_dashboard.js

# 基于 TWAI 假驱动的 C++ 测试（ASan/UBSan）
g++ -std=c++17 -Wall -Wextra -Werror -fsanitize=address,undefined -Itests/fakes -I. tests/test_p4.cpp -o /tmp/p4can_test && /tmp/p4can_test
g++ -std=c++17 -Wall -Wextra -Werror -fsanitize=address,undefined -I. tests/test_network.cpp -o /tmp/p4network_test && /tmp/p4network_test

# BLE 加密（TinyCrypt ECC 回移）测试
g++ -std=c++17 tests/test_p4_crypto_selftest.cpp -o /tmp/p4crypto_selftest && /tmp/p4crypto_selftest
python -X utf8 tests/test_p4_tinycrypt_host.py
python -X utf8 tests/test_p4_crypto_link.py   # 需要构建产物（ELF/map）
```

`tests/test_dashboard.py` 会对所有 `*.h`、`*.ino`、`*.md` 强制执行全仓库不变量：禁止 `MCP2515`/`SPI` 引用、禁止旧方案命名（正则 `t-?2-?can`，注意任何 md 文件中出现都会导致测试失败）、禁止在原生适配器之外直接调用 `twai_transmit/receive/driver_install/...`（`p4_can.h` 是唯一拥有 TWAI API 调用权的文件）。编写文档时避免使用旧产品名。

## 架构

### 单一编译单元

`.ino` 按精心安排的顺序包含约 50 个头文件，整体编译为一个 sketch。多个头文件带有注释 "Kept in the same translation unit to preserve proven runtime behavior"（保持在同一编译单元以保留已验证的运行时行为）—— 该包含顺序是功能性的：

1. `p4can_core_state.h` → `p4can_forward.h`（显式原型声明，取代 Arduino 自动原型生成器）
2. `*_pure.h` 纯逻辑 → 持有状态的各头文件（`s3xy_ble.h`、`can_core.h`、`can_busoff_persistence.h`、`vehicle_logic.h`、`continuous_ap_runtime.h`、`web_api.h`、`can_runtime.h`、`can_usb_logger_task.h`）

各头文件在文件作用域定义 `static` 全局变量，因此包含顺序决定初始化与锁的次序（例如 `mode_h_runtime.h` 由 `can_core.h` 在共享 NAG 上下文之后包含；`can_input_arbitration.h` 在两个运行时状态所有者之后包含）。新增模块时：纯逻辑放 `*_pure.h`（不含 Arduino 头文件、无 I/O、可在主机上单测），状态/任务放在运行时头文件中，并在 `.ino` 的正确位置包含。

### Pure / Runtime 拆分

`*_pure.h` 刻意不依赖 Arduino，可直接被主机测试覆盖。对应的运行时头文件拥有 FreeRTOS 任务、portMUX 保护的状态和 NVS。新功能请沿用此拆分。

### 启动流程（在 .ino 中）

NVS 初始化 → 读取车辆配置档案；任何 NVS/档案失败都会进入**故障关闭的 SAFE SETUP MODE**（仅 Wi-Fi/Web/OTA/档案选择，不初始化 CAN/BLE，不擦除 NVS）。档案有效时：加载各配置（`featureCfgLoad`、`r79CfgLoad` 等，含 schema 迁移），按物理顺序 0/1/2 安装 3 路 TWAI，启动 CAN RX 任务与监控任务，随后启动 Wi-Fi AP/Web 任务，最后在蓝牙主控开关开启时启动 S3XY BLE 任务。

### 关键层次

- `p4_can.h` / `can_runtime.h` — 原生 TWAI 适配器、各总线 RX 任务、bus-off 恢复监控；全仓库唯一调用 TWAI 驱动 API 的位置。
- `can_core.h` — 启动捕获、防提示音（nag-killer，含 Mode H 拟人变体）、共享 TX 屏障（`canTxBarrierMutex`，串行化所有注入）。
- `vehicle_logic.h` — R79 EU 策略、召唤、自动变道、车辆记录器。
- `web_api.h` — Web 服务器、REST API、OTA；提供 gzip 压缩的 `index_html.h`。
- `can_usb_logger*.h` — USB CDC CAN 日志器（协议 2）；需要硬件 CDC + 开机 CDC，并要求关闭诊断（.ino 中有编译期 `#error` 保护）。
- `vehicle_profile.h` — 5 种车型档案（Model YL / Y Juniper / Y Legacy / 3 Highland / 3 Legacy）及总线拓扑；所有档案均使用全部三条总线（BODY/CHASSIS/PARTY）。
- `src/p4_tinycrypt/` — 包内自带的 Espressif TinyCrypt ECC 修复回移（上游提交 `9fd7cb7e...`），修复 NimBLE 配对失败码 `1291`（DHKey 校验）。Arduino 会自动编译 `src/`；`third_party/tinycrypt/` 存放其引用的上游头文件。`tools/verify_p4_crypto.py` 校验构建出的 ELF 将这些符号绑定到本包对象 —— 发布器在校验失败时阻止生成产物。

### 仪表盘

`dashboard_source.html`（可编辑，约 365 KB）→ `tools/build_dashboard.py` → `index_html.h`（生成的字节块）。`tests/test_dashboard.py` 断言解压后的头文件与源文件逐字节一致，且生成结果可复现；两者必须同步提交。

## 约定

- `SOURCE_BASELINE.json` 记录每个受跟踪源文件的 SHA-256；修改源码后需重新生成（发布器会校验 `sourceMatchesWorkspace`）。
- `VALIDATION.md` 和 `CHANGELOG.md` 记录每个版本的证据与基线保留承诺。本项目的纪律是：除非变更明确说明，否则 CAN 策略、引脚映射、NVS schema 和仪表盘字节在版本间保持不变 —— 改动这些区域前先查阅上述文档。
- 日志使用 `P4CAN_SERIAL_PRINTF`/`P4CAN_SERIAL_PRINTLN`（除非开启 `P4CAN_SERIAL_DIAGNOSTICS` 否则静默），因为 USB CDC 保留给 CAN 日志协议。
- 许可证：GPL-3.0-or-later。
