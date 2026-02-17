<template>
  <div class="wrap">
    <div class="panel">
      <div class="row">
        <button @click="flyToTokyo">東京</button>
        <button @click="flyToOsaka">大阪</button>
        <button @click="flyToFukuoka">福岡</button>

        <a
          href="https://developer.tomtom.com/traffic-api/documentation/tomtom-orbis-maps/product-information/introduction"
          target="_blank"
          rel="noopener noreferrer"
          class="official-link"
        >
          公式サイトへ
        </a>
      </div>

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

    <div ref="mapEl" class="map"></div>
  </div>
</template>

<script setup>
import { onMounted, onBeforeUnmount, ref } from "vue";
import maplibregl from "maplibre-gl";

/* =========================================================
   1) 設定値
   ========================================================= */

const key = import.meta.env.VITE_TOMTOM_API_KEY;

const ORBIS_API_VERSION = 1;
const ORBIS_STYLE = "light";
const ORBIS_TILE_SIZE = 256;
const ORBIS_TRAFFIC_MODEL_ID = -1;

/* =========================================================
   2) 状態管理
   ========================================================= */

const mapEl = ref(null);
let map = null;

/* デフォルトはTraffic系すべてOFF */
const showBase = ref(true);
const showFlowRaster = ref(false);
const showFlowVector = ref(false);
const showIncRaster = ref(false);
const showIncVector = ref(false);

const BASE = "base";
const FLOW_R = "flowRaster";
const FLOW_V = "flowVector";
const INC_R = "incRaster";
const INC_V = "incVector";

/* =========================================================
   3) タイルURL
   ========================================================= */

function baseTiles() {
  return "https://cyberjapandata.gsi.go.jp/xyz/std/{z}/{x}/{y}.png";
}

function flowRasterTiles() {
  return `https://api.tomtom.com/maps/orbis/traffic/tile/flow/{z}/{x}/{y}.png?apiVersion=${ORBIS_API_VERSION}&key=${key}&style=${ORBIS_STYLE}&tileSize=${ORBIS_TILE_SIZE}`;
}

function incRasterTiles() {
  return `https://api.tomtom.com/maps/orbis/traffic/tile/incidents/{z}/{x}/{y}.png?apiVersion=${ORBIS_API_VERSION}&key=${key}&style=${ORBIS_STYLE}&t=${ORBIS_TRAFFIC_MODEL_ID}&tileSize=${ORBIS_TILE_SIZE}`;
}

function flowVectorTiles() {
  return `https://api.tomtom.com/maps/orbis/traffic/tile/flow/{z}/{x}/{y}.pbf?apiVersion=${ORBIS_API_VERSION}&key=${key}`;
}

function incVectorTiles() {
  return `https://api.tomtom.com/maps/orbis/traffic/tile/incidents/{z}/{x}/{y}.pbf?apiVersion=${ORBIS_API_VERSION}&key=${key}&t=${ORBIS_TRAFFIC_MODEL_ID}`;
}

/* =========================================================
   4) Source / Layer 管理
   ========================================================= */

function removeLayerIfExists(id) {
  if (map.getLayer(id)) map.removeLayer(id);
}

function removeSourceIfExists(id) {
  if (map.getSource(id)) map.removeSource(id);
}

