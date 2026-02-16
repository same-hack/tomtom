<template>
  <div class="wrap">
    <!--
      ============================================
      操作用パネル（画面左上に重ねる）
      ============================================
      - 地図の移動（東京/大阪/福岡）
      - レイヤー表示切替（Base / Flow / Incidents を raster / vector で切替）
      - 公式ドキュメントへのリンク
    -->
    <div class="panel">
      <div class="row">
        <!--
          都市ボタン：
          - map.flyTo() で指定座標へ移動するだけ
          - flyTo完了後に incidentDetails(API) を1回叩いて console に結果を出す
            →「その地点でAPIが取れているか」を確認しやすい
        -->
        <button @click="flyToTokyo">東京</button>
        <button @click="flyToOsaka">大阪</button>
        <button @click="flyToFukuoka">福岡</button>

        <!--
          TomTom Orbis Maps Traffic API の公式ドキュメント。
          target="_blank" で別タブ、rel=... はセキュリティ対策（推奨）。
        -->
        <a
          href="https://developer.tomtom.com/traffic-api/documentation/tomtom-orbis-maps/product-information/introduction"
          target="_blank"
          rel="noopener noreferrer"
          class="official-link"
        >
          公式サイトへ
        </a>
      </div>

      <!--
        レイヤー表示切替：
        - MapLibre の layer の visibility を visible/none に切り替える
        - OFFにしても “APIが叩かれない” わけではなく、
          MapLibreが必要に応じてタイル取得を止めたり再開したりする
          （表示が消えるので結果として取得量が減ることはある）
      -->
      <div class="row">
        <label class="chk">
          <input type="checkbox" v-model="showBase" @change="applyVisibility" />
          Base
        </label>

        <label class="chk">
          <input
            type="checkbox"
            v-model="showFlowRaster"
            @change="applyVisibility"
          />
          Flow (raster)
        </label>

        <label class="chk">
          <input
            type="checkbox"
            v-model="showFlowVector"
            @change="applyVisibility"
          />
          Flow (vector)
        </label>

        <label class="chk">
          <input
            type="checkbox"
            v-model="showIncRaster"
            @change="applyVisibility"
          />
          Incidents (raster)
        </label>

        <label class="chk">
          <input
            type="checkbox"
            v-model="showIncVector"
            @change="applyVisibility"
          />
          Incidents (vector)
        </label>
      </div>
    </div>

    <!-- MapLibre の描画領域（ここに地図が表示される） -->
    <div ref="mapEl" class="map"></div>
  </div>
</template>

<script setup>
/**
 * このコンポーネントがやっていること（概要）
 * ------------------------------------------------------------
 * 1) MapLibre で地図を表示（styleは背景だけ、タイルは自前で追加）
 * 2) Base は国土地理院のラスタタイル（std）を表示
 * 3) Traffic は TomTom Orbis Maps Traffic API のタイルを表示
 *    - Flow: raster(PNG) / vector(PBF)
 *    - Incidents: raster(PNG) / vector(PBF)
 * 4) 追加で incidentDetails(JSON API) を必要に応じて叩き、consoleに出す
 *
 * 重要：このコードは “表示確認・調査用” です。
 * 本番のUI/UXは別途整える想定（ログ量/エラーハンドリング等）。
 */

import { onMounted, onBeforeUnmount, ref } from "vue";
import maplibregl from "maplibre-gl";

/* =========================================================
   1) 設定値（環境変数 / Orbisパラメータ）
   ========================================================= */

/**
 * TomTom API Key
 * - Vite の環境変数から読み込み（例：.env に VITE_TOMTOM_API_KEY=xxxx）
 * - これが無いと Orbis のタイルや incidentDetails が 401/403 になりやすい
 */
const key = import.meta.env.VITE_TOMTOM_API_KEY;

/**
 * TomTom Orbis Maps Traffic API のクエリパラメータ
 * - apiVersion: APIのバージョン
 * - style: rasterタイルの見た目（light/dark等）
 * - tileSize: ラスタタイルのピクセルサイズ（通常256）
 * - t: trafficModelId（-1は “最新” を意味する）
 */
