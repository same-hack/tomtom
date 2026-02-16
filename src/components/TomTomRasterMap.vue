<template>
  <div class="wrap">
    <div class="panel">
      <div class="row">
        <button @click="start" :disabled="running">開始</button>
        <button @click="stop" :disabled="!running">停止</button>
        <button @click="flyToBerlin">ベルリンへ</button>
        <button @click="flyToFukuoka">福岡へ</button>
      </div>

      <div class="row">
        <label class="chk">
          <input type="checkbox" v-model="showBase" @change="applyVisibility" />
          Base
        </label>
        <label class="chk">
          <input type="checkbox" v-model="showInc" @change="applyVisibility" />
          Incidents
        </label>
        <label class="chk">
          <input type="checkbox" v-model="showFlow" @change="applyVisibility" />
          Flow
        </label>
      </div>

      <div class="log">
        <div>
          <b>Mode:</b> {{ USE_ORBIS_TRAFFIC ? "Orbis" : "TomTom Maps" }}
        </div>
        <div><b>Status:</b> {{ status }}</div>
        <div><b>Last:</b> {{ last }}</div>
        <div>
          <b>Move:</b>
          {{
            programmaticMove
              ? "programmatic"
              : manualMoving
                ? "manual"
                : running
                  ? "auto"
                  : "-"
          }}
        </div>
        <div v-if="USE_ORBIS_TRAFFIC && showFlow" class="hint">
          ※ Orbisモードでは
          FlowSegmentData（数値）を使わず、Flowはタイル表示のみです
        </div>
      </div>
    </div>

    <div ref="mapEl" class="map"></div>
  </div>
</template>

<script setup>
import { onMounted, onBeforeUnmount, ref } from "vue";
import maplibregl from "maplibre-gl";

/**
 * =========================
 *  Orbis切り替えスイッチ
 * =========================
 *
 * true  : TomTom Orbis Traffic API（日本でタイルが出る想定）
 * false : 従来の /traffic/services/* /traffic/map/* を使う
 */
const USE_ORBIS_TRAFFIC = true;

// Orbis params
const ORBIS_API_VERSION = 1;
const ORBIS_STYLE = "light"; // "dark" も可（好み）
const ORBIS_TILE_SIZE = 256;
// trafficModelId: -1 は “最新” を指す（incidents tile / incidentDetails で使用）
const ORBIS_TRAFFIC_MODEL_ID = -1;

const mapEl = ref(null);
let map = null;

const key = import.meta.env.VITE_TOMTOM_API_KEY;

// UI
const running = ref(false);
const status = ref("idle");
const last = ref("-");

// Visibility toggles
const showBase = ref(true);
const showInc = ref(true);
const showFlow = ref(true);

// Sources/Layers
const BASE_SOURCE = "base";
const BASE_LAYER = "baseLayer";
const FLOW_SOURCE = "flow";
const FLOW_LAYER = "flowLayer";
const INC_SOURCE = "inc";
const INC_LAYER = "incLayer";

// Control
let aborter = null;
let manualMoveTimer = null;
let inFlight = false;

const programmaticMove = ref(false);

// ✅ 初期 moveend 連打対策（ロード直後はAPI叩かない）
let readyAt = 0; // Date.now()
const INITIAL_SILENCE_MS = 1500;

// ✅ 手動移動判定（ドラッグ/ホイール/タッチ中のみAPI）
const manualMoving = ref(false);
let manualFlagTimer = null;
function markManualMoving() {
  manualMoving.value = true;
  if (manualFlagTimer) clearTimeout(manualFlagTimer);
  manualFlagTimer = setTimeout(() => (manualMoving.value = false), 800);
}

// ---- Tile URLs ----
function baseTilesUrl() {
  // ベース地図は従来のTomTom MapsでOK（表示の安定性が高い）
  return `https://api.tomtom.com/map/1/tile/basic/main/{z}/{x}/{y}.png?key=${encodeURIComponent(
    key,
  )}&tileSize=256&view=Unified`;
}

function flowTilesUrl() {
  if (USE_ORBIS_TRAFFIC) {
    // Orbis Flow Raster Tiles
    return (
      `https://api.tomtom.com/maps/orbis/traffic/tile/flow/{z}/{x}/{y}.png` +
      `?apiVersion=${ORBIS_API_VERSION}` +
      `&key=${encodeURIComponent(key)}` +
      `&style=${encodeURIComponent(ORBIS_STYLE)}` +
      `&tileSize=${ORBIS_TILE_SIZE}`
    );
  }
  // 従来（TomTom Maps Traffic）
  return `https://api.tomtom.com/traffic/map/4/tile/flow/relative0/{z}/{x}/{y}.png?key=${encodeURIComponent(
    key,
  )}&tileSize=256`;
}

