# Redmi Buds 8 Pro 調査メモ

2026-10-29 時点の調査結果。Redmi Buds 8 Pro のヘッドトラッキング / IMU に
Android からアクセスできるかを調べたときの記録。

## 対象

| 項目 | 内容 |
|---|---|
| イヤホン | REDMI Buds 8 Pro |
| ファームウェア | 3.2.2.6（Xiaomi Earbuds アプリ表示） |
| ペアリング先 | Xiaomi 17T Pro (`2602EPTC0R`) |
| Android | 16 (BP2A.250605.031.A3) |
| アプリ | `com.mi.earphone` 1.38.0i |
| Bluetooth 拡張 | `com.xiaomi.bluetooth` (`/system_ext/app/BluetoothExtension`) |

個体の Bluetooth MAC アドレスは伏せている。

## 結論

- ヘッドトラッキングは **Android の動的センサー `android.sensor.head_tracker`
  （`TYPE_HEAD_TRACKER` = 37）として公開されている**。
- ただしデフォルトでは system / audioserver だけがアクセスできる。
  `adb shell cmd sensorservice unrestrict-ht` でテスト用に制限を外せる。
- 取得できるのは **イヤホン側で算出された姿勢（回転ベクトル）と角速度**。
  生の加速度・ジャイロ値（raw IMU）は公開されていない。
- 今回のセッションではセンサーの発見・登録までは成功したが、
  **実データ（イベント）は 0 件**だった。データが流れる条件は未確定。

## ドキュメント

| ファイル | 内容 |
|---|---|
| [device.md](device.md) | 製品仕様、観測した Bluetooth プロファイル、UUID、センサー情報 |
| [protocols.md](protocols.md) | Xiaomi SPP、Fast Pair message stream、HID head tracker のプロトコル |
| [android-head-tracker.md](android-head-tracker.md) | Android センサーとしてのアクセス方法と制限 |
| [investigation-log.md](investigation-log.md) | 調査手順、使用コマンド、証拠、未解決事項 |

## Status

| 項目 | 状態 |
|---|---|
| Classic Bluetooth 接続 | 確認済み |
| Xiaomi SPP (RFCOMM ch28) | 確認済み |
| Fast Pair message stream (RFCOMM ch17) | 確認済み |
| HID head tracker の存在 | 確認済み |
| Android `TYPE_HEAD_TRACKER` センサーの存在 | 確認済み |
| `unrestrict-ht` 後のセンサー列挙・登録 | 確認済み |
| ヘッドトラッキング実データの受信 | **未確認（0 events）** |
| 生 IMU (accel/gyro) の取得 | 不可（公開インターフェースなし） |

## 注意

- HCI snoop log、bugreport、APK には MAC アドレスや利用状況が含まれる。
  リポジトリにはコミットしない。
- `unrestrict-ht` はテスト用のフラグ。任意のアプリが頭の動きを読める状態に
  なるため、常用しない。再起動で戻る可能性がある。
- 本ドキュメントのプロトコル解析は相互運用目的の観測記録であり、
  ファームウェアやアプリの再配布を意図したものではない。
