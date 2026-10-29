# 調査ログ

2026-10-29 実施。Redmi Buds 8 Pro のヘッドトラッキング / IMU に
Android からアクセスできるかを調べた記録。

## 環境

| 項目 | 内容 |
|---|---|
| ホスト | NixOS / flake devShell (`adb`, `uv`) |
| 端末 | Xiaomi 17T Pro (`2602EPTC0R`) |
| Android | 16 (SDK 36) |
| イヤホン | REDMI Buds 8 Pro |
| ファームウェア | 3.2.2.6 |
| アプリ | `com.mi.earphone` 1.38.0i |

## 1. adb 接続

端末が `no permissions` だったため、Xiaomi の USB Vendor ID `2717` を
udev に追加した。

```bash
sudo tee /etc/udev/rules.d/51-android.rules >/dev/null <<'EOF'
SUBSYSTEM=="usb", ATTR{idVendor}=="2717", MODE="0666", GROUP="plugdev"
EOF
sudo udevadm control --reload-rules
sudo udevadm trigger --subsystem-match=usb --action=add
```

端末側で USB デバッグを許可。

## 2. Bluetooth 状態の確認

```bash
adb shell dumpsys bluetooth_manager
adb shell dumpsys audio
adb shell dumpsys sensorservice
```

確認できたこと:

- `REDMI Buds 8 Pro` は DUAL、Class `0x001F00`
- A2DP / HFP / AVRCP / HID Host / SPP
- SDP UUID: `110b`, `110e`, `111e`, `11124`, `0000fd2d`, `df21fe2c`
- RFCOMM ch17 (Fast Pair), ch28 (Xiaomi)
- HID Host 接続 (`HID connection state=2`)
- 動的センサー `android.sensor.head_tracker(37)`

## 3. HCI snoop log の有効化

MIUI では `settings put global bluetooth_hci_log 1` が
`WRITE_SECURE_SETTINGS` 不足で失敗したため、開発者オプションの UI から
`Bluetooth HCI snoop log` を `Enabled` にした。

その後 Bluetooth を OFF/ON してログを開始。

```bash
adb shell svc bluetooth disable
adb shell svc bluetooth enable
```

## 4. HCI ログの取得

```bash
adb bugreport /tmp/bugreport
unzip -j /tmp/bugreport.zip \
  FS/data/misc/bluetooth/logs/btsnoop_hci.log
```

解析:

```bash
tshark -r btsnoop_hci.log -q -z io,phs
tshark -r btsnoop_hci.log -Y 'btsdp' -T fields \
  -e frame.number -e bthci_acl.src.bd_addr -e _ws.col.Info
tshark -r btsnoop_hci.log -Y 'data && btrfcomm.channel==28' \
  -T fields -e frame.number -e bthci_acl.src.bd_addr -e data.data
tshark -r btsnoop_hci.log -Y 'btl2cap.psm==0x0011 || btl2cap.psm==0x0013' \
  -T fields -e frame.number -e btl2cap.cid -e bthid.transaction_type -e data.data
```

分かったこと:

- Xiaomi SPP は RFCOMM ch28、`FE DC BA ... EF` フレーム
- Fast Pair Message Stream は RFCOMM ch17
- HID は PSM `0x0011` / `0x0013`、feature report 6
- HID interrupt channel の input report は 0 件

## 5. アプリの解析

APK を取得して jadx で逆コンパイル。

```bash
adb shell pm path com.mi.earphone
adb pull /data/app/.../base.apk earphone.apk
jadx -d out --no-res --no-imports classes*.dex
```

確認したクラス:

- `com.mi.earphone.settings.ui.spatialaudio.SpatialAudioVM`
- `com.mi.earphone.bluetoothsdk.setting.function.DeviceConfigSpatialAudio`
- `com.mi.earphone.device.manager.export.Function`

分かったこと:

