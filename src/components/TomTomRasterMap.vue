<template>
  <div class="wrap">
    <!-- =========================
         操作用パネル（最小構成）
         - 都市へ移動（flyTo）
         - レイヤー表示ON/OFF
         ========================= -->
    <div class="panel">
      <div class="row">
        <button @click="flyToTokyo">東京</button>
        <button @click="flyToOsaka">大阪</button>
        <button @click="flyToFukuoka">福岡</button>
      </div>

      <div class="row">
        <!-- Base: 背景地図（国土地理院タイル） -->
        <label class="chk">
          <input type="checkbox" v-model="showBase" @change="applyVisibility" />
          Base
        </label>

        <!-- Flow: 渋滞（速度/相対速度）レイヤー -->
        <!-- raster: PNG（サーバ側で描画済みの“絵”） -->
        <label class="chk">
          <input
            type="checkbox"
            v-model="showFlowRaster"
            @change="applyVisibility"
          />
          Flow (raster)
        </label>

        <!-- vector: PBF（ラインFeature） -->
        <label class="chk">
          <input
            type="checkbox"
            v-model="showFlowVector"
            @change="applyVisibility"
          />
          Flow (vector)
        </label>

        <!-- Incidents: 事故・規制等 -->
        <!-- raster: PNG（サーバ側で描画済みの“絵”） -->
        <label class="chk">
          <input
            type="checkbox"
            v-model="showIncRaster"
            @change="applyVisibility"
          />
          Incidents (raster)
        </label>

        <!-- vector: PBF（点/線Feature） -->
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

    <!-- MapLibre の描画領域 -->
    <div ref="mapEl" class="map"></div>
  </div>
</template>

<script setup>
import { onMounted, onBeforeUnmount, ref } from "vue";
import maplibregl from "maplibre-gl";

/**
 * ============================================
 * 設定値（環境変数/固定パラメータ）
 * ============================================
 */

// TomTom API Key（.env の VITE_TOMTOM_API_KEY を参照）
const key = import.meta.env.VITE_TOMTOM_API_KEY;

// Orbis Traffic API のクエリパラメータ（このまま固定でOK）
const ORBIS_API_VERSION = 1; // apiVersion
const ORBIS_STYLE = "light"; // rasterタイルの見た目（light/dark等）
const ORBIS_TILE_SIZE = 256; // tileSize
const ORBIS_TRAFFIC_MODEL_ID = -1; // t=-1 は “最新” を指定（incidents系で利用）

/**
 * ============================================
 * Vue / MapLibre 状態
 * ============================================
 */
const mapEl = ref(null);
let map = null;

/**
 * 表示ON/OFF（チェックボックス）
 * - “描画を消す/出す”だけ（APIの取得はMapLibreが必要に応じて行う）
 */
const showBase = ref(true);
const showFlowRaster = ref(true);
const showFlowVector = ref(false);
const showIncRaster = ref(true);
const showIncVector = ref(false);

/**
 * レイヤー/ソースID
 * - MapLibre 内で一意である必要あり
 * - ここにまとまっていると後から追いやすい
 */
const BASE = "base";
const FLOW_R = "flowRaster";
const FLOW_V = "flowVector";
const INC_R = "incRaster";
const INC_V = "incVector";

/**
 * ============================================
 * Tile URL（どのデータをどこから取るか）
 * ============================================
 *
 * ※ {z}/{x}/{y} はMapLibreが自動で差し込む（タイル座標）
 * ※ baseだけ国土地理院、交通はTomTom Orbis
 */

// 以前のTomTomベース地図（残しておく：比較用）
// function baseTiles() {
//   return `https://api.tomtom.com/map/1/tile/basic/main/{z}/{x}/{y}.png?key=${key}&tileSize=256&view=Unified`;
// }

/**
 * Base（背景地図）
 * - 国土地理院 標準地図（std）
 * - キー不要
 * - 運用時は出典表記など規約に従うこと
 */
function baseTiles() {
  return "https://cyberjapandata.gsi.go.jp/xyz/std/{z}/{x}/{y}.png";
}

/**
 * Flow（渋滞）Raster（PNG）
 * - “描画済みの画像”が返る（道路が色で塗られている）
 */
