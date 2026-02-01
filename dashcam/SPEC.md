# Dashcam 仕様書

## 目的
Raspberry Pi 5 上で
・Raspberry Pi カメラモジュール V3（広角）×2のプレビューとパラメーター調整/撮影
・２系統あるLEDのPWM制御
・磁気センサ(Qwiic Micro - MMC5983MA搭載 磁気センサ)の方位角計測を可視化　×1
・距離センサ(シングルポイントLiDAR TSD10)情報の可視化　×1
・温度センサーによる外気温と筐体内温度の可視化(SHT31使用 高精度温湿度センサーモジュールキット)×2
を提供する。

## システム構成
-`UI_service`
  - UIの配信　HTML/CSS/JS で構成。`sensor_service` のSSEに接続し、方位角/距離情報を表示。
  - 主要依存: Picamera2, OpenCV, FastAPI, libgpiod(gpioset)。
- `camera_service`
  - 役割: カメラプレビュー(MJPEG)、静止画撮影、LED制御、
  - 主要依存: Picamera2, OpenCV, FastAPI, libgpiod(gpioset)。
- `sensor_service`
  - 役割1: MMC5983MA 磁気センサの方位角取得、UI向けSSE配信。
  - 役割2: MMC5983MA 磁気センサの方位角取得、UI向けSSE配信。
  - 主要依存1: smbus2, FastAPI。【F:dashcam/sensor_service/app.py†L1-L259】

## ディレクトリ構成
- `dashcam/camera_service/`
  - `app.py`: カメラ/LED/撮影APIとUI配信。
  - `static/index.html`: ダッシュボードUI。
  - `static/ui.css`: UIスタイル。
  - `static/ui.js`: UIロジック。
- `dashcam/sensor_service/`
  - `app.py`: センサ計測とSSE配信API。【F:dashcam/camera_service/app.py†L1-L185】【F:dashcam/camera_service/static/index.html†L1-L153】【F:dashcam/camera_service/static/ui.css†L1-L230】【F:dashcam/camera_service/static/ui.js†L1-L259】【F:dashcam/sensor_service/app.py†L1-L259】

## camera_service 仕様

### 起動・ライフサイクル
- 起動時に `cam0` と `cam1` を開始し、シャットダウン時に停止する。
- LED制御用の `gpioset` プロセスはシャットダウン時に停止する。【F:dashcam/camera_service/app.py†L90-L138】【F:dashcam/camera_service/app.py†L144-L185】

### カメラストリーム
- 2台のカメラをMJPEGで配信。
- プレビュー設定: 1536x864 RGB、15fps、JPEG品質80。静止画はフル解像度で取得。
- `CameraStream` が最新JPEGを保持し、`/stream/*` で `multipart/x-mixed-replace` 形式を返す。【F:dashcam/camera_service/app.py†L26-L121】【F:dashcam/camera_service/app.py†L144-L185】

### 静止画撮影
- `/api/capture/shot` にPOSTすると、2台分の静止画を同一セッションIDで保存。
- 保存先: `${CAM_STORE_DIR}/cam0/<session>/<index>.jpg` と `${CAM_STORE_DIR}/cam1/<session>/<index>.jpg`。
- セッションIDは `YYYYMMDD_HHMMSS` 形式。連番は6桁ゼロ埋め。
- `download` エンドポイントで保存画像を取得可能。【F:dashcam/camera_service/app.py†L28-L185】

### 露出補正(EV)
- `POST /api/cam/0/ev` と `POST /api/cam/1/ev` で `ExposureValue` を設定。
- 受信ボディ: `{ "ev": <float> }`。【F:dashcam/camera_service/app.py†L28-L175】

### LED 制御
- `GET /api/led`: 現在のLED状態、GPIOチップ名、ライン番号を返す。
- `POST /api/led`: `on` を指定してON/OFFを切替。
- `gpioset` プロセスを保持することでライン出力を維持する。エラー時は500を返す。【F:dashcam/camera_service/app.py†L71-L171】

