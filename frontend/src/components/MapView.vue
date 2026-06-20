<script setup lang="ts">
import { onMounted, onUnmounted, ref, watch, nextTick } from 'vue';
import L from 'leaflet';
import 'leaflet/dist/leaflet.css';
import { useDroneStore } from '../store/drone';

const store = useDroneStore();
const mapContainer = ref<HTMLElement>();
let map: L.Map | null = null;
let waypointLayer: L.LayerGroup | null = null;
let routeLayer: L.Polyline | null = null;
let zoneLayer: L.LayerGroup | null = null;
let droneMarker: L.CircleMarker | null = null;

const addMode = ref(false);

declare global {
  interface Window {
    __droneRemoveWaypoint?: (id: string) => void;
    __droneToggleLock?: (id: string) => void;
  }
}

function initMap() {
  if (!mapContainer.value || map) return;
  map = L.map(mapContainer.value).setView(store.mapCenter, 12);
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '© OpenStreetMap',
    maxZoom: 18,
  }).addTo(map);

  waypointLayer = L.layerGroup().addTo(map);
  zoneLayer = L.layerGroup().addTo(map);

  window.__droneRemoveWaypoint = (id: string) => store.removeWaypoint(id);
  window.__droneToggleLock = (id: string) => {
    store.toggleWaypointLock(id);
    drawWaypoints();
    drawRoute();
  };

  map.on('click', (e: L.LeafletMouseEvent) => {
    if (addMode.value) {
      store.addWaypoint(e.latlng.lat, e.latlng.lng);
    }
  });
}

function drawNoFlyZones() {
  if (!zoneLayer) return;
  zoneLayer.clearLayers();
  for (const zone of store.noFlyZones) {
    const color =
      zone.type === 'airport' ? '#ef4444' :
      zone.type === 'military' ? '#f97316' : '#a855f7';
    L.circle([zone.center[0], zone.center[1]], {
      radius: zone.radius,
      color,
      fillColor: color,
      fillOpacity: 0.15,
      weight: 2,
    })
      .bindPopup(`<b>${zone.name}</b><br>Type: ${zone.type}<br>Radius: ${zone.radius}m`)
      .addTo(zoneLayer);
  }
}

function drawWaypoints() {
  if (!waypointLayer) return;
  waypointLayer.clearLayers();
  store.waypoints.forEach((wp, idx) => {
    const isLocked = !!wp.locked;
    const marker = L.circleMarker([wp.lat, wp.lng], {
      radius: isLocked ? 10 : 8,
      color: isLocked ? '#f59e0b' : '#3b82f6',
      fillColor: isLocked ? '#fbbf24' : '#60a5fa',
      fillOpacity: 0.9,
      weight: isLocked ? 3 : 2,
      dashArray: isLocked ? 'none' : undefined,
    });
    const label = `${isLocked ? '🔒 ' : ''}WP${idx + 1}`;
    marker.bindTooltip(label, {
      permanent: true,
      direction: 'top',
      className: isLocked ? 'wp-tooltip wp-tooltip-locked' : 'wp-tooltip',
    });

    const lockBtnLabel = isLocked
      ? `<button onclick="window.__droneToggleLock('${wp.id}')" style="margin-top:4px;color:#2563eb;padding:2px 6px;border:1px solid #2563eb;border-radius:4px;background:transparent;cursor:pointer">🔓 解锁</button>`
      : `<button onclick="window.__droneToggleLock('${wp.id}')" style="margin-top:4px;color:#f59e0b;padding:2px 6px;border:1px solid #f59e0b;border-radius:4px;background:transparent;cursor:pointer">🔒 锁定</button>`;

    const canRemove = !isLocked;
    const removeBtn = canRemove
      ? `<button onclick="window.__droneRemoveWaypoint('${wp.id}')" style="margin-top:4px;color:#ef4444;padding:2px 6px;border:1px solid #ef4444;border-radius:4px;background:transparent;cursor:pointer">删除</button>`
      : `<span style="margin-top:4px;color:#64748b;font-size:11px">（锁定中无法删除）</span>`;

    const dragHint = isLocked ? `<div style="font-size:10px;color:#94a3b8;margin-top:2px">位置已锁定，不可拖动</div>` : '';

    marker.bindPopup(`
      <div style="min-width:180px">
        <b>航点 ${idx + 1}${isLocked ? ' 🔒' : ''}</b><br>
        高度: ${wp.altitude}m<br>
        速度: ${wp.speed} m/s<br>
        动作: ${wp.action}<br>
        ${lockBtnLabel}
        ${removeBtn}
        ${dragHint}
      </div>
    `);

    if (!isLocked) {
      (marker as any).dragging?.enable();
      marker.on('dragend', (e: any) => {
        const ll = e.target.getLatLng();
        store.updateWaypoint(wp.id, { lat: ll.lat, lng: ll.lng });
      });
    } else {
      (marker as any).dragging?.disable();
    }

    marker.addTo(waypointLayer!);
  });
}