function flowRasterTiles() {
  return `https://api.tomtom.com/maps/orbis/traffic/tile/flow/{z}/{x}/{y}.png?apiVersion=${ORBIS_API_VERSION}&key=${key}&style=${ORBIS_STYLE}&tileSize=${ORBIS_TILE_SIZE}`;
}

/**
 * Incidents（事故・規制）Raster（PNG）
 * - “描画済みの画像”が返る（アイコン等）
 * - t=-1（最新モデル）を指定
 */
function incRasterTiles() {
  return `https://api.tomtom.com/maps/orbis/traffic/tile/incidents/{z}/{x}/{y}.png?apiVersion=${ORBIS_API_VERSION}&key=${key}&style=${ORBIS_STYLE}&t=${ORBIS_TRAFFIC_MODEL_ID}&tileSize=${ORBIS_TILE_SIZE}`;
}

/**
 * Flow（渋滞）Vector（PBF）
 * - PBF（Mapbox Vector Tile）形式でFeatureが返る
 * - MapLibre側で自由にスタイル付け可能
 */
function flowVectorTiles() {
  return `https://api.tomtom.com/maps/orbis/traffic/tile/flow/{z}/{x}/{y}.pbf?apiVersion=${ORBIS_API_VERSION}&key=${key}`;
}

/**
 * Incidents（事故・規制）Vector（PBF）
 * - 点（ポイント）や線（区間）のFeatureが返る
 * - t=-1（最新モデル）を指定
 */
function incVectorTiles() {
  return `https://api.tomtom.com/maps/orbis/traffic/tile/incidents/{z}/{x}/{y}.pbf?apiVersion=${ORBIS_API_VERSION}&key=${key}&t=${ORBIS_TRAFFIC_MODEL_ID}`;
}

/**
 * ============================================
 * レイヤー表示制御（visibilityの切替）
 * ============================================
 *
 * - レイヤーが存在する場合のみ setLayoutProperty を呼ぶ
 * - visibility = "none" にすると画面から消える
 */
function setVisible(id, v) {
  if (map.getLayer(id)) {
    map.setLayoutProperty(id, "visibility", v ? "visible" : "none");
  }
}

/**
 * 画面上のON/OFFをMapLibreのレイヤーに反映
 * ※ログはデバッグ用途（必要なければ消してOK）
 */