const ORBIS_API_VERSION = 1;
const ORBIS_STYLE = "light";
const ORBIS_TILE_SIZE = 256;
const ORBIS_TRAFFIC_MODEL_ID = -1;

/* =========================================================
   2) Vue / MapLibre 状態
   ========================================================= */

/**
 * mapEl:
 * - template側の <div ref="mapEl"> にアクセスするためのref
 * map:
 * - MapLibre の Map インスタンス（onMountedで生成）
 */
const mapEl = ref(null);
let map = null;

/**
 * 表示ON/OFF（チェックボックス）
 * - “どのレイヤーを表示するか” を切り替えるだけのフラグ
 */
const showBase = ref(true);
const showFlowRaster = ref(true);
const showFlowVector = ref(false);
const showIncRaster = ref(true);
const showIncVector = ref(false);

/**
 * MapLibreの source/layer を識別するためのID
 * - 文字列は任意だが「一意」である必要がある
 * - “名前を揃える” と後から読みやすい
 */
const BASE = "base";
const FLOW_R = "flowRaster";
const FLOW_V = "flowVector";
const INC_R = "incRaster";
const INC_V = "incVector";

/* =========================================================
   3) タイルURL定義（どこから何を取るか）
   ========================================================= */

/**
 * Base（背景地図）：
 * - 国土地理院 標準地図（std）
 * - APIキー不要で使える（ただし利用規約・出典表記などは別途対応）
 * - “道路が細かく見えるベース” として、TomTomのbaseより分かりやすいことが多い
 */
function baseTiles() {
  return "https://cyberjapandata.gsi.go.jp/xyz/std/{z}/{x}/{y}.png";
}

/**
 * Flow raster（PNG）：
 * - 交通状況が “描画済みの画像” として返る
 * - 表示が手軽（スタイルはサーバ側で決まる）
 */
function flowRasterTiles() {
  return `https://api.tomtom.com/maps/orbis/traffic/tile/flow/{z}/{x}/{y}.png?apiVersion=${ORBIS_API_VERSION}&key=${key}&style=${ORBIS_STYLE}&tileSize=${ORBIS_TILE_SIZE}`;
}

/**
 * Incidents raster（PNG）：
 * - 事故/規制などが “描画済みの画像” として返る
 * - t=-1（最新モデル）を付けて最新に追従させる
 */
function incRasterTiles() {
  return `https://api.tomtom.com/maps/orbis/traffic/tile/incidents/{z}/{x}/{y}.png?apiVersion=${ORBIS_API_VERSION}&key=${key}&style=${ORBIS_STYLE}&t=${ORBIS_TRAFFIC_MODEL_ID}&tileSize=${ORBIS_TILE_SIZE}`;
}

/**
 * Flow vector（PBF）：
 * - Mapbox Vector Tile(PBF) 形式で “線データ(Feature)” が返る
 * - MapLibre側で自由に色分け/太さなどを設定できる
 */
function flowVectorTiles() {
  return `https://api.tomtom.com/maps/orbis/traffic/tile/flow/{z}/{x}/{y}.pbf?apiVersion=${ORBIS_API_VERSION}&key=${key}`;
}

/**
 * Incidents vector（PBF）：
 * - Mapbox Vector Tile(PBF) 形式で “点/線” のFeatureが返る
 * - t=-1 を付けて最新に追従
 */
function incVectorTiles() {
  return `https://api.tomtom.com/maps/orbis/traffic/tile/incidents/{z}/{x}/{y}.pbf?apiVersion=${ORBIS_API_VERSION}&key=${key}&t=${ORBIS_TRAFFIC_MODEL_ID}`;
}

/* =========================================================
   4) レイヤー表示制御（visibility切替）
   ========================================================= */

/**
 * setVisible:
 * - 指定レイヤーが存在する場合だけ visibility を変更する
 * - MapLibreは visibility="none" のレイヤーは描画しない
 */
