# プロトコルメモ

Redmi Buds 8 Pro と Android の間で観測したプロトコル。
断りがない限り、HCI snoop log とアプリの逆コンパイル結果に基づく。

## 全体像

```text
Android
  ├─ A2DP / HFP / AVRCP       音声・通話
  ├─ HID Host                 head tracker (vendor HID)
  ├─ RFCOMM ch17              Fast Pair Message Stream
  └─ RFCOMM ch28              Xiaomi 独自 SPP（設定・状態）
```

ヘッドトラッキングは **HID**、設定は **Xiaomi SPP** に分かれている。
IMU の生データを流すチャンネルは見つかっていない。

## Xiaomi 独自 SPP

### 接続情報

- RFCOMM channel: **28**
- UUID: `0000fd2d-0000-1000-8000-00805f9b34fb`
- SDP 上の名前: `Xiaomi Inc.`

### フレーム形式（観測）

```text
FE DC BA | class (1) | command id (2, little-endian) | length (1) | seq (1) | payload | EF
```

- `FE DC BA`: 固定ヘッダ
- `class`: `C1` / `C3` / `C4` / `C7` などを観測
- `command id`: 2 bytes。リトルエンディアンと推定
- `length`: `seq` + payload のバイト数
- `seq`: パケットごとに増えるシーケンス番号
- `EF`: 固定ターミネータ

観測例:

```text
FE DC BA C4 02 00 05 06 FF FF FF FF EF
  header  class id=0x0002 len=5 seq=6 payload=FF FF FF FF  end

FE DC BA C1 F3 00 19 0B 00 01 00 02 00 03 00 04 00 0A 00 0B
          00 0F 00 0D 00 25 00 1E 00 23 00 27 EF
  header  class id=0x00F3 len=0x19 seq=0x0B
  payload = 機能 ID のリスト（0x0001, 0x0002, ...）
```

`C1 F3 00` の payload は 16-bit の機能 ID リストに見える。
この個体では `0001, 0002, 0003, 0004, 000A, 000B, 000F, 000D,
0025, 001E, 0023, 0027` が並んでいた。

### ANC 切り替えの例（外部解析）

公開されている Redmi Buds 5 の解析例では、ANC 切り替えは次の形式。

```text
FE DC BA C7 0E 00 04 09 02 04 01 EF
```

`02 04 01` は TLV と解釈でき、

- `02`: 後続長
- `04`: 機能コード（ANC モード）
- `01`: 値（01=ANC, 00=OFF, 03=透過）

と説明されている。Redmi Buds 8 Pro でも同じ `FE DC BA` フレームを
確認しているが、ANC の機能コードが同一かは未検証。

### 機能 ID（アプリ内定数）

`com.mi.earphone` の `Function.java` に定義されている ID の一部。
UI の機能表示や設定項目の識別に使われる。

| ID | 名前 |
|---|---|
| 1002 | `FUNC_ANC_NOISE_THREE` |
| 1003 | `FUNC_ANC_NOISE_SIX` |
| 1004 | `FUNC_ANC_TRANSPARENT_TWO` |
| 1005 | `FUNC_ANC_TRANSPARENT_THREE` |
| 1006 | `FUNC_ANC_NOISE_FOUR` |
| 1007 | `FUNC_ANC_NOISE_TWO` |
| 1008 | `FUNC_SMART_NOISE_REDUCTION` |
| 1009 | `FUNC_NOISE_IMMERSE` |
| 2001 | `FUNC_SOUND_SETTING_VIRTUAL_SURROUND` |
| 2002 | `FUNC_SOUND_SETTING_CLOSE_SPATIAL_AUDIO` |
| 2003 | `FUNC_SOUND_SETTING_SPATIAL_AUDIO_PREFERENCE` |
| **2004** | **`FUNC_SOUND_SETTING_SPATIAL_AUDIO_HEAD_TRACKING`** |
| 2005 | `FUNC_SOUND_SETTING_SPATIAL_AUDIO_VIRTUAL_SURROUND` |
| 2006 | `FUNC_SOUND_SETTINGS_SOUND_MODE` |
| 2007 | `FUNC_SOUND_SETTINGS_SPATIAL_AUDIO_NOTIFY` |
| 2008 | `FUNC_SOUND_SETTINGS_ADAPTIVE_SENSE` |
| 2009 | `FUNC_SOUND_EFFECT_RENDERING` |
| 2016 | `FUNC_SOUND_SETTINGS_SOUND_MODE_OTA` |
| 2017 | `FUNC_SOUND_SETTINGS_ADAPTIVE_VOLUME` |
| 2018 | `FUNC_PERSONAL_SPATIAL_AUDIO` |
| 2019 | `FUNC_AUDIBILITY_ADAPTATION` |
| 2020 | `FUNC_AUDIO_MODE` |
| 2021 | `FUNC_NOTIFICATION_VOLUME` |
| 3001 | `FUNC_SMART_FREE_PICK` |
| 3002 | `FUNC_WEAR_DETECTION` |
| 3003 | `FUNC_FIT_DETECT` |
| 3004 | `FUNC_DUAL_DEVICE_CONNECTION` |
| 3005 | `FUNC_VOICE_CONTROL` |
| 3006 | `FUNC_HEARING_HEALTH` |
| 3007 | `FUNC_NOTIFICATION` |
| 3008 | `FUNC_AUTO_PICK_CALL` |
| 3009 | `FUNC_AUTO_GAMING_MODE` |
| 3011 | `FUNC_LOW_LATENCY` |
| 3012 | `FUNC_PERSONALIZED_NOISE_REDUCTION` |
| 3013 | `FUNC_EAR_CANAL_DETECTION` |
| 3014 | `FUNC_NOTIFICATION_TTS` |
| 3015 | `FUNC_EARBOX_SOUND` |
| 4001-4006, 4011-4018 | ジェスチャー設定 |
| 5001 | `FUNC_FIND_DEVICE` |
| 5004 | `FUNC_FIRMWARE_UPDATE` |
| 6001 | `FUNC_LABORATORY_NOISE` |
| 6002 | `FUNC_LABORATORY_WEAR_DETECTION` |
| 7001-7005 | USB / dongle 系 |
| 8001 | `FUNC_SWIM_LENGTH` |

