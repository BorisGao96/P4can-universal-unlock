# P4-CAN 通用解锁 v3.29.3 BETA（中文说明）

使用 ESP32-P4/C6 开发板 + 3× SN65HVD230 CAN 收发器，解锁全部功能的终极方案。
包含既有通用解锁方案的全部功能 + 防提示音（nag-killer）。

**功能亮点：**
- 防提示音（Nag-killer）
- 高级 EAP（NOA 中自动打转向灯）
- EU 解锁（绕过 R79 欧盟限制）
- 召唤解锁（±85m），已修复 ESP32-P4 稳定性问题
- 激活 TLSSC（若所在地区不可用）
- 持续 AP（Continuous AP，实验性）
- ISA 提示音抑制
- 国家/地图区域伪装
- 视觉速度控制设置
- 油门映射（Pedal map）设置
- S3XY 按钮蓝牙支持（需少量修复）
- 开关：恢复免确认变道设置（EU 下不可用，请使用自动转向灯）**
- 可选禁用驾驶员监控（DMS）

部分功能需要有效的 EAP/FSD 订阅。

支持五种车型档案（Model 3/Y/YL）的多种运行模式。

- **Android 手机仪表盘**：APK 下载链接见英文版 [README.md](README.md) 的 "Android dashboard" 一节

## 接线

| 物理总线 | 控制器 | TX | RX |
|---|---|---|---|
| CAN 1 BODY（车身） | TWAI 0 | GPIO20 | GPIO21 |
| CAN 2 CHASSIS（底盘） | TWAI 1 | GPIO22 | GPIO23 |
| CAN 3 PARTY（娱乐） | TWAI 2 | GPIO26 | GPIO27 |

<img width="718" height="442" alt="ESP32-P4-WIRING" src="https://github.com/user-attachments/assets/7cbf03c8-ebb4-41c9-b624-cad1b8879684" />

## 构建

使用 Arduino-ESP32 3.3.12，选择 `ESP32P4 Dev Module`，post-v3 硅片版本、32 MB Flash、`app5M_fat24M_32MB` 分区、启用 PSRAM、硬件 CDC/JTAG 并开启 CDC on boot：

```text
esp32:esp32:esp32p4:ChipVariant=postv3,FlashSize=32M,PartitionScheme=app5M_fat24M_32MB,PSRAM=enabled,USBMode=hwcdc,CDCOnBoot=cdc,FlashMode=qio
```

推荐配置（追求最高性能与稳定性）：
ESP32-P4 + 3× SN65HVD230，
移除 120 Ω 电阻并启用高速模式。

请检查你的 ESP32-P4 芯片版本（V1.X 或 V3.X）并按对应版本构建。

### ☕ 支持项目

[![Crypto](https://img.shields.io/badge/Crypto-Donate-F7931A?style=for-the-badge&logo=bitcoin&logoColor=white)](https://tesla-mods.fkey.id/) [![PayPal](https://img.shields.io/badge/PayPal-Donate-00457C?style=for-the-badge&logo=paypal&logoColor=white)](https://www.paypal.com/donate/?hosted_button_id=Q2YMMMJYTAPJE)
<a href="https://www.buymeacoffee.com/xbmod" target="_blank"><img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png" alt="Buy Me a Coffee" style="height: 30px !important;width: 108px !important;" ></a>

欢迎请作者喝杯咖啡，助力后续更新~

### Discord

加入项目 Discord 服务器：

https://discord.gg/euPbYG8Npc

## 许可证

GNU GPL v3 或更高版本。详见 [LICENSE](LICENSE.txt)。
