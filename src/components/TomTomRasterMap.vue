<template>
  <div class="wrap">
    <div class="panel">
      <button @click="toggleBase">ベースON/OFF</button>
      <button @click="fetchIncidentDetailsBbox">
        画面範囲の件数(JSON)取得
      </button>
      <button @click="openCenterIncidentTile">中心タイル(PNG)を開く</button>

      <label class="chk">
        <input type="checkbox" v-model="autoFetchOnMove" />
        moveendで自動取得
      </label>

      <label class="row">
        incidents style:
        <select v-model="incidentStyle" @change="reloadIncidentRaster">
          <option value="s0">s0</option>
          <option value="day">day</option>
          <option value="dark">dark</option>
          <option value="s3">s3</option>
        </select>
      </label>

      <span class="hint">ズーム14〜16で確認推奨</span>
    </div>

    <div ref="mapEl" class="map"></div>
  </div>
</template>

<script setup>
import { onMounted, onBeforeUnmount, ref } from "vue";
import maplibregl from "maplibre-gl";

const mapEl = ref(null);
let map = null;

const autoFetchOnMove = ref(true);
const incidentStyle = ref("s0");

const key = import.meta.env.VITE_TOMTOM_API_KEY;

// IDs
const BASE_SOURCE_ID = "tomtomBase";
const BASE_LAYER_ID = "tomtomBaseLayer";
const INCIDENT_RASTER_SOURCE_ID = "tomtomIncidentsRaster";
const INCIDENT_RASTER_LAYER_ID = "tomtomIncidentsRasterLayer";

let baseVisible = true;

// ---- URL builders ----
function baseTilesUrl() {
  // TomTom Map Display raster tiles
  return `https://api.tomtom.com/map/1/tile/basic/main/{z}/{x}/{y}.png?key=${encodeURIComponent(
    key,
  )}&tileSize=256&view=Unified`;
}

function incidentRasterTilesUrl(style) {
  // TomTom Traffic Incidents raster tiles
  // t=-1 => latest model
  return `https://api.tomtom.com/traffic/map/4/tile/incidents/${style}/{z}/{x}/{y}.png?key=${encodeURIComponent(
    key,
  )}&tileSize=256&t=-1`;
}

// ---- Helpers ----
function lngLatToTileXY(lng, lat, z) {
  const n = 2 ** z;
  const x = Math.floor(((lng + 180) / 360) * n);

  const latRad = (lat * Math.PI) / 180;
  const y = Math.floor(
    ((1 - Math.log(Math.tan(latRad) + 1 / Math.cos(latRad)) / Math.PI) / 2) * n,
  );
  return { x, y };
}

// ---- UI actions ----
function toggleBase() {
  if (!map?.getLayer(BASE_LAYER_ID)) return;
  baseVisible = !baseVisible;
  map.setLayoutProperty(
    BASE_LAYER_ID,
    "visibility",
    baseVisible ? "visible" : "none",
  );
}

function reloadIncidentRaster() {
  if (!map) return;
  if (map.getLayer(INCIDENT_RASTER_LAYER_ID))
    map.removeLayer(INCIDENT_RASTER_LAYER_ID);
  if (map.getSource(INCIDENT_RASTER_SOURCE_ID))
    map.removeSource(INCIDENT_RASTER_SOURCE_ID);

  map.addSource(INCIDENT_RASTER_SOURCE_ID, {
    type: "raster",
    tiles: [incidentRasterTilesUrl(incidentStyle.value)],
    tileSize: 256,
  });

  map.addLayer({
    id: INCIDENT_RASTER_LAYER_ID,
    type: "raster",
    source: INCIDENT_RASTER_SOURCE_ID,
    paint: { "raster-opacity": 1.0 },
  });
}

function openCenterIncidentTile() {
  if (!map) return;

  const z = Math.round(map.getZoom());
  const { lng, lat } = map.getCenter();
  const { x, y } = lngLatToTileXY(lng, lat, z);

  const url =
    `https://api.tomtom.com/traffic/map/4/tile/incidents/${incidentStyle.value}/${z}/${x}/${y}.png` +
    `?key=${encodeURIComponent(key)}&tileSize=256&t=-1`;

  window.open(url, "_blank");
  console.log("center incident tile:", { z, x, y, url });
}