function drawRoute() {
  if (routeLayer && map) {
    map.removeLayer(routeLayer);
    routeLayer = null;
  }
  if (store.waypoints.length < 2 || !map) return;

  const latlngs = store.waypoints.map((w) => [w.lat, w.lng] as [number, number]);

  let hasDanger = false;
  for (const wp of store.waypoints) {
    for (const zone of store.noFlyZones) {
      const d = Math.sqrt(
        (wp.lat - zone.center[0]) ** 2 + (wp.lng - zone.center[1]) ** 2
      ) * 111000;
      if (d < zone.radius * 1.5) hasDanger = true;
    }
  }

  routeLayer = L.polyline(latlngs, {
    color: hasDanger ? '#ef4444' : '#22c55e',
    weight: 3,
    opacity: 0.8,
    dashArray: hasDanger ? '8,4' : undefined,
  }).addTo(map);
}

function drawSimDrone() {
  if (!map || store.waypoints.length < 2) return;
  const progress = store.simProgress / 100;
  const totalWp = store.waypoints.length;
  const segIdx = Math.min(Math.floor(progress * (totalWp - 1)), totalWp - 2);
  const segProgress = (progress * (totalWp - 1)) - segIdx;
  const wp1 = store.waypoints[segIdx];
  const wp2 = store.waypoints[segIdx + 1];
  const lat = wp1.lat + (wp2.lat - wp1.lat) * segProgress;
  const lng = wp1.lng + (wp2.lng - wp1.lng) * segProgress;

  if (droneMarker) {
    droneMarker.setLatLng([lat, lng]);
  } else {
    droneMarker = L.circleMarker([lat, lng], {
      radius: 10,
      color: '#fbbf24',
      fillColor: '#f59e0b',
      fillOpacity: 1,
      weight: 3,
    }).addTo(map);
  }
}

watch(() => store.waypoints, () => {
  drawWaypoints();
  drawRoute();
}, { deep: true });

watch(() => store.noFlyZones.length, drawNoFlyZones);
watch(() => store.simProgress, drawSimDrone);

onMounted(() => {
  nextTick(initMap);
});

onUnmounted(() => {
  if (map) {
    map.remove();
    map = null;
  }
  delete window.__droneRemoveWaypoint;
  delete window.__droneToggleLock;
});

function toggleAddMode() {
  addMode.value = !addMode.value;
}

function handlePlanRoute() {
  if (store.waypoints.length < 2) return;
  const first = store.waypoints[0];
  const last = store.waypoints[store.waypoints.length - 1];
  store.planRoute([first.lat, first.lng], [last.lat, last.lng]);
}

function handleReplanWithLocks() {
  store.replanRoutePreservingLocks();
}

const hasLockedWaypoints = ref(false);
watch(
  () => store.waypoints,
  () => {
    hasLockedWaypoints.value = store.waypoints.some((w) => w.locked);
  },
  { deep: true, immediate: true }
);
</script>

<template>
  <div class="relative w-full h-full">
    <div ref="mapContainer" class="w-full h-full rounded-lg" />
    <div class="absolute top-2 right-2 z-[1000] flex flex-col gap-1">
      <button
        @click="toggleAddMode"
        :class="addMode ? 'bg-blue-600 text-white' : 'bg-gray-800 text-gray-300'"
        class="px-3 py-1 rounded text-xs font-medium shadow hover:opacity-90 transition"
      >
        {{ addMode ? '✦ 添加模式' : '○ 点击添加' }}
      </button>
      <button
        @click="handlePlanRoute"
        class="px-3 py-1 rounded text-xs font-medium bg-green-700 text-white shadow hover:opacity-90 transition"
      >
        🧭 规划航线
      </button>
      <button
        v-if="store.waypoints.length >= 2"
        @click="handleReplanWithLocks"
        :class="hasLockedWaypoints ? 'bg-amber-600 hover:bg-amber-500' : 'bg-gray-700 hover:bg-gray-600'"
        class="px-3 py-1 rounded text-xs font-medium text-white shadow hover:opacity-90 transition"
        :title="hasLockedWaypoints ? '重新规划，保留已锁定航点' : '重新规划航线'"
      >
        🔒 重新规划{{ hasLockedWaypoints ? '（保留锁定）' : '' }}
      </button>
      <button
        @click="store.clearRoute()"
        class="px-3 py-1 rounded text-xs font-medium bg-red-700 text-white shadow hover:opacity-90 transition"
      >
        🗑 清除
      </button>
    </div>

    <div class="absolute bottom-2 left-2 z-[1000] bg-slate-900/80 backdrop-blur rounded p-2 text-[10px] text-slate-300 space-y-1">
      <div class="flex items-center gap-2">
        <span class="inline-block w-3 h-3 rounded-full bg-blue-400 border-2 border-blue-500"></span>
        <span>普通航点（可拖动）</span>
      </div>
      <div class="flex items-center gap-2">
        <span class="inline-block w-3 h-3 rounded-full bg-amber-400 border-[3px] border-amber-500"></span>
        <span>🔒 锁定航点（位置固定）</span>
      </div>
    </div>
  </div>
</template>

<style scoped>
:deep(.wp-tooltip) {
  background: rgba(30, 41, 59, 0.9);
  color: #e2e8f0;
  border: 1px solid #475569;
  font-size: 10px;
  padding: 1px 4px;
  border-radius: 4px;
}
:deep(.wp-tooltip-locked) {
  background: rgba(146, 64, 14, 0.9);
  color: #fef3c7;
  border: 1px solid #f59e0b;
  font-weight: 600;
}
</style>
