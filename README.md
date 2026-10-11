# gamma ファームウェア

gamma 機体（beta の次世代機）のメインボード（`gamma_turret`、ESP32-P4）ファームウェア。

gamma は基板3枚セット（`gamma_turret` / `gamma_neck` / `gamma_chassis`）で構成されるが、
マイコンを載せるのは砲塔の `gamma_turret` のみ。本リポジトリはその制御ファームウェアを管理する。
`gamma_neck` / `gamma_chassis` はマイコン無しの受動分配基板（電源・CAN の分配のみ）。

> **ステータス凡例**
> - ✅ 確定・実装済み
> - 🔧 実装中
> - 💭 構想段階（変更の可能性あり）

---

## システム構成・データフロー

```
[ Jetson Orin Nano ]
  │ UART（3.3V TTL）
  ▼
[ gamma_turret（M5Stack Stamp-P4 / ESP32-P4） ]  ← BNO055 IMU（UART、基板直載せ）
  ├── CAN1（砲塔・ジンバル系）
  │     ├── DM3507  × 2 → 首 Roll / Pitch 軸
  │     └── MHF5005 × 1 → 首 Yaw 軸
  ├── CAN2（車体系）
  │     │   スリップリング経由で gamma_neck → gamma_chassis へ
  │     ├── CAN_CHASSIS 1〜4
  │     └── CAN_LEG 1〜4
  ├── PWM × 2 → ESC → 射出ローラー × 2
  └── UART（半二重、1kΩ 合流） → ST3215 サーボ（装填）
```

### 基板間接続

| 区間 | 内容 |
|---|---|
| gamma_turret ⇔ gamma_neck | 24V 幹線、CAN1（首モータへ）、CAN2（パススルー） |
| gamma_neck ⇔ gamma_chassis | スリップリング経由。24V、CAN2、ゲームモジュール線（PA 4P） |
| gamma_chassis → 各脚・車体機構 | 24V 分配、CAN2 分配（CAN_CHASSIS1〜4 / CAN_LEG1〜4） |

### 電源系統

| 電圧 | 生成元 | 供給先 |
|---|---|---|
| 24V | 幹線 | 首モータ3台、P4 キャリア、車体側 |
| 7.4V | 外部 DCDC（XT30 で往復） | ST3215 サーボ、ESC |
| 5V | M78AR05-1（1A、24V から） | Stamp-P4 |
| 3.3V | Stamp-P4 内蔵 DCDC | BNO055、CAN トランシーバ VIO、Jetson UART |

---

## 基板詳細

### gamma_turret（制御基板）

| 項目 | 内容 | 状態 |
|---|---|---|
| コア | M5Stack Stamp-P4（ESP32-P4、16MB Flash / 32MB PSRAM）、ピンソケットで差し替え | ✅ |
| CAN | 2ch（MCP2562FD × 2、VIO = 3.3V） | ✅ |
| ESC PWM | 2ch（射出ローラー） | ✅ |
| ST3215 UART | 半二重バス（1kΩ 合流回路） | ✅ |
| Jetson UART | 3.3V TTL | ✅ |
| IMU | BNO055 を基板直載せ、UART 接続 | ✅ |
| HMI | LED × 6（電源 / デバッグ / アクティビティ）、BOOT / RESET タクト SW | ✅ |
| 基板状態 | 2026-08-10 設計承認待ち（ピン配置・回路方式・BOM 確定済み） | 🔧 |

#### 受信: Jetson からの指令

<!-- TODO: プロトコル（rabcl の 28B/112B を踏襲するか、新規定義か）、ボーレート -->

#### アクチュエータ制御

| アクチュエータ | 通信 | 内容 | 状態 |
|---|---|---|---|
| 首 Yaw 軸 | CAN1（LK モータ） | MHF5005 | 💭 |
| 首 Pitch 軸 | CAN1（DAMIAO） | DM3507 | 💭 |
| 首 Roll 軸 | CAN1（DAMIAO） | DM3507 | 💭 |
| 車体（CAN_CHASSIS 1〜4） | CAN2 | <!-- TODO: モータ種別 --> | 💭 |
| 脚（CAN_LEG 1〜4） | CAN2 | <!-- TODO: モータ種別 --> | 💭 |
| 射出ローラー × 2 | PWM → ESC | <!-- TODO: 周波数、パルス幅 --> | 💭 |
| 装填 | 半二重 UART | ST3215 サーボ | 💭 |