function setVisible(id, v) {
  if (map.getLayer(id)) {
    map.setLayoutProperty(id, "visibility", v ? "visible" : "none");
  }
}

/**
 * applyVisibility:
 * - チェックボックスの状態をMapLibreレイヤーへ反映
 * - ここでは “状態をconsoleに出すだけ”
 */
function applyVisibility() {
  setVisible(BASE, showBase.value);

  setVisible(FLOW_R, showFlowRaster.value);
  setVisible(FLOW_V, showFlowVector.value);

  setVisible(INC_R, showIncRaster.value);
  // incidents vector は線と点が別レイヤーなので2つまとめてON/OFF
  setVisible("incLine", showIncVector.value);
  setVisible("incPoint", showIncVector.value);

  console.log("visibility:", {
    base: showBase.value,
    flowRaster: showFlowRaster.value,
    flowVector: showFlowVector.value,
    incRaster: showIncRaster.value,
    incVector: showIncVector.value,
  });
}

/* =========================================================
   5) JSON APIを叩いて結果をconsoleへ（incidentDetails）
   ========================================================= */

/**
 * logIncidentDetails:
 * - 現在表示している範囲（bbox）で incidentDetails を1回叩く
 * - 目的：タイル描画だけでなく、JSONでも “どれだけ事故があるか” を確認したい
 */
async function logIncidentDetails() {
  // bbox は (west,south,east,north) の順
  const b = map.getBounds();
  const bbox = `${b.getWest()},${b.getSouth()},${b.getEast()},${b.getNorth()}`;

  // incidentDetails（Orbis）
  const url =
    `https://api.tomtom.com/maps/orbis/traffic/incidentDetails` +
    `?apiVersion=${ORBIS_API_VERSION}` +
    `&key=${key}` +
    `&bbox=${bbox}` +
    `&timeValidityFilter=present` +
    `&t=${ORBIS_TRAFFIC_MODEL_ID}`;

  // ネットワーク時間を計測してログに出す（遅延調査に便利）
  const t0 = performance.now();
  const res = await fetch(url);
  const txt = await res.text();
  const ms = Math.round(performance.now() - t0);

  console.log("incidentDetails", {
    request: { bbox, url },
    response: { status: res.status, timeMs: ms, body: txt },
  });
}

/* =========================================================
   6) 都市移動（flyTo）
   ========================================================= */

/**
 * flyTo:
 * - 指定した中心座標へ移動し、移動完了(moveend)後に incidentDetails をログ出す
 * - 目的：手動で地図を動かさなくても、代表地点でAPIが取れるか確認できる
 */
async function flyTo(center, label) {
  console.log("flyTo:", label);

  // speedはアニメ速度。zoomは指定していないので現在のズームを維持する。
  map.flyTo({ center, speed: 1.2 });

  // flyTo完了（moveend）を待つ
  await new Promise((r) => map.once("moveend", r));

  console.log("arrived:", label);

  // 到着後のbboxでAPI確認
  await logIncidentDetails();
}

/**
 * 都市ボタン（駅付近の座標）
 * - 必要なら微調整してOK
 */
function flyToTokyo() {
  flyTo([139.767125, 35.681236], "Tokyo");
}
function flyToOsaka() {
  flyTo([135.498302, 34.702485], "Osaka");
}
function flyToFukuoka() {
  flyTo([130.4201, 33.5903], "Fukuoka");
}

/* =========================================================
   7) MapLibre 初期化（onMounted / onBeforeUnmount）
   ========================================================= */