- ヘッドトラッキング UI は `FUNC_SOUND_SETTING_SPATIAL_AUDIO_HEAD_TRACKING = 2004`
- 設定コマンドは `DeviceConfigSpatialAudio`（config ID 29）
- 1 byte のビットフィールドで spatial audio / head tracking を設定
- IMU データを受信・パースするコードは見当たらない

## 6. BluetoothExtension の解析

`/system_ext/app/BluetoothExtension/BluetoothExtension.apk` を取得。

```bash
adb pull /system_ext/app/BluetoothExtension/BluetoothExtension.apk
jadx -d out --no-res --no-imports classes.dex
```

確認したクラス:

- `com.android.bluetooth.ble.app.headset.Spatialcontroller`
- `com.android.bluetooth.ble.app.headset.V` / `T`
- `com.android.bluetooth.ble.app.broadcast.BluetoothPluginHelperService`

`Spatialcontroller`:

```java
public native boolean createspatialhid(String mac);
public native boolean destroyspatialhid(String mac);
public native void sendspatialdata(String mac, float[] data);
```

`/system_ext/lib64/libspatial.so` も取得。

```bash
adb pull /system_ext/lib64/libspatial.so
strings libspatial.so
```

文字列:

```text
#AndroidHeadTracker#1.0
MMA-BLUETOOTH-DEVICE
UHID_OPEN from uhid-dev
UHID_SET_REPORT
UHID_GET_REPORT
dynamic poll thread
```

AOSP の Head tracker HID protocol を実装しているとみられる。

## 7. Android センサーの読み取り

最初は `getDynamicSensorList()` が空だった。
`SensorService` の `mHtRestricted` が true で、system / audioserver 以外には
見えないため。

```bash
adb shell cmd sensorservice unrestrict-ht
```

その後、`app_process` で以下を確認。

```text
DYN name=REDMI Buds 8 Pro type=37 vendor=Bluetooth 02b0:02b0
FOUND REDMI Buds 8 Pro minDelay=10000
REGISTER true
```

ただしイベントは 0 件。

```text
EVT count = 0
```

同じプログラムで加速度センサーは取れている。

```text
ACC_EVT 4011852789431 0.13005 0.55695003 9.805051
```

空間オーディオのログ:

```text
SpatialAudioFunction: isSpatialAudioOpen=true,isHeadTrackingOpen=true
MisoundSA: ... virtualization=1, head track angle=-0.00
```

## 8. 検証できたこと / できなかったこと

### できた

- イヤホンの Bluetooth プロファイルと UUID の特定
- Xiaomi SPP ch28 のフレーム形式の観測
- Fast Pair Message Stream ch17 の観測
- HID head tracker の存在確認
- Android `TYPE_HEAD_TRACKER` センサーの列挙
- `unrestrict-ht` 後のセンサー登録
- アプリ内の空間オーディオ設定コマンドの特定

### できなかった

- ヘッドトラッキングの実データ受信
- HID input report の観測
- 生 IMU (accel/gyro) の取得
- feature report 6 の正確なビット配置の特定

## 9. 環境に加えた変更

| 変更 | 戻し方 |
|---|---|
| `/etc/udev/rules.d/51-android.rules` 追加 | ファイル削除 + `udevadm control --reload-rules` |
| 開発者オプション `USB debugging (Security settings)` ON | UI で OFF |
| 開発者オプション `Bluetooth HCI snoop log` Enabled | UI で Disabled |
| `cmd sensorservice unrestrict-ht` | `cmd sensorservice restrict-ht` |

## 10. 次のステップ候補

1. イヤホン装着・アプリ前景・音楽再生中に、HID input report が出るか
   HCI ログで確認する。
2. feature report 6 の値を変えて、input report が開始するか試す。
3. HID レポート記述子を取得して、report ID とフィールドを確定する。
4. それでも出ない場合、イヤホン側ファームウェアの仕様と判断する。
5. 生 IMU が必要なら、HID 以外のデバッグチャンネルを探す。
