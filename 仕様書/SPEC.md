# ボアホール監視システム 仕様書（完成版・改訂反映）

## 1. 目的
本システムは Raspberry Pi 5 上で動作するボアホール向け監視・記録システムであり、以下の機能を統合的に提供する。

- 広角カメラ（2系統）による静止画・動画の取得
- LED（2系統）のPWM制御
- 磁気センサによる方位角計測
- 単点LiDARによる距離計測
- 外気／筐体内の温湿度計測
- 露点温度算出および結露リスク判定
- Web UI による可視化・操作

本仕様書は、設計・実装・運用における単一の正（Single Source of Truth）とする。

---

## 2. システム構成

- camera_service：カメラ制御、静止画・動画取得、LED制御、UI配信
- sensor_service：センサ取得、集計、CSV記録、SSE配信、判定ロジック

---

## 3. ディレクトリ構造と保存方針

実行時生成物はすべて `data/` 配下に保存する。

- 静止画：`data/cam/`
- 動画：`data/video/`
- センサCSV：`data/csv/`

---

## 4. カメラ仕様

### 4.1 構成
- Raspberry Pi Camera Module V3（広角）×2
- cam0 / cam1 は CSI ポート若番順

### 4.2 静止画
- JPEG形式
- EXIF（GPSImgDirectionRef="T", GPSImgDirection）に撮影方位（真方位）を記録
- 方位は磁気偏角・カメラオフセット補正後の値を用いる

### 4.3 動画
- MP4（H.264）
- 方位情報は動画ファイルに埋め込まず、同一ディレクトリの `meta.json` に記録する

---

## 5. センサ仕様

| センサ | UI反映周期 | 集計 |
|---|---|---|
| 磁気 | 0.5秒 | 中央値 |
| 温湿度 | 1.0秒 | 中央値 |
| 距離 | 1.0秒 | 中央値 |

距離値には有効性を示すステータスを付与する。

- VALID：有効データが十分に存在
- NO_DATA：有効データ不足

---

## 6. 結露リスク判定

外気露点温度を Magnus 式で算出し、以下で判定する。

ΔT = 内気温 − 外気露点温度

| 状態 | 条件 |
|---|---|
| SAFE | ΔT ≥ safe_delta_c |
| CAUTION | caution_delta_c ≤ ΔT < safe_delta_c |
| WARNING | condensing_delta_c < ΔT < caution_delta_c |
| CONDENSING | ΔT ≤ condensing_delta_c |

閾値は設定ファイルで変更可能。

---

## 7. 方位補正と記録

- 磁気偏角（declination）を設定ファイルで指定
- カメラごとの取付オフセットを設定ファイルで指定
- 真方位 = 磁気方位 + 偏角 + オフセット

---

## 8. CSVログ
- sensor_service が記録
- UI表示値と同一の中央値を記録

---

## 9. 備考
- 設定は `config.yaml` に集約する
- UI仕様は別紙「UI仕様書」に定義する
