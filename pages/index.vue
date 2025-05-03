<template>
    <LMap v-if="userPosition" ref="map" style="height: 60vh; width: 100%" :zoom="zoom" :center="userPosition"
        :use-global-leaflet="false">
        <LTileLayer :url="tileLayerUrl" :attribution="tileLayerAttribution" layer-type="base" name="OpenStreetMap" />
        <LMarker :lat-lng="userPosition" />
        <LPolyline :lat-lngs="path" color="blue" :weight="5" :opacity="0.7" />
    </LMap>

    <div v-if="isTracking">
        <div class="flex flex-row justify-center items-center">
            <div class="border border-gray-300 rounded-md p-2 m-2 w-1/2 text-center">
                <strong class="font-normal">Temps :</strong> <br />
                <span class="font-bold">{{ getTimeInHours() }}</span>

            </div>
            <div class="border border-gray-300 rounded-md p-2 m-2 w-1/2 text-center">
                <strong class="font-medium">Distance :</strong> <br />
                <span class="font-bold">0m</span>
            </div>
        </div>

        <div class="flex flex-row justify-center items-center">
            <div class="border border-gray-300 rounded-md p-2 m-2 w-1/2 text-center">
                <strong class="font-medium">Vitesse actuelle :</strong> <br />
                <span class="font-bold">0 km/h</span>
            </div>
            <div class="border border-gray-300 rounded-md p-2 m-2 w-1/2 text-center">
                <strong class="font-medium">Vitesse moyenne :</strong> <br />
                <span class="font-bold">0 km/h</span>
            </div>
        </div>

        <div class="text-center">
            <button class="py-1 pr-2 pl-2 m-2 bg-cyan-600 text-white rounded-sm" @click="stopTracking">Arrêter
                l'enregistrement</button>
        </div>
    </div>
    <div v-else>
        <div class="flex flex-row justify-center items-center">
            <button @click="startTracking(false)" class="py-1 pr-2 pl-2 m-2 bg-cyan-600 text-white rounded-sm">
                Enregistrer le parcours
            </button>

            <button @click="startTracking(true)" class="py-1 pr-2 pl-2 m-2 bg-cyan-600 text-white rounded-sm">
                Enregistrer le parcours (dev)
            </button>
        </div>
    </div>
</template>

<script setup lang="ts">
const userPosition = ref<[number, number]>();
const path = ref<[number, number][]>([]);
const zoom = ref<number>(17);
const isTracking = ref<boolean>(false);
let watchId: number | null = null;
let watchIdDev: NodeJS.Timeout | null = null;
let timeId: NodeJS.Timeout | null = null;
const time = ref<number>(0);

const devModeStartPoint: [number, number] = [46.038206977649395, 4.120474065622551];

const tileLayerUrl = 'https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png';
const tileLayerAttribution = '&copy; OpenStreetMap contributors';

onMounted(() => {
    getCurrentPosition(false); // Get the current position on mount
});

const getCurrentPosition = (devMode: boolean = false) => {
    if (!navigator.geolocation) return alert('Geolocation not supported');

    navigator.geolocation.getCurrentPosition((pos) => {
        const coords: [number, number] = [pos.coords.latitude, pos.coords.longitude];

        if (devMode) {
            userPosition.value = devModeStartPoint;
            path.value = [
                devModeStartPoint
            ];
        } else {
            userPosition.value = coords;
            path.value.push(coords);
        }
    }, (err) => {
        console.error('GPS error', err);
    },
        {
            enableHighAccuracy: true,
            maximumAge: 1000,
            timeout: 10000
        }
    );
}

const startTracking = (devMode: boolean = false) => {
    if (!navigator.geolocation) {
        console.log("Geolocation is not supported by this browser.");
        return;
    }

    isTracking.value = true;
    time.value = 0;
    path.value = [];
    getCurrentPosition(devMode); // Get the current position when starting tracking
    requestWakeLock();

    timeId = setInterval(() => {
        time.value += 1;
    }, 1000);

    if (devMode) {
        userPosition.value = devModeStartPoint;
        path.value = [
            devModeStartPoint
        ];
        watchIdDev = setInterval(() => {
            const coords: [number, number] = [path.value[0][0] + Math.random() * 0.0012, path.value[0][1] + Math.random() * 0.0012];
            userPosition.value = coords;
            path.value = [...path.value, coords];
        }, 1000);
        return;
    } else {
        watchId = navigator.geolocation.watchPosition((position) => {
            const coords: [number, number] = [position.coords.latitude, position.coords.longitude];

            if (!userPosition.value || haversineDistance(userPosition.value, coords) > 10) {
                // Only update the path if the user has moved more than 10 meters
                userPosition.value = coords;
                path.value = [...path.value, coords];
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
}

const stopTracking = () => {
    if (watchId) {
        navigator.geolocation.clearWatch(watchId);
        watchId = null;
    }
    if (watchIdDev) {
        clearInterval(watchIdDev);
        watchIdDev = null;
    }
    if (timeId) {
        clearInterval(timeId);
        timeId = null;
    }
    isTracking.value = false;

    try {
        localStorage.setItem('lastTrackedPath', JSON.stringify(path.value));
    } catch (error) {
        console.error('Error saving path to localStorage', error);
    }
}

const haversineDistance = (a: [number, number], b: [number, number]) => {
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

const requestWakeLock = async () => {
    let wakeLock: WakeLockSentinel | null = null;
    try {
        wakeLock = await navigator.wakeLock.request('screen');
        console.log('Wake Lock is active!');
    } catch (err: any) {
        console.error(`${err.name}, ${err.message}`);
    }
}

const getTimeInHours = () => {
    const hours = Math.floor(time.value / 3600);
    const minutes = Math.floor((time.value % 3600) / 60);
    const seconds = time.value % 60;
    return `${hours}h ${minutes}m ${seconds}s`;
}
</script>