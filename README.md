# gamma STM32 ファームウェア

gamma 機体（beta の次世代機）のメインボードファームウェア。

> **ステータス凡例**
> - ✅ 確定・実装済み
> - 🔧 実装中
> - 💭 構想段階（変更の可能性あり）

---

## システム構成・データフロー

<!-- TODO: 上位（Raspberry Pi / M5 など）→ main_board → 各モータ・センサ の構成図 -->

```
[ 上位コントローラ ]
  │ 
  ▼
[ main_board ]
  ├── 
  └── 
```

---

## 基板詳細

### main_board

<!-- TODO: マイコン、受信インタフェース、モータ制御表、CAN ID 一覧、送信方式 -->

#### 受信: 上位からの指令

#### モータ制御

| モータ | 通信 | 内容 | 状態 |
|---|---|---|---|

#### CAN ID 一覧

| 定数名 | TX ID | RX ID | 内容 |
|---|---|---|---|

#### CAN TX 送信方式

---

## beta からの変更点

| 項目 | beta | gamma | 状態 |
|---|---|---|---|
| ボード構成 | 1 枚（Nucleo F303K8） |  |  |
| 車輪モータ | DM2325 × 4（DAMIAO CAN、速度制御） |  |  |
| Pitch 軸モータ | DM3507（DAMIAO MIT モード、CAN 0x42/0x52） |  |  |
| Yaw 軸モータ | MHF5005（LK モータ、CAN 0x141、トルク制御） |  |  |
| 上位との通信 | UART 230400 bps（rabcl 28B 指令 / 112B フィードバック） |  |  |
| IMU | BNO055（無効化中） |  |  |
| システムクロック | 36 MHz（HSI + PLL×9） |  |  |

---

## 共通仕様

### クロック・CAN ビットレート

| 項目 | 値 |
|---|---|

### モータ設定値・制御パラメータ

---

## 依存ライブラリ

[rabcl](https://github.com/rabbits-robotics/rabcl) をサブモジュールとして使用。

```bash
git submodule update --init
```

---

## ビルド方法

<!-- TODO: beta と同じく CubeMX → stm32pio → PlatformIO か確認 -->
