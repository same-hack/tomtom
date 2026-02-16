<template>
  <div class="wrap">
    <div class="panel">
      <div class="row">
        <button @click="flyToTokyo">東京</button>
        <button @click="flyToOsaka">大阪</button>
        <button @click="flyToFukuoka">福岡</button>
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

const key = import.meta.env.VITE_TOMTOM_API_KEY;

const ORBIS_API_VERSION = 1;
const ORBIS_STYLE = "light";
const ORBIS_TILE_SIZE = 256;
const ORBIS_TRAFFIC_MODEL_ID = -1;

const mapEl = ref(null);
let map = null;

// visibility
const showBase = ref(true);
const showFlowRaster = ref(true);
const showFlowVector = ref(false);
const showIncRaster = ref(true);
const showIncVector = ref(false);

// source/layer ids
const BASE = "base";
const FLOW_R = "flowRaster";
const FLOW_V = "flowVector";
const INC_R = "incRaster";
const INC_V = "incVector";

// ---------- URLs ----------
function baseTiles() {
  return `https://api.tomtom.com/map/1/tile/basic/main/{z}/{x}/{y}.png?key=${key}&tileSize=256&view=Unified`;
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

// ---------- visibility ----------
function setVisible(id, v) {
  if (map.getLayer(id)) {
    map.setLayoutProperty(id, "visibility", v ? "visible" : "none");
  }
}

function applyVisibility() {
  setVisible(BASE, showBase.value);
  setVisible(FLOW_R, showFlowRaster.value);
  setVisible(FLOW_V, showFlowVector.value);
  setVisible(INC_R, showIncRaster.value);
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

// ---------- API logging ----------
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

  const t0 = performance.now();
  const res = await fetch(url);
  const txt = await res.text();
  const ms = Math.round(performance.now() - t0);

  console.log("incidentDetails", {
    status: res.status,
    timeMs: ms,
    body: txt,
  });
}

// ---------- move ----------
async function flyTo(center, label) {
  console.log("flyTo:", label);
  map.flyTo({ center, speed: 1.2 });
  await new Promise((r) => map.once("moveend", r));
  console.log("arrived:", label);
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

// ---------- init ----------
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
    zoom: 11,
  });

  map.on("load", () => {
    // Base
    map.addSource(BASE, {
      type: "raster",
      tiles: [baseTiles()],
      tileSize: 256,
    });
    map.addLayer({ id: BASE, type: "raster", source: BASE });

    // Flow raster
    map.addSource(FLOW_R, {
      type: "raster",
      tiles: [flowRasterTiles()],
      tileSize: 256,
    });
    map.addLayer({ id: FLOW_R, type: "raster", source: FLOW_R });

    // Inc raster
    map.addSource(INC_R, {
      type: "raster",
      tiles: [incRasterTiles()],
      tileSize: 256,
    });
    map.addLayer({ id: INC_R, type: "raster", source: INC_R });

    // Flow vector
    map.addSource(FLOW_V, { type: "vector", tiles: [flowVectorTiles()] });
    map.addLayer({
      id: FLOW_V,
      type: "line",
      source: FLOW_V,
      "source-layer": "Traffic flow",
      paint: {
        "line-color": "#00ff88",
        "line-width": 3,
      },
    });

    // Inc vector
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

    applyVisibility();
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
</style>