// ---- JSON fetch (Incident Details) ----
async function fetchIncidentDetailsBbox() {
  if (!map) return;
  if (!key) {
    console.error("VITE_TOMTOM_API_KEY が未設定です (.env)");
    return;
  }

  const b = map.getBounds();
  const west = b.getWest();
  const south = b.getSouth();
  const east = b.getEast();
  const north = b.getNorth();

  // bbox = lonMin,latMin,lonMax,latMax
  const bbox = `${west},${south},${east},${north}`;

  // なるべく軽く（必要なものだけ）
  const fields =
    `{incidents{type,geometry{type,coordinates},properties{` +
    `id,iconCategory,magnitudeOfDelay,delay,length,` +
    `events{description,code,iconCategory},startTime,endTime,roadNumbers,timeValidity` +
    `}}}`;

  // ★重要：language=ja-JP はこのAPIで弾かれるので付けない（400回避）
  // もし日本語を試すなら language=ja を追加してみる（環境により対応差あり）
  const url =
    `https://api.tomtom.com/traffic/services/5/incidentDetails` +
    `?key=${encodeURIComponent(key)}` +
    `&bbox=${encodeURIComponent(bbox)}` +
    `&fields=${encodeURIComponent(fields)}` +
    `&timeValidityFilter=present`;

  console.log("Incident Details request:", { bbox, url });

  try {
    const res = await fetch(url, { cache: "no-store" });
    const text = await res.text();

    if (!res.ok) {
      console.error("Incident Details HTTP error:", res.status, text);
      return;
    }

    const json = JSON.parse(text);
    const incidents = json?.incidents ?? [];

    console.log("==== Incident Details result ====");
    console.log("bbox:", bbox);
    console.log("incident count:", incidents.length);
    console.log(
      "sample(0..2):",
      incidents.slice(0, 3).map((f) => ({
        id: f?.properties?.id,
        iconCategory: f?.properties?.iconCategory,
        delay: f?.properties?.delay,
        length: f?.properties?.length,
        desc: f?.properties?.events?.[0]?.description,
      })),
    );
  } catch (e) {
    console.error("Incident Details fetch failed:", e);
  }
}

// ---- Map init ----
function onMoveEnd() {
  if (!autoFetchOnMove.value) return;
  fetchIncidentDetailsBbox();
}

onMounted(() => {
  if (!key) {
    alert("VITE_TOMTOM_API_KEY が未設定です (.env を確認)");
    return;
  }

  map = new maplibregl.Map({
    container: mapEl.value,
    style: {
      version: 8,
      sources: {},
      layers: [
        // 背景（真っ黒だと“空タイル”と区別しにくいので少し明るい）
        {
          id: "bg",
          type: "background",
          paint: { "background-color": "#2b2b2b" },
        },
      ],
    },
    center: [139.767, 35.681],
    zoom: 12,
  });

  map.addControl(new maplibregl.NavigationControl(), "top-right");

  map.on("load", () => {
    // Base
    map.addSource(BASE_SOURCE_ID, {
      type: "raster",
      tiles: [baseTilesUrl()],
      tileSize: 256,
    });
    map.addLayer({ id: BASE_LAYER_ID, type: "raster", source: BASE_SOURCE_ID });

    // Incident raster
    reloadIncidentRaster();

    // 初回にJSON件数
    fetchIncidentDetailsBbox();

    // サイズ変更対策（80vw/80vhなどのとき）
    requestAnimationFrame(() => map.resize());
    window.addEventListener("resize", () => map.resize());

    map.on("moveend", onMoveEnd);
  });
});

onBeforeUnmount(() => {
  if (map) {
    map.off("moveend", onMoveEnd);
    map.remove();
  }
});
</script>

<style scoped>
.wrap {
  width: 80vw;
  height: 80vh;
  position: relative;
  overflow: hidden;
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
  background: rgba(255, 255, 255, 0.92);
  padding: 8px 10px;
  border-radius: 10px;
  display: flex;
  gap: 10px;
  align-items: center;
  flex-wrap: wrap;
}

.chk {
  display: inline-flex;
  gap: 6px;
  align-items: center;
}

.row {
  display: inline-flex;
  gap: 6px;
  align-items: center;
}

.hint {
  font-size: 12px;
  opacity: 0.8;
}
</style>
