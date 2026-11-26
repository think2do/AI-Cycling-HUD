# AI骑行头显设备

骑士 iOS App、ESP32-S3 头显与小智语音固件，以及 ESP32-C3 车载屏固件。骑士在手机上用高德骑行 SDK 规划路线，经蓝牙将下一步转向和距离发给头显。Mac 上的订单服务可选，用于手机与小智协作准备美团自提，付款由用户在美团完成。

## 目录

| 路径 | 内容 |
| --- | --- |
| [app/骑士Knight-iOS](app/骑士Knight-iOS) | Xcode 工程、App 页面、高德骑行导航、Core Bluetooth |
| [firmware/knight_hud](firmware/knight_hud) | ESP32-S3 蓝牙头显与 SSD1306 OLED 固件 |
| [firmware/knight_vehicle_c3](firmware/knight_vehicle_c3) | PANBOPO 360×360 车载屏、路线图与门店卡片 |
| [firmware/xiaozhi-esp32](firmware/xiaozhi-esp32) | 小智语音固件、AI-MAIN 板型和自提回执 |
| [server](server) | 可选的局域网订单服务、回归测试与头显调试工具 |
| [docs/hud/ai-hud-protocol.md](docs/hud/ai-hud-protocol.md) | 蓝牙 UUID、UTF-8 分片和头显命令 |

## 双 ESP32 显示方案

小智 ESP32-S3 负责语音与头盔 HUD，另一块 ESP32 驱动车载屏。骑士 App 用高德算路，可向 C3 同步路线、位置、转向和距离。实体头盔 HUD 验收待完成。[查看技术方案](docs/architecture/technical-architecture.md)

## 构建 iPhone App

用 Xcode 打开 `app/骑士Knight-iOS/KnightApp.xcodeproj`，选择 `KnightApp`。真机安装前在 Signing & Capabilities 选择自己的开发团队和 Bundle ID，部署目标为 iOS 15 及以上。

如需骑士内导航，在仓库根目录执行下列命令，再将自己申请的高德 iOS Key 写入 `app/骑士Knight-iOS/.tmp/AMapKey.local.xcconfig` 的 `KNIGHT_AMAP_IOS_KEY`。Key 要与实际 Bundle ID 匹配。使用真机 Debug 构建。App 会在开始算路前请求高德隐私同意和系统定位许可。 [高德开放平台](https://lbs.amap.com/)

```sh
bash 'app/骑士Knight-iOS/install_amap_sdk.sh'
```

## 构建头显和小智固件

头显使用 ESP32-S3-WROOM-1 N16R8 与 SSD1306 128×64 I²C OLED。SCL 接 GPIO9，SDA 接 GPIO8，OLED 使用 3.3 V 供电并与开发板共地。有源蜂鸣器信号接 GPIO17，确认键用板载 BOOT。烧录前请核对板型和串口。

```sh
arduino-cli core install esp32:esp32@3.3.12
arduino-cli lib install U8g2@2.36.19
bash firmware/knight_hud/build.sh
```

ESP32-S3 AI-MAIN 小智固件使用 ESP-IDF 5.4.4。环境安装和自定义唤醒词配置见 [小智构建说明](firmware/xiaozhi-esp32/README-自定义唤醒词.md)。

```sh
bash firmware/xiaozhi-esp32/build_aimain.sh
```

## 可选的局域网订单服务

用项目自己的服务配置生成手机和小智的独立凭据。界面中的美团「本机执行」依赖 iPhone 调试授权。订单服务仅在可信局域网使用。

```sh
python3 server/knight_orders.py --configure "$(scutil --get LocalHostName).local"
python3 server/knight_orders.py --interface en0
```

在 Mac 上用 `networksetup -listallhardwareports` 确认实际网卡，按需替换 `en0`。iOS 构建传入 `-xcconfig ~/.local/share/knight-orders/iPhone.xcconfig`。AI-MAIN 固件构建会读取同目录的 `knight_order_build.h`，也可在 ESP-IDF 中用 `KNIGHT_ORDERS_CONFIG_DIR` 指定配置目录。配置、密钥和订单数据库保存在本机。

## 语音任务与车载屏

- 找附近餐厅时，骑士在 iPhone 调试版内只读查询美团，核对城市、地标及评分，再将该门店和最多三道可核实的菜发送到 C3。用户随后说「开始导航」，骑士才启动到同一分店的高德骑行导航。
- 小布可经签名蓝牙把查店、点单和骑行请求交给骑士转发。普通骑行可独立发起，导航期间待办点单会暂停并在结束后续办。车载屏短暂显示订单更新，再继续展示路线。
- 付款在美团由用户完成。查店任务保持购物车和交易状态。

## 验证状态

- 骑士内置高德地图覆盖导航、自提、自由骑行、个人轨迹与打卡。车队位置共享和实体头盔 HUD 待验收。
- 广州黄村地铁站附近的美团语音查店、C3 店名及三道菜显示、语音确认启动高德骑行和 C3 路线同步均已真机验证。地址流程修订后，黄村地铁站 B 口的只读预检已收到 C3 接收确认。修订版本的完整真人语音复测待完成。
- 29 项服务端回归测试通过。

[骑行地图](docs/changes/complete-cycling-maps/proposal.md) · [语音控屏](docs/changes/hands-free-voice-control/proposal.md) · [自提结算及语音启动导航](docs/changes/verify-voice-pickup-navigation/proposal.md) · [美团查店和小屏推荐](docs/changes/voice-meituan-store-recommendation/proposal.md) · [签名蓝牙中继](docs/changes/relay-knight-voice-requests/proposal.md) · [C3 构建与屏幕协议](firmware/knight_vehicle_c3/README.md)

```sh
python3 -m unittest discover -s server
```

## 许可证

项目自有原创代码采用 [Apache-2.0](LICENSE)。小智上游源码保留 [MIT](firmware/xiaozhi-esp32/LICENSE)，Poki 组件保留 [AGPL-3.0-only](app/骑士Knight-iOS/KnightApp/PokiNative/LICENSE)。分发链接 Poki 的 iOS 调试版时，需遵守 AGPL 对组合程序的要求。高德 SDK 由官方按其许可提供。详见 [NOTICE](NOTICE)。

## 协作与仓库说明

本仓库为 `think2do` 组织下的项目仓库，由 [@hxy940219220-collab](https://github.com/hxy940219220-collab) 与 [@206758633-sudo](https://github.com/206758633-sudo) 合作开发。