function applyVisibility() {
  // Base
  setVisible(BASE, showBase.value);

  // Flow: raster / vector
  setVisible(FLOW_R, showFlowRaster.value);
  setVisible(FLOW_V, showFlowVector.value);

  // Incidents: raster / vector
  setVisible(INC_R, showIncRaster.value);

  // incidents vectorは “線” と “点” を別レイヤーで描画している
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

/**
 * ============================================
 * API実行結果をログ出力（incidentDetails）
 * ============================================
 *
 * - タイル（raster/vector）はMapLibreが自動取得するため
 *   “自分で叩くAPI”としては incidentDetails を用意
 * - 現在表示範囲（bbox）で incidentDetails を叩き、レスポンスをconsoleに出す
 */
async function logIncidentDetails() {
  // 画面表示範囲（西南東北）を bbox として組み立て
  const b = map.getBounds();
  const bbox = `${b.getWest()},${b.getSouth()},${b.getEast()},${b.getNorth()}`;

  // incidentDetails（JSON）
  const url =
    `https://api.tomtom.com/maps/orbis/traffic/incidentDetails` +
    `?apiVersion=${ORBIS_API_VERSION}` +
    `&key=${key}` +
    `&bbox=${bbox}` +
    `&timeValidityFilter=present` +
    `&t=${ORBIS_TRAFFIC_MODEL_ID}`;

  // 実行時間を測っておく（ネットワーク状況の把握に便利）
  const t0 = performance.now();
  const res = await fetch(url);
  const txt = await res.text();
  const ms = Math.round(performance.now() - t0);

  console.log("incidentDetails", {
    request: { url, bbox },
    response: { status: res.status, timeMs: ms, body: txt },
  });
}

/**
 * ============================================
 * FlyTo（都市へジャンプ）
 * ============================================
 *
 * - flyTo後、moveendを待ってから incidentDetails を1回ログ出力する
 * - 目的：移動先で“APIが取れているか”を確実に確認する
 */
async function flyTo(center, label) {
  console.log("flyTo:", label);

  // speed は移動アニメーション速度（数値が大きいほど速い）
  map.flyTo({ center, speed: 1.2 });

  // flyToの完了（moveend）を待つ
  await new Promise((r) => map.once("moveend", r));

  console.log("arrived:", label);

  // 移動後の表示範囲でAPI結果をログ出し
  await logIncidentDetails();
}

/**
 * 都市ボタン用（座標は駅付近）
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

/**
 * ============================================
 * 初期化（MapLibreの起動＆レイヤー登録）
 * ============================================
 */
onMounted(() => {
  // APIキー未設定のときはここで気づけるように（画面に出したいならalertでもOK）
  if (!key) {
    console.warn("VITE_TOMTOM_API_KEY が未設定です (.env を確認)");
  }

  // MapLibre Map 初期化
  // style は最小構成（背景色のみ）にして、タイルは自前で追加
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
    center: [139.767125, 35.681236], // 初期は東京
    zoom: 11,
  });

  // 読み込み完了後にソース/レイヤーを追加
  map.on("load", () => {
    /**
     * -------------------------
     * Base（国土地理院）
     * -------------------------
     */
    map.addSource(BASE, {
      type: "raster",
      tiles: [baseTiles()],
      tileSize: 256,
    });
    map.addLayer({ id: BASE, type: "raster", source: BASE });

    /**
     * -------------------------
     * Flow raster（PNG）
     * -------------------------
     */
    map.addSource(FLOW_R, {
      type: "raster",
      tiles: [flowRasterTiles()],
      tileSize: 256,
    });
    map.addLayer({ id: FLOW_R, type: "raster", source: FLOW_R });

    /**
     * -------------------------
     * Incidents raster（PNG）
     * -------------------------
     */
    map.addSource(INC_R, {
      type: "raster",
      tiles: [incRasterTiles()],
      tileSize: 256,
    });
    map.addLayer({ id: INC_R, type: "raster", source: INC_R });

    /**
     * -------------------------
     * Flow vector（PBF）
     * -------------------------
     *
     * "source-layer" は PBF内のレイヤー名
     * 表示が出ない場合は、ここが合ってない可能性がある
     */
    map.addSource(FLOW_V, {
      type: "vector",
      tiles: [flowVectorTiles()],
    });
    map.addLayer({
      id: FLOW_V,
      type: "line",
      source: FLOW_V,
      "source-layer": "Traffic flow",
      paint: {
        // ひとまず固定色（要件は“表示できること”）
        "line-color": "#00ff88",
        "line-width": 3,
      },
    });

    /**
     * -------------------------
     * Incidents vector（PBF）
     * -------------------------
     *
     * incidentsは線/点が別レイヤーで入っているため、
     * MapLibre側でも別レイヤーとして描画している
     */
    map.addSource(INC_V, {
      type: "vector",
      tiles: [incVectorTiles()],
    });

    // 区間（line）
    map.addLayer({
      id: "incLine",
      type: "line",
      source: INC_V,
      "source-layer": "Traffic incident flow",
      paint: { "line-color": "#ff6600", "line-width": 4 },
    });

    // 点（point）
    map.addLayer({
      id: "incPoint",
      type: "circle",
      source: INC_V,
      "source-layer": "Traffic incident points",
      paint: { "circle-radius": 5, "circle-color": "#ff6600" },
    });

    // 初期の表示ON/OFF反映
    applyVisibility();

    // 起動確認ログ（必要なければ消してOK）
    console.log("map ready");
  });

  /**
   * MapLibre内部のロードエラー（タイル取得失敗など）
   * - 401/403/CORSなどの調査に役立つ
   */
  map.on("error", (e) => {
    console.log("map error:", e.error);
  });
});

onBeforeUnmount(() => {
  // コンポーネント破棄時にMapを破棄（メモリリーク防止）
  if (map) map.remove();
});
</script>

<style scoped>
.wrap {
  width: 90vw;
  height: 85vh;
  position: relative;
}
.map {
  width: 100%;
  height: 100%;
}

/* 操作用パネル */
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
</style>