function applyVisibility() {
  if (!map) return;

  /* Base */
  if (!map.getSource(BASE)) {
    map.addSource(BASE, {
      type: "raster",
      tiles: [baseTiles()],
      tileSize: 256,
    });
    map.addLayer({ id: BASE, type: "raster", source: BASE });
  }
  map.setLayoutProperty(
    BASE,
    "visibility",
    showBase.value ? "visible" : "none",
  );

  /* Flow Raster */
  if (showFlowRaster.value) {
    if (!map.getSource(FLOW_R)) {
      map.addSource(FLOW_R, {
        type: "raster",
        tiles: [flowRasterTiles()],
        tileSize: 256,
      });
      map.addLayer({ id: FLOW_R, type: "raster", source: FLOW_R });
    }
  } else {
    removeLayerIfExists(FLOW_R);
    removeSourceIfExists(FLOW_R);
  }

  /* Inc Raster */
  if (showIncRaster.value) {
    if (!map.getSource(INC_R)) {
      map.addSource(INC_R, {
        type: "raster",
        tiles: [incRasterTiles()],
        tileSize: 256,
      });
      map.addLayer({ id: INC_R, type: "raster", source: INC_R });
    }
  } else {
    removeLayerIfExists(INC_R);
    removeSourceIfExists(INC_R);
  }

  /* Flow Vector */
  if (showFlowVector.value) {
    if (!map.getSource(FLOW_V)) {
      map.addSource(FLOW_V, { type: "vector", tiles: [flowVectorTiles()] });
      map.addLayer({
        id: FLOW_V,
        type: "line",
        source: FLOW_V,
        "source-layer": "Traffic flow",
        paint: { "line-color": "#00ff88", "line-width": 3 },
      });
    }
  } else {
    removeLayerIfExists(FLOW_V);
    removeSourceIfExists(FLOW_V);
  }

  /* Inc Vector */
  if (showIncVector.value) {
    if (!map.getSource(INC_V)) {
      map.addSource(INC_V, { type: "vector", tiles: [incVectorTiles()] });

      map.addLayer({
        id: "incLine",
        type: "line",
        source: INC_V,
        "source-layer": "Traffic incident flow",
        paint: { "line-color": "#ff6600", "line-width": 4 },
      });

      map.addLayer({
        id: "incPoint",
        type: "circle",
        source: INC_V,
        "source-layer": "Traffic incident points",
        paint: { "circle-radius": 5, "circle-color": "#ff6600" },
      });
    }
  } else {
    removeLayerIfExists("incLine");
    removeLayerIfExists("incPoint");
    removeSourceIfExists(INC_V);
  }

  console.log("visibility updated");
}

/* =========================================================
   5) JSON API確認
   ========================================================= */

async function logIncidentDetails() {
  const b = map.getBounds();
  const bbox = `${b.getWest()},${b.getSouth()},${b.getEast()},${b.getNorth()}`;

  const url =
    `https://api.tomtom.com/maps/orbis/traffic/incidentDetails` +
    `?apiVersion=${ORBIS_API_VERSION}` +
    `&key=${key}` +
    `&bbox=${bbox}` +
    `&timeValidityFilter=present` +
    `&t=${ORBIS_TRAFFIC_MODEL_ID}`;

  const res = await fetch(url);
  const txt = await res.text();

  console.log("incidentDetails:", res.status, txt);
}

/* =========================================================
   6) flyTo（移動中はTraffic完全停止）
   ========================================================= */

async function flyTo(center, label) {
  console.log("flyTo:", label);

  /* 現在の状態保存 */
  const prev = {
    flowR: showFlowRaster.value,
    flowV: showFlowVector.value,
    incR: showIncRaster.value,
    incV: showIncVector.value,
  };

  /* すべて停止 */
  showFlowRaster.value = false;
  showFlowVector.value = false;
  showIncRaster.value = false;
  showIncVector.value = false;
  applyVisibility();

  map.flyTo({ center, speed: 1.2 });

  await new Promise((r) => map.once("moveend", r));

  console.log("arrived:", label);

  /* 元に戻す（ここで初取得） */
  showFlowRaster.value = prev.flowR;
  showFlowVector.value = prev.flowV;
  showIncRaster.value = prev.incR;
  showIncVector.value = prev.incV;
  applyVisibility();

  await logIncidentDetails();
}

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
   7) 初期化
   ========================================================= */

onMounted(() => {
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
    center: [139.767125, 35.681236],
    zoom: 15,
  });

  map.on("load", () => {
    applyVisibility(); // Baseのみ有効
    console.log("map ready");
  });

  map.on("error", (e) => {
    console.log("map error:", e.error);
  });
});

onBeforeUnmount(() => {
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
.official-link {
  color: #4ade80;
  text-decoration: none;
  font-weight: bold;
}
.official-link:hover {
  text-decoration: underline;
}
</style>
