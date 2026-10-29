# デバイス仕様と観測情報

## 製品仕様（公開情報）

ベンダー公表値。実測値ではない。

| 項目 | 内容 |
|---|---|
| 製品名 | REDMI Buds 8 Pro |
| 発表 | 中国 2026-01-29 / 日本 2026-02-24 |
| ドライバー | 11mm ダイナミック + 6.7mm セラミック（同軸） |
| DAC | デュアル DAC |
| ANC | 最大 55dB（Xiaomi Deep Space Noise Cancellation 3.0） |
| マイク | 3 基、風速 12m/s 対応 |
| コーデック | SBC / AAC / LDAC（Hi-Res Wireless） |
| Bluetooth | 5.4、LE Audio / Auracast ready |
| 空間オーディオ | Xiaomi dimensional audio（ヘッドトラッキング対応） |
| ヘッドトラッキング | 6 軸ジャイロ内蔵、イヤホン側で処理 |
| Dolby | Dolby Audio 対応。Dolby モードはヘッドトラッキング非対応 |
| バッテリー | 約 8.5 時間（ANC OFF、イヤホン単体）、ケース併用で最大約 35 時間 |
| 急速充電 | 5 分で約 2 時間 |
| 防塵防水 | IP54 |
| その他 | Google Fast Pair、デュアルデバイス接続 |

補足:

- LDAC 96kHz 以上では空間オーディオが無効になる（Xiaomi FAQ）。
- 「ヘッドトラッキングは Xiaomi dimensional audio のみ」で、
  Dolby モードでは使えない。

## 観測した個体

| 項目 | 内容 |
|---|---|
| 名前 | REDMI Buds 8 Pro |
| Bluetooth Class | `0x001F00` |
| デバイスタイプ | DUAL (BR/EDR + LE) |
| ファームウェア | 3.2.2.6 |
| Fast Pair hardware version | `FAST_PAIR_11728A` |
| Fast Pair companion app | `com.mi.earphone` |
| Fast Pair device type | Untethered Headset |
| MAC | 伏せる |

Fast Pair の custom metadata では L/R のバッテリーとケースの充電状態も
取得できていた。

## 接続プロファイル（観測）

`dumpsys bluetooth_manager` の bonded device 情報より:

- A2DP（earbuds 側は Audio Sink）
- HFP（earbuds 側は Hands-Free）
- AVRCP Controller（earbuds 側は Target）
- PBAP Client（policy は -1 で未使用）
- HID Host
- SPP

SDP で見えた UUID:

| UUID | 推定される用途 |
|---|---|
| `0000110b` | Audio Sink (A2DP) |
| `0000110e` | A/V Remote Control Target |
| `0000111e` | Hands-Free |
| `00001124` | HID |
| `0000fd2d-0000-1000-8000-00805f9b34fb` | Xiaomi 独自 SPP |
| `df21fe2c-2515-4fdb-8886-f12c4d67927c` | Google Fast Pair Message Stream |

## RFCOMM チャンネル

| Channel | UUID | 用途 |
|---|---|---|
| 17 | `df21fe2c-...` | Fast Pair Message Stream |
| 28 | `0000fd2d-...` | Xiaomi 独自 SPP（設定・状態） |

HFP 用の RFCOMM チャンネルも別途開設される。

## HID

- HID Host として接続している。
- L2CAP PSM: `0x0011` (HID-Control), `0x0013` (HID-Interrupt)
- `dumpsys bluetooth_manager` では `HID connection state=2`（CONNECTED）、
  `HOGP connection state=0`。
- feature report 6 の GET_REPORT / SET_REPORT を観測。
- `libspatial.so` 内に `#AndroidHeadTracker#1.0`、
  `MMA-BLUETOOTH-DEVICE`、`UHID_*` の文字列がある。

## Android センサー

`dumpsys sensorservice` で以下が見える。

```text
0x01000003) REDMI Buds 8 Pro | Bluetooth 02b0:02b0 | ver: 16777216
  type: android.sensor.head_tracker(37)
  continuous | minRate=10.00Hz | maxRate=100.00Hz
  no batching | non-wakeUp | dynamic
```

- `TYPE_HEAD_TRACKER` = 37
- 動的センサー（dynamic sensor）
- vendor は `Bluetooth 02b0:02b0`
- `getDefaultSensor(37)` では取得できない。
  `getDynamicSensorList(TYPE_ALL)` を使う。
- デフォルトでは system / audioserver 以外からは見えない。

## LE について

今回の HCI ログでは、イヤホンとの LE ACL / GATT 接続は観測されなかった。
周囲の BLE アドバタイズは観測されたが、ヘッドトラッキングは
Classic の HID / SPP 側でやり取りされている可能性が高い。

## 参考

- Xiaomi Global: REDMI Buds 8 Pro 製品ページ
  <https://www.mi.com/global/product/redmi-buds-8-pro/>
- Xiaomi Global: REDMI Buds 8 Pro FAQ
  <https://www.mi.com/global/support/faq/details/KA-666754/>
- Gadgets360: Redmi Buds 8 Pro Launched ...
  <https://www.gadgets360.com/audio/news/redmi-buds-8-pro-price-launch-availability-specifications-features-10908324>
- Gizmochina: Redmi Buds 8 Pro debuts ...
  <https://www.gizmochina.com/2026/01/29/redmi-buds-8-pro-launched-specs-price/>
- PR TIMES: REDMI Buds 8 Pro 日本発売
  <https://prtimes.jp/main/html/rd/p/000000213.000098799.html>
