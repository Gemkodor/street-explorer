<template>
    <LMap v-if="userPosition" ref="map" style="height: 70vh; width: 100%" :zoom="zoom" :center="userPosition"
        :use-global-leaflet="false">
        <LTileLayer :url="tileLayerUrl" :attribution="tileLayerAttribution" layer-type="base" name="OpenStreetMap" />
        <LMarker :lat-lng="userPosition" />
        <LPolyline :lat-lngs="path" color="blue" :weight="5" :opacity="0.7" />
    </LMap>

    <div v-if="isTracking">
        Enregistrement en cours...<br />

        <button @click="stopTracking">Arrêter l'enregistrement</button>

        <div v-for="coords in path">
            <p>Latitude: {{ coords[0] }}, Longitude: {{ coords[1] }}</p>
        </div>
    </div>
    <div v-else>
        <button @click="startTracking" class="fixed top-4 left-4 z-50 p-2 bg-blue-600 text-white rounded">
            Enregistrer le parcours
        </button>
    </div>
    <p>

    </p>
</template>

<script setup lang="ts">
const userPosition = ref(null);
const path = ref([]);
const zoom = ref(17);
const isTracking = ref(false);
let watchId = null;

const tileLayerUrl = 'https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png';
const tileLayerAttribution = '&copy; OpenStreetMap contributors';

onMounted(() => {
    if (!navigator.geolocation) return alert('Geolocation not supported');

    navigator.geolocation.getCurrentPosition((pos) => {
        const coords = [pos.coords.latitude, pos.coords.longitude];
        userPosition.value = coords;
        path.value.push(coords);
    }, (err) => {
        console.error('GPS error', err);
    },
        {
            enableHighAccuracy: true,
            maximumAge: 1000,
            timeout: 10000
        }
    );
});

const startTracking = () => {
    if (!navigator.geolocation) {
        console.log("Geolocation is not supported by this browser.");
        return;
    }

    isTracking.value = true;
    requestWakeLock();

    watchId = navigator.geolocation.watchPosition((position) => {
        const coords = [position.coords.latitude, position.coords.longitude];

        if (!userPosition.value || haversineDistance(userPosition.value, coords) > 10) {
            // Only update the path if the user has moved more than 10 meters
            userPosition.value = coords;
            path.value.push(coords);
        }
    }, (error) => {
        console.error("Error getting location: ", error);
    },
        {
            enableHighAccuracy: true,
            maximumAge: 1000,
            timeout: 10000
        }
    );
}

const stopTracking = () => {
    if (watchId) {
        navigator.geolocation.clearWatch(watchId);
        watchId = null;
    }
    isTracking.value = false;

    try {
        localStorage.setItem('lastTrackedPath', JSON.stringify(path.value));
    } catch (error) {
        console.error('Error saving path to localStorage', error);
    }
}

const haversineDistance = (a, b) => {
    const R = 6371e3; // metres
    const φ1 = a[0] * Math.PI / 180; // φ in radians
    const φ2 = b[0] * Math.PI / 180; // φ in radians
    const Δφ = (b[0] - a[0]) * Math.PI / 180; // difference in latitude in radians
    const Δλ = (b[1] - a[1]) * Math.PI / 180; // difference in longitude in radians

    const d = Math.sin(Δφ / 2) * Math.sin(Δφ / 2) +
        Math.cos(φ1) * Math.cos(φ2) *
        Math.sin(Δλ / 2) * Math.sin(Δλ / 2);
    return R * (2 * Math.atan2(Math.sqrt(d), Math.sqrt(1 - d))); // in metres
}

const requestWakeLock = () => {
    let wakeLock: WakeLockSentinel | null = null;
    try {
        wakeLock = await navigator.wakeLock.request('screen');
        console.log('Wake Lock is active!');
    } catch (err) {
        console.error(`${err.name}, ${err.message}`);
    }
}
</script>