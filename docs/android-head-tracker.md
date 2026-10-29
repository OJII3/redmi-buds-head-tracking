# Android からヘッドトラッキングを読む

## 前提

Redmi Buds 8 Pro のヘッドトラッキングは、Android の動的センサー

```text
android.sensor.head_tracker (TYPE_HEAD_TRACKER = 37)
```

として公開される。センサー名は `REDMI Buds 8 Pro`。

ただし **デフォルトでは system / audioserver だけがアクセスできる**。
通常のアプリから使うには、テスト用に制限を外す必要がある。

## 制限の解除

```bash
adb shell cmd sensorservice unrestrict-ht
```

戻す場合:

```bash
adb shell cmd sensorservice restrict-ht
```

- `SensorService` の `mHtRestricted` を切り替えるテスト用コマンド。
- shell は `MANAGE_SENSORS` を持っているため実行できる。
- 再起動で戻る可能性がある。
- 解除中は任意のアプリが頭の動きを読める。常用しない。

確認:

```bash
adb shell dumpsys sensorservice | grep -A2 'REDMI Buds'
```

期待する表示:

```text
0x01000003) REDMI Buds 8 Pro | Bluetooth 02b0:02b0 | ver: 16777216
  type: android.sensor.head_tracker(37)
  continuous | minRate=10.00Hz | maxRate=100.00Hz | dynamic
```

## センサーの列挙

`getDefaultSensor(37)` では取れない。
動的センサーなので `getDynamicSensorList()` を使う。

```java
SensorManager sm = (SensorManager) context.getSystemService(Context.SENSOR_SERVICE);

Sensor target = null;
for (Sensor s : sm.getDynamicSensorList(Sensor.TYPE_ALL)) {
    if (s.getType() == 37) {        // TYPE_HEAD_TRACKER
        target = s;
    }
}
```

## 値

`SensorEvent.values` のレイアウト（Android 公式ドキュメント）:

| index | 内容 |
|---|---|
| 0 | 回転ベクトル X（rad） |
| 1 | 回転ベクトル Y（rad） |
| 2 | 回転ベクトル Z（rad） |
| 3 | 角速度 X（rad/s） |
| 4 | 角速度 Y（rad/s） |
| 5 | 角速度 Z（rad/s） |

- クォータニオンではなく **回転ベクトル（軸角）**。
- 回転ベクトルの大きさは `[0, π]` rad。
- 基準フレームは任意。磁北基準の絶対方位ではない。
- ドリフトは許容される。
- `SensorEvent.firstEventAfterDiscontinuity` が true のときは、
  参照フレームがリセットされた可能性がある。

クォータニオンが必要な場合:

```java
float x = v[0], y = v[1], z = v[2];
float angle = (float) Math.sqrt(x*x + y*y + z*z);
if (angle < 1e-6f) {
    // identity
} else {
    float s = (float) Math.sin(angle / 2) / angle;
    float qx = x * s, qy = y * s, qz = z * s;
    float qw = (float) Math.cos(angle / 2);
}
```

## 最小コード

`app_process` で動かすプロトタイプ。実運用は APK にする。

```java
import android.content.Context;
import android.hardware.*;
import android.os.Looper;

public class HeadTrackerReader {
    public static void main(String[] args) throws Exception {
        Looper.prepareMainLooper();

        Class<?> at = Class.forName("android.app.ActivityThread");
        java.lang.reflect.Method systemMain = at.getDeclaredMethod("systemMain");
        systemMain.setAccessible(true);
        Object thread = systemMain.invoke(null);
        java.lang.reflect.Method getSystemContext =
            at.getDeclaredMethod("getSystemContext");
        getSystemContext.setAccessible(true);
        Context ctx = (Context) getSystemContext.invoke(thread);

        SensorManager sm =
            (SensorManager) ctx.getSystemService(Context.SENSOR_SERVICE);

        Sensor target = null;
        for (Sensor s : sm.getDynamicSensorList(Sensor.TYPE_ALL)) {
            System.out.println("DYN " + s.getName() + " type=" + s.getType());
            if (s.getType() == 37) target = s;
        }
        if (target == null) {
            System.out.println("NO_HEAD_TRACKER");
            return;
        }

        sm.registerListener(new SensorEventListener() {
            public void onSensorChanged(SensorEvent e) {
                System.out.println("EVT " + e.timestamp + " "
                    + e.values[0] + " " + e.values[1] + " " + e.values[2] + " "
                    + e.values[3] + " " + e.values[4] + " " + e.values[5]);
            }
            public void onAccuracyChanged(Sensor s, int a) {}
        }, target, 10000); // 100Hz

        Looper.loop();
    }
}
```

ビルド例:

```bash
# android.jar は platform 36、d8 は build-tools 37 のもの
javac -source 8 -target 8 -bootclasspath android.jar -d classes HeadTrackerReader.java
d8 --lib android.jar --min-api 26 --output . classes/*.class
adb push classes.dex /data/local/tmp/ht.dex
adb shell 'CLASSPATH=/data/local/tmp/ht.dex app_process /system/bin HeadTrackerReader'
```

## 実測状況

2026-10-29 のセッション:

```text
DYN name=REDMI Buds 8 Pro type=37 vendor=Bluetooth 02b0:02b0
FOUND REDMI Buds 8 Pro minDelay=10000
REGISTER true
EVT count = 0
```

同じプログラムでスマホ本体の加速度センサーは取れているため、
読み取り側の問題ではない。

```text
ACC_EVT 4011852789431 0.13005 0.55695003 9.805051
ACC_EVT 4011872748721 0.132 0.55395 9.81
```

空間オーディオ側のログも:

```text
SpatialAudioFunction: isSpatialAudioOpen=true,isHeadTrackingOpen=true
MisoundSA: ... virtualization=1, head track angle=-0.00
```

つまり「設定は ON だが、姿勢データが phone まで来ていない」状態。
センサーの存在と登録は確認できたが、実データの受信は未確認。

## データが来ないときに確認すること

1. イヤホンを装着しているか（in-ear detection）
2. Xiaomi Earbuds アプリで
   `Audio effects → Dimensional audio` を開き、
   `Immersive sound` と `Track head movement` が両方 ON か
3. 音声コーデックが LDAC 96kHz になっていないか
   （96kHz 以上では空間オーディオ無効）
4. HID の Reporting State が All Events になっているか
5. HCI ログで HID interrupt channel (CID `0x5240`) に input report が
   出ているか

## 制約

- 取得できるのはイヤホン側で算出された姿勢と角速度のみ。
- 生の加速度・ジャイロは取れない。
- センサーは non-wakeup。画面 OFF 中の連続取得には向かない。
- `unrestrict-ht` はテスト用。製品アプリで依存しない。