function incidentTilesUrl() {
  if (USE_ORBIS_TRAFFIC) {
    // Orbis Incidents Raster Tiles
    return (
      `https://api.tomtom.com/maps/orbis/traffic/tile/incidents/{z}/{x}/{y}.png` +
      `?apiVersion=${ORBIS_API_VERSION}` +
      `&key=${encodeURIComponent(key)}` +
      `&style=${encodeURIComponent(ORBIS_STYLE)}` +
      `&t=${encodeURIComponent(String(ORBIS_TRAFFIC_MODEL_ID))}` +
      `&tileSize=${ORBIS_TILE_SIZE}`
    );
  }
  // 従来（TomTom Maps Traffic）
  return `https://api.tomtom.com/traffic/map/4/tile/incidents/s1/{z}/{x}/{y}.png?key=${encodeURIComponent(
    key,
  )}&tileSize=256`;
}

// ---- Helpers ----
function safeJsonParse(text) {
  try {
    return JSON.parse(text);
  } catch {
    return null;
  }
}
function sleep(ms) {
  return new Promise((r) => setTimeout(r, ms));
}
function nowStr() {
  return new Date().toLocaleTimeString();
}
function metersToLat(m) {
  return m / 111320;
}
function metersToLng(m, lat) {
  return m / (111320 * Math.cos((lat * Math.PI) / 180));
}
function getBboxParamFromMap() {
  const b = map.getBounds();
  return `${b.getWest()},${b.getSouth()},${b.getEast()},${b.getNorth()}`;
}
function setLast(msg) {
  last.value = `${nowStr()} ${msg}`;
}

// ---- Visibility ----
function setLayerVisible(layerId, visible) {
  if (!map || !map.getLayer(layerId)) return;
  map.setLayoutProperty(layerId, "visibility", visible ? "visible" : "none");
}
function applyVisibility() {
  setLayerVisible(BASE_LAYER, showBase.value);
  setLayerVisible(INC_LAYER, showInc.value);
  setLayerVisible(FLOW_LAYER, showFlow.value);
}

// ---- APIs ----
async function fetchIncidentsInViewport() {
  const bbox = getBboxParamFromMap();

  // fields は Orbis / 従来 どちらも “fields” パラメータで絞る用途として維持
  const fields = encodeURIComponent(
    "{incidents{type,geometry{type,coordinates},properties{iconCategory,magnitudeOfDelay,from,to,roadNumbers}}}",
  );

  const url = USE_ORBIS_TRAFFIC
    ? // ✅ Orbis Incident Details
      `https://api.tomtom.com/maps/orbis/traffic/incidentDetails` +
      `?apiVersion=${ORBIS_API_VERSION}` +
      `&key=${encodeURIComponent(key)}` +
      `&bbox=${encodeURIComponent(bbox)}` +
      `&fields=${fields}` +
      `&timeValidityFilter=present` +
      `&t=${encodeURIComponent(String(ORBIS_TRAFFIC_MODEL_ID))}`
    : // 従来（TomTom Maps Traffic）
      `https://api.tomtom.com/traffic/services/5/incidentDetails` +
      `?key=${encodeURIComponent(key)}` +
      `&bbox=${encodeURIComponent(bbox)}` +
      `&fields=${fields}` +
      `&timeValidityFilter=present`;

  const res = await fetch(url, { cache: "no-store", signal: aborter?.signal });
  const text = await res.text();
  const json = safeJsonParse(text);

  if (!res.ok) {
    console.warn("incidentDetails error:", res.status, text);
    return { ok: false, status: res.status, raw: json ?? text };
  }

  // Orbis: { incidents: [...] } / 従来: { incidents: [...] } or { incidentDetails: { incidents } }
  const incidents = json?.incidents ?? json?.incidentDetails?.incidents ?? [];
  return { ok: true, incidents };
}

/**
 * =========================
 * Flow（数値）の扱い
 * =========================
 *
 * - TomTom Maps の flowSegmentData は、日本で常時400になる実測が出ている
 * - Orbisは “Flowはタイル中心” のため、ここでは数値APIを使わず「表示のみ」にする
 *
 * どうしても数値が必要なら：
 * - TomTomの別プロダクト（Intermediate Traffic / Flow Detailed 等）をバックエンド経由で使う構成にするのが現実的
 */
