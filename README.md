# TomTom Orbis Traffic API Demo

## 概要

本リポジトリは **TomTom Orbis Maps Traffic API** を利用し、

- 交通量（Flow）
- 事故・規制情報（Incidents）

を **Rasterタイル（PNG）** および **Vectorタイル（PBF）** で表示するデモアプリです。

地図描画には **MapLibre GL JS** を使用しています。  
ベースマップには国土地理院標準地図タイル（std）を使用しています。

---

## 本デモで実装している内容

- Flow（交通量）を Raster / Vector で切替表示
- Incidents（事故・規制）を Raster / Vector で切替表示
- 東京 / 大阪 / 福岡 への移動ボタン
- 移動中はTrafficレイヤーを停止（無駄なAPI取得防止）
- 到着地点でのみTrafficを取得
- APIエラー（403等）の確認

---

# 使用API

本デモでは **TomTom Orbis Maps Traffic API** を使用しています。

公式製品情報：
https://developer.tomtom.com/traffic-api/documentation/tomtom-orbis-maps/product-information/introduction

---

## 本デモで使用しているエンドポイント

---

### ● Flow (Raster)

/maps/orbis/traffic/tile/flow/{z}/{x}/{y}.png

公式ドキュメント：  
https://developer.tomtom.com/traffic-api/documentation/tomtom-orbis-maps-traffic-flow-raster-tile

---

### ● Flow (Vector)

/maps/orbis/traffic/tile/flow/{z}/{x}/{y}.pbf

公式ドキュメント：  
https://developer.tomtom.com/traffic-api/documentation/tomtom-orbis-maps-traffic-flow-vector-tile

---

### ● Incidents (Raster)

/maps/orbis/traffic/tile/incidents/{z}/{x}/{y}.png

公式ドキュメント：  
https://developer.tomtom.com/traffic-api/documentation/tomtom-orbis-maps-traffic-incidents-raster-tile

---

### ● Incidents (Vector)

/maps/orbis/traffic/tile/incidents/{z}/{x}/{y}.pbf

公式ドキュメント：  
https://developer.tomtom.com/traffic-api/documentation/tomtom-orbis-maps-traffic-incidents-vector-tile

---

## APIベースURL

https://api.tomtom.com

---

# 料金体系

TomTom Orbis Maps Traffic API は従量課金制です。

公式価格ページ：
https://developer.tomtom.com/pricing

---

## 課金単位

Traffic API は主に以下の単位で課金されます：

- **Rasterタイル取得回数**
- **Vectorタイル取得回数**
- JSON API呼び出し回数（incidentDetails等）

1タイル取得 ＝ 1リクエストとしてカウントされます。

ズームレベルが高いほど、表示領域内で取得するタイル枚数は増加します。

---

## 無料枠

TomTom Developer アカウントには  
**月間無料利用枠** が設定されています。

無料枠の詳細（常に公式で確認してください）：
https://developer.tomtom.com/pricing

---

## 重要な注意事項

- 地図表示中は MapLibre が自動でタイルを取得します
- パン・ズーム操作により取得枚数が増加します
- 高ズーム（15以上）ではタイル取得枚数が急増します
- API上限を超えると **403 Forbidden** が返却されます
- クレジットカード未登録の場合、自動課金は発生しませんが利用停止になります

---

## 技術スタック

- Vue 3
- MapLibre GL JS
- TomTom Orbis Maps Traffic API
- 国土地理院標準地図タイル

---

## 環境変数設定

`.env` に以下を設定してください：

VITE_TOMTOM_API_KEY=YOUR_API_KEY

※ `.env` ファイルはリポジトリに含めないでください。

---

## 想定用途

- API動作検証
- Raster / Vector 表示差異確認
- 課金挙動検証
- トラフィックタイル挙動調査

---

## 公式情報

TomTom Orbis Maps Traffic API  
https://developer.tomtom.com/traffic-api/documentation/tomtom-orbis-maps/product-information/introduction

TomTom Pricing  
https://developer.tomtom.com/pricing