#### CAN ID 一覧

<!-- TODO: CAN1 / CAN2 それぞれの ID 割り当て -->

| バス | 定数名 | TX ID | RX ID | 内容 |
|---|---|---|---|---|

#### ファームウェア申し送り（ハード設計からの制約）

- 当面 `CONFIG_FREERTOS_UNICORE=y`（デュアルコアハング回避）
- GPIO26 / GPIO27 の USB 機能を無効化
- UART0 ログ出力を無効化
- TWAI のループバック自己試験は使わない

### gamma_neck（受動分配基板）

- スリップリング（車体 ⇔ 砲塔）の受け口
- 24V 分配（首モータ3台 + P4 キャリア）
- CAN 分配（CAN1 → 首モータ、CAN2 → P4 へパススルー）
- ゲームモジュール線（PA 4P）パススルー
- CAN 終端抵抗は基板に載せず、モータ内蔵の切替終端で運用（バス最遠の1台のみ ON）
- 状態: 2026-08-12 spec 最終版・案A（純受動・最小構成）承認済み。これから KiCad で作図

> ⚠️ リング電源入力と MHF5005 出力が同じ PH 6P コネクタ。挿し違えると 24V がモータの CAN ピンに入る。
> シルク表記と組立時確認で担保する。

### gamma_chassis（受動分配ハブ）

- 24V 入力（6P）と電源分配（Power1〜8 / Vout1）
- CAN2 分配: CAN_CHASSIS1〜4 / CAN_LEG1〜4 / CAN_TURRET1
- 状態: gamma 向けに継続更新中（最新 2026-10-10）

---

## beta からの変更点

| 項目 | beta | gamma | 状態 |
|---|---|---|---|
| マイコン | STM32F303K8（Nucleo） | **ESP32-P4（M5Stack Stamp-P4）** | ✅ |
| ボード構成 | 1 枚 | **3 枚（turret = 制御、neck / chassis = 受動分配）** | ✅ |
| 上位コントローラ | Raspberry Pi 5 | **Jetson Orin Nano** | ✅ |
| CAN | 1 系統（1 Mbps） | **2 系統（CAN1 = 砲塔・ジンバル、CAN2 = 車体）** | ✅ |
| 砲塔軸 | Yaw + Pitch の 2 軸 | **Yaw + Pitch + Roll の 3 軸ジンバル** | ✅ |
| Yaw 軸モータ | MHF5005（LK、トルク制御） | MHF5005（LK） | 🔧 |
| Pitch 軸モータ | DM3507（DAMIAO MIT モード） | DM3507（DAMIAO） | 🔧 |
| Roll 軸モータ | なし | **DM3507（DAMIAO）** | 🔧 |
| 車輪 / 脚 | DM2325 × 4（CAN 速度制御） | **CAN_CHASSIS × 4 + CAN_LEG × 4** <!-- TODO: モータ種別 --> | 💭 |
| 装填 | 市販 MD + DC モータ（PWM） | **ST3215 サーボ（半二重 UART）** | 💭 |
| 射出 | ESC × 2（PWM） | ESC × 2（PWM） | 💭 |
| IMU | BNO055（I2C、無効化中） | **BNO055（UART、基板直載せ）** | 💭 |
| 車体 ⇔ 砲塔接続 | 直結 | **スリップリング** | ✅ |

---

## 共通仕様

### クロック・CAN ビットレート

<!-- TODO: TWAI のビットレート、制御周期 -->

| 項目 | 値 |
|---|---|

### モータ設定値・制御パラメータ

<!-- TODO -->

---

## 依存ライブラリ

<!-- TODO: rabcl を使い続けるか（ESP-IDF 上でのビルド方法）-->

---

## ビルド方法

<!-- TODO: ESP-IDF か PlatformIO か、ターゲット設定、書き込み手順 -->