async function fetchFlowSegmentData(lat, lng) {
  if (USE_ORBIS_TRAFFIC) {
    return {
      ok: false,
      status: "orbis_tiles_only",
      raw: "Orbis mode uses flow tiles only (no flowSegmentData).",
    };
  }

  // 従来（TomTom Maps Traffic）: ただし日本では400が続く可能性が高い
  const z = Math.min(22, Math.max(0, Math.round(map.getZoom())));
  const url =
    `https://api.tomtom.com/traffic/services/4/flowSegmentData/relative0/${z}/json` +
    `?key=${encodeURIComponent(key)}` +
    `&point=${encodeURIComponent(`${lat},${lng}`)}` +
    `&unit=kmph`;

  const res = await fetch(url, { cache: "no-store", signal: aborter?.signal });
  const text = await res.text();
  const json = safeJsonParse(text);

  if (!res.ok) {
    console.warn("flowSegmentData error:", res.status, text);
    return { ok: false, status: res.status, raw: json ?? text };
  }

  const fsd = json?.flowSegmentData ?? {};
  const current = Number(fsd.currentSpeed);
  const free = Number(fsd.freeFlowSpeed);
  const ratio = free > 0 ? current / free : null;
  return { ok: true, fsd, ratio, lat, lng };
}

// ✅ 近傍探索：試行回数を減らし、最大リクエスト数も制限
async function fetchFlowNear(lat, lng) {
  // Orbisモードは数値を取らない
  if (USE_ORBIS_TRAFFIC) return { ok: false, status: "orbis_tiles_only" };

  const radii = [0, 80, 160, 240, 360, 520]; // 6段階（軽量化）
  const dirs = [
    [0, 0],
    [1, 0],
    [-1, 0],
    [0, 1],
    [0, -1],
    [1, 1],
    [1, -1],
    [-1, 1],
    [-1, -1],
  ];

  const MAX_TRIES = 30;
  let tries = 0;

  for (const r of radii) {
    for (const [dx, dy] of dirs) {
      if (++tries > MAX_TRIES) return { ok: false, status: "try_limit" };
      if (aborter?.signal?.aborted) return { ok: false, status: "aborted" };

      const lat2 = lat + metersToLat(r) * dy;
      // ✅ lat2 を使って経度換算（地味にズレるのを防止）
      const lng2 = lng + metersToLng(r, lat2) * dx;

      const res = await fetchFlowSegmentData(lat2, lng2);
      if (res.ok) return res;

      // 400は次へ、403などは即返す
      if (res.status && res.status !== 400) return res;
    }
  }
  return { ok: false, status: 400 };
}

function isCongested(flow) {
  const closure = !!flow?.fsd?.roadClosure;
  const ratio = flow?.ratio;
  return closure || (ratio != null && ratio < 0.8);
}

// ---- Moveend handler (manual only) ----
function onMoveEnd() {
  if (!map) return;

  // 初期ロード直後は叩かない
  if (Date.now() - readyAt < INITIAL_SILENCE_MS) return;

  // プログラム移動中は叩かない
  if (programmaticMove.value) return;

  // 自動運行中は loop 側でチェックするので、ここでは叩かない
  if (running.value) return;

  // 手動操作っぽい時だけ叩く
  if (!manualMoving.value) return;

  if (manualMoveTimer) clearTimeout(manualMoveTimer);
  manualMoveTimer = setTimeout(() => {
    checkAndMaybeStop("manual");
  }, 350);
}

// ---- Detection (single flight) ----
async function checkAndMaybeStop(label) {
  if (!map) return false;
  if (inFlight) return false;
  inFlight = true;

  try {
    if (!aborter) aborter = new AbortController();

    const center = map.getCenter();

    // Incidents
    if (showInc.value) {
      status.value = `checking incidents (${label})...`;
      const inc = await fetchIncidentsInViewport();

      if (inc.ok && inc.incidents.length > 0) {
        setLast(`INCIDENT(${label}) count=${inc.incidents.length}`);
        stop();
        alert(`事故/規制を検出: ${inc.incidents.length}件`);
        return true;
      }

      if (!inc.ok) {
        setLast(`INC none/error status=${inc.status ?? "-"}`);
      }
    }

    // Flow
    if (showFlow.value) {
      if (USE_ORBIS_TRAFFIC) {
        // Orbis: Flowはタイル表示のみ（数値チェックはしない）
        status.value = `flow visible (tiles only) (${label})...`;
        setLast(`FLOW(${label}) tiles-only (no numeric check)`);
      } else {
        status.value = `checking flow (${label})...`;
        const flow = await fetchFlowNear(center.lat, center.lng);

        if (flow.ok) {
          setLast(`FLOW ok ratio=${flow.ratio?.toFixed(2) ?? "-"}`);
          if (isCongested(flow)) {
            setLast(
              `CONGESTION(${label}) ratio=${flow.ratio?.toFixed(2) ?? "-"}`,
            );
            stop();
            alert("渋滞/通行止めを検出");
            return true;
          }
        } else {
          setLast(`FLOW none/error status=${flow.status ?? "-"}`);
        }
      }
    }

    status.value = `no hit (${label})`;
    return false;
  } finally {
    inFlight = false;
  }
}