### 空間オーディオ設定

UI 上のヘッドトラッキングは機能 ID **2004** でゲートされるが、
実際の設定コマンドは `DeviceConfigSpatialAudio` で、config ID は **29 (0x1D)**。

アプリの `DeviceConfigSpatialAudio` は 1 byte のビットフィールドを送る。
逆コンパイル結果から読み取れる配置:

| bit | 内容 |
|---|---|
| 7 | spatial audio open |
| 6-5 | preference (0=高音質, 1=低遅延) |
| 4 | head tracking |
| 3 | virtual surround |
| 2-0 | 未使用（0） |

`paramsToValue()` は

```text
virtualSurround + headTracking + preference(2bit) + spatialAudioOpen
```

を結合して 1 byte にしている。`valueToParams()` は上記の位置で読み戻す。

この設定は「イヤホン側の空間オーディオを ON/OFF し、
ヘッドトラッキングを有効にする」ためのコマンドであり、
IMU の生データを要求するものではない。

## Fast Pair Message Stream

- RFCOMM channel: **17**
- UUID: `df21fe2c-2515-4fdb-8886-f12c4d67927c`
- 形式（Google Fast Pair 仕様）:

```text
group (1) | code (1) | additional data length (2, big-endian) | data
```

観測例:

```text
03 08 00 02 01 24
03 01 00 03 11 72 8A
03 03 00 03 5A 5F FF   （周期的に再送）
```

`03 03 00 03 5A 5F FF` は接続維持や状態通知に見えるが、
ヘッドトラッキングのストリームではない。

## HID Head Tracker

### Android 側の標準プロトコル

AOSP の Head tracker HID protocol（Android 13+）では、
HID のアプリケーションコレクションとして以下を公開する。

- Sensor Description: `#AndroidHeadTracker#1.0`
- Reporting State（read/write）
- Power State（read/write）
- Report Interval（read/write）
- Input report:
  - Custom Value 1: 姿勢（回転ベクトル、rad）
  - Custom Value 2: 角速度（rad/s）
  - Custom Value 3: 参照フレームのリセットカウンタ

Android はこれを動的センサー `TYPE_HEAD_TRACKER` として公開する。

### この個体で観測したもの

- HID Host として接続（`HID connection state=2`）
- feature report **6** の GET_REPORT / SET_REPORT
- SET_REPORT の値: `0x1C`, `0x02`, `0x01`
- GET_REPORT の応答: `0x00`
- HID interrupt channel (CID `0x5240`) の input report は **0 件**

feature report 6 が Reporting State / Power State / Report Interval の
どれに相当するかは未確定。ベンダー独自のレポート ID の可能性がある。

### `libspatial.so`

`/system_ext/lib64/libspatial.so` に以下の JNI がある。

```text
Java_com_android_bluetooth_ble_app_headset_Spatialcontroller_createspatialhid
Java_com_android_bluetooth_ble_app_headset_Spatialcontroller_destroyspatialhid
Java_com_android_bluetooth_ble_app_headset_Spatialcontroller_sendspatialdata
```

文字列:

```text
#AndroidHeadTracker#1.0
MMA-BLUETOOTH-DEVICE
UHID_OPEN from uhid-dev
UHID_SET_REPORT: Report type = %d, report_size = %d report id = %d ...
UHID_GET_REPORT: ...
dynamic poll thread
```

`Spatialcontroller` は Java から

```java
createspatialhid(mac)
sendspatialdata(mac, float[3])
destroyspatialhid(mac)
```

を呼ぶ。`sendspatialdata` は 30ms 周期とみられる Timer から呼ばれている。

### データが流れない場合の候補

AOSP 仕様では、以下がすべて満たされないと input report は送られない。

1. Power State = Full Power
2. Reporting State = All Events
3. Report Interval != 0

この個体では feature report 6 に `0x1C`, `0x02`, `0x01` を書いていたが、
それが上記 3 条件を満たす値かは未確認。
また、イヤホンが装着されていない場合に止まる可能性もある。

## 取得できないもの

- 生の加速度（raw accelerometer）
- 生のジャイロ（raw gyroscope）
- 個別の IMU サンプル

Android のセンサー一覧にも `TYPE_ACCELEROMETER_LIMITED_AXES` や
`TYPE_GYROSCOPE_LIMITED_AXES` は出ていない。
Xiaomi Earbuds アプリのコードにも IMU をパースする処理は見当たらない。

## 未解決事項

- HID input report が流れる正確な条件
- feature report 6 のビット配置と正しい設定値
- HID レポート記述子の全体（report ID とフィールドの対応）
- `unrestrict-ht` が再起動後も保持されるか
- Dolby モード時に phone 側センサーを使うか