onMounted(() => {
  // APIキー未設定は事故りやすいので警告（UI表示は不要なのでconsoleのみ）
  if (!key) {
    console.warn("VITE_TOMTOM_API_KEY が未設定です（.env を確認）");
  }

  // MapLibre Map生成
  // - style は背景だけ（sources/layersは load 後に addSource/addLayer する）
  map = new maplibregl.Map({
    container: mapEl.value,
    style: {
      version: 8,
      sources: {},
      layers: [
        {
          id: "bg",
          type: "background",
          paint: { "background-color": "#1d1d1d" },
        },
      ],
    },
    center: [139.767125, 35.681236], // 初期位置：東京
    zoom: 11,
  });

  /**
   * map.on("load"):
   * - MapLibreの初期ロード完了後に
   *   “タイルソース” と “レイヤー” を登録する
   */
  map.on("load", () => {
    // -------------------------
    // Base（国土地理院・ラスタ）
    // -------------------------
    map.addSource(BASE, {
      type: "raster",
      tiles: [baseTiles()],
      tileSize: 256,
    });
    map.addLayer({
      id: BASE,
      type: "raster",
      source: BASE,
    });

    // -------------------------
    // Flow raster（PNG）
    // -------------------------
    map.addSource(FLOW_R, {
      type: "raster",
      tiles: [flowRasterTiles()],
      tileSize: 256,
    });
    map.addLayer({
      id: FLOW_R,
      type: "raster",
      source: FLOW_R,
    });

    // -------------------------
    // Incidents raster（PNG）
    // -------------------------
    map.addSource(INC_R, {
      type: "raster",
      tiles: [incRasterTiles()],
      tileSize: 256,
    });
    map.addLayer({
      id: INC_R,
      type: "raster",
      source: INC_R,
    });

    // -------------------------
    // Flow vector（PBF）
    // -------------------------
    map.addSource(FLOW_V, {
      type: "vector",
      tiles: [flowVectorTiles()],
    });

    /**
     * 注意：
     * - "source-layer" は PBFの中のレイヤー名
     * - 表示されない場合、ここが一致していない可能性が高い
     */
    map.addLayer({
      id: FLOW_V,
      type: "line",
      source: FLOW_V,
      "source-layer": "Traffic flow",
      paint: {
        // ここは最低限の固定スタイル（表示確認用）
        "line-color": "#00ff88",
        "line-width": 3,
      },
    });

    // -------------------------
    // Incidents vector（PBF）
    // -------------------------
    map.addSource(INC_V, {
      type: "vector",
      tiles: [incVectorTiles()],
    });

    // Incidents: 線（規制区間など）
    map.addLayer({
      id: "incLine",
      type: "line",
      source: INC_V,
      "source-layer": "Traffic incident flow",
      paint: {
        "line-color": "#ff6600",
        "line-width": 4,
      },
    });

    // Incidents: 点（事故アイコン等）
    map.addLayer({
      id: "incPoint",
      type: "circle",
      source: INC_V,
      "source-layer": "Traffic incident points",
      paint: {
        "circle-radius": 5,
        "circle-color": "#ff6600",
      },
    });

    // 初期状態のON/OFF反映
    applyVisibility();

    console.log("map ready");
  });

  /**
   * map.on("error"):
   * - タイル取得失敗（401/403/CORS/ネットワーク等）のログ確認に使える
   * - MapLibre内部エラーはここに来ることがある
   */
  map.on("error", (e) => {
    console.log("map error:", e.error);
  });
});

onBeforeUnmount(() => {
  // Mapインスタンスを破棄（メモリリーク防止）
  if (map) map.remove();
});
</script>

<style scoped>
.wrap {
  width: 90vw;
  height: 85vh;
  position: relative;
}

/* 地図表示領域（全面） */
.map {
  width: 100%;
  height: 100%;
}

/* パネルは地図の上に重ねる */
.panel {
  position: absolute;
  z-index: 10;
  top: 10px;
  left: 10px;
  background: rgba(0, 0, 0, 0.7);
  color: #fff;
  padding: 10px;
  border-radius: 12px;
}

.row {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

.chk {
  display: flex;
  gap: 6px;
  align-items: center;
}

/* 公式リンク（見た目だけ） */
.official-link {
  color: #4ade80;
  text-decoration: none;
  font-weight: bold;
}
.official-link:hover {
  text-decoration: underline;
}
</style>