### 静的UI配信
- `GET /` は `/ui/` へリダイレクト。
- `/ui` に `static/` をマウント。`/stream/*` を静的配信に吸われないための設計。【F:dashcam/camera_service/app.py†L112-L141】

### 環境変数
- `LED_GPIOCHIP` (既定: `gpiochip0`)
- `LED_LINE` (既定: `17`)
- `CAM_STORE_DIR` (既定: `/home/admin/dashcam/cam_store`)【F:dashcam/camera_service/app.py†L65-L122】

## sensor_service 仕様

### センサ取得
- MMC5983MA をI2Cで読み取る。
- 18bitのXYZデータから方位角を計算。
- 読み取りは `READ_HZ` で実行し、UI用には `AVG_WINDOW_SEC` の平均方位角を返す。【F:dashcam/sensor_service/app.py†L1-L214】

### API
- `GET /` : エンドポイント一覧をプレーンテキストで返す。
- `GET /health` : センサ取得状況を返す。
- `GET /api/sensors/latest` : 最新の平均方位角と距離情報をJSONで返す。
- `GET /api/sensors/stream` : SSEで `event: sensors` を送信。UIはこのストリームを使用。【F:dashcam/sensor_service/app.py†L1-L259】

### 出力ペイロード
```json
{
  "ts": "ISO8601(JST)",
  "heading_deg_avg": 0.0,
  "distance_mm_avg": null,
  "distance_status": "NO_DATA",
  "distance_valid_ratio": 0.0
}
```
【F:dashcam/sensor_service/app.py†L168-L259】

### CORS
- すべてのオリジン/メソッド/ヘッダを許可し、UIからのアクセスを許容する。【F:dashcam/sensor_service/app.py†L35-L53】

### 環境変数
- `MMC_I2C_BUS` (既定: `1`)
- `MMC_I2C_ADDR` (既定: `0x30`)
- `UI_HZ` (既定: `1.0`)
- `AVG_WINDOW_SEC` (既定: `1.0`)
- `MMC_READ_HZ` (既定: `20.0`)【F:dashcam/sensor_service/app.py†L20-L31】

## UI 仕様

### 画面構成
- カメラプレビュー2面 (CAM0/CAM1)。クリックで全画面表示。
- 撮影ボタンと「Last Shot」プレビュー、保存メタ情報の表示。
- LEDトグルスイッチ。
- EVスライダー(カメラ別)。
- センサHUD(方位角/距離)と距離のグラフ(1/5/15分切替)。【F:dashcam/camera_service/static/index.html†L1-L153】

### 動作仕様
- カメラ画像は `/stream/0` `/stream/1` へ `<img>` で接続し、読み込み失敗時はオーバーレイを表示。
- EV変更は200msデバウンスでAPIに送信。
- 撮影ボタン押下で `/api/capture/shot` を呼び出し、保存先にリンクを作成。
- LEDトグルは `GET /api/led` で初期状態を取得し、変更時に `POST /api/led` を実行。
- センサSSEは `sensor_service` を `:81` で起動する想定。UI側で `EventSource` を使い HUD/グラフ更新。
- 距離のグラフは直近最大16分のバッファを保持し、表示範囲を切替可能。【F:dashcam/camera_service/static/ui.js†L1-L259】

## データ保存
- 静止画は `CAM_STORE_DIR` 以下に保存される。
- セッションIDは初回撮影時に発行し、撮影回数に応じて連番が増加する。【F:dashcam/camera_service/app.py†L28-L185】

## 想定実行ポート
- `camera_service`: UIとカメラAPIを提供 (ポートは起動時設定に依存)。
- `sensor_service`: `ui.js` では `:81` を想定してSSE接続するため、同ポートでの起動が前提。【F:dashcam/camera_service/static/ui.js†L1-L259】