// ---- Auto loop (zoomは変えない) ----
async function loop() {
  while (running.value) {
    const c = map.getCenter();

    // 6km移動（広域） ※centerのみ変更、zoomは触らない
    const lat = c.lat + metersToLat(6000);
    const lng = c.lng + metersToLng(6000, c.lat);

    status.value = "moving (auto)...";
    map.easeTo({ center: [lng, lat], duration: 900 }); // ✅ zoom無し
    await new Promise((r) => map.once("moveend", r));
    if (!running.value) break;

    const stopped = await checkAndMaybeStop("auto");
    if (stopped) break;

    await sleep(400);
  }
}

// ---- Buttons ----
function start() {
  if (!map || running.value) return;
  aborter = new AbortController();
  running.value = true;
  status.value = "started";
  setLast("started");
  loop();
}

function stop() {
  running.value = false;
  status.value = "stopped";
  try {
    aborter?.abort();
  } catch {}
  aborter = null;
  inFlight = false;
}

async function flyToBerlin() {
  if (!map) return;
  programmaticMove.value = true;
  setLast("flyTo Berlin (API paused)");
  map.flyTo({ center: [13.405, 52.52], speed: 1.2 }); // ✅ zoom無し
  await new Promise((r) => map.once("moveend", r));
  programmaticMove.value = false;
  setLast("arrived Berlin (API resumed)");
}

async function flyToFukuoka() {
  if (!map) return;
  programmaticMove.value = true;
  setLast("flyTo Fukuoka (API paused)");
  map.flyTo({ center: [130.4017, 33.5902], speed: 1.2 }); // ✅ zoom無し
  await new Promise((r) => map.once("moveend", r));
  programmaticMove.value = false;
  setLast("arrived Fukuoka (API resumed)");
}

// ---- Map init ----
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
        {
          id: "bg",
          type: "background",
          paint: { "background-color": "#1d1d1d" },
        },
      ],
    },
    center: [139.7595, 35.684], // Tokyo
    zoom: 11,
  });

  map.addControl(new maplibregl.NavigationControl(), "top-right");

  // ✅ 手動操作検知
  map.on("dragstart", markManualMoving);
  map.on("zoomstart", markManualMoving);
  map.on("rotatestart", markManualMoving);
  map.on("pitchstart", markManualMoving);
  map.on("wheel", markManualMoving);
  map.on("touchstart", markManualMoving);

  map.on("load", () => {
    map.addSource(BASE_SOURCE, {
      type: "raster",
      tiles: [baseTilesUrl()],
      tileSize: 256,
    });
    map.addLayer({ id: BASE_LAYER, type: "raster", source: BASE_SOURCE });

    map.addSource(FLOW_SOURCE, {
      type: "raster",
      tiles: [flowTilesUrl()],
      tileSize: 256,
    });
    map.addLayer({
      id: FLOW_LAYER,
      type: "raster",
      source: FLOW_SOURCE,
      paint: { "raster-opacity": 0.9 },
    });

    map.addSource(INC_SOURCE, {
      type: "raster",
      tiles: [incidentTilesUrl()],
      tileSize: 256,
    });
    map.addLayer({
      id: INC_LAYER,
      type: "raster",
      source: INC_SOURCE,
      paint: { "raster-opacity": 0.95 },
    });

    applyVisibility();

    readyAt = Date.now();
    map.on("moveend", onMoveEnd);

    status.value = "ready";
    setLast("ready");
  });
});

onBeforeUnmount(() => {
  if (manualMoveTimer) clearTimeout(manualMoveTimer);
  if (manualFlagTimer) clearTimeout(manualFlagTimer);
  try {
    aborter?.abort();
  } catch {}
  if (map) {
    map.off("moveend", onMoveEnd);
    map.remove();
  }
});
</script>

<style scoped>
.wrap {
  width: 90vw;
  height: 85vh;
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
  background: rgba(0, 0, 0, 0.7);
  color: #fff;
  padding: 10px 12px;
  border-radius: 12px;
  display: flex;
  flex-direction: column;
  gap: 10px;
  min-width: 560px;
}

.row {
  display: inline-flex;
  gap: 10px;
  align-items: center;
  flex-wrap: wrap;
}
.chk {
  display: inline-flex;
  gap: 6px;
  align-items: center;
  user-select: none;
}
.log {
  font-size: 12px;
  border-top: 1px solid rgba(255, 255, 255, 0.2);
  padding-top: 8px;
  line-height: 1.35;
}
.hint {
  opacity: 0.85;
  font-size: 11px;
}
button {
  cursor: pointer;
}
input[type="checkbox"] {
  accent-color: #2dd4bf;
}
</style>
