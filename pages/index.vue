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
                <span class="font-bold">{{ formatDistance(distance) }}</span>
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
const userPosition = ref<TrackedPoint>();
const path = ref<TrackedPoint[]>([]);
const zoom = ref<number>(17);
const isTracking = ref<boolean>(false);
let watchId: number | null = null;
let watchIdDev: NodeJS.Timeout | null = null;
let timeId: NodeJS.Timeout | null = null;
const time = ref<number>(0);

const devModeStartPoint: TrackedPoint = {
    lat: 46.038206977649395,
    lng: 4.120474065622551,
    timestamp: Date.now()
};

const tileLayerUrl = 'https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png';
const tileLayerAttribution = '&copy; OpenStreetMap contributors';

const distance = computed(() => {
    if (import.meta.client) {
        return calculateDistance(path.value);
    }

    return 0;
});

type TrackedPoint = {
    lat: number;
    lng: number;
    timestamp: number;
}

type TrackedPath = {
    id: string;
    name?: string;
    date: number; // timestamp
    points: TrackedPoint[];
}

onMounted(() => {
    getCurrentPosition(false); // Get the current position on mount
});

const getCurrentPosition = (devMode: boolean = false) => {
    if (!navigator.geolocation) return alert('Geolocation not supported');

    navigator.geolocation.getCurrentPosition((pos) => {
        const point: TrackedPoint = {
            lat: pos.coords.latitude,
            lng: pos.coords.longitude,
            timestamp: Date.now()
        };

        if (devMode) {
            userPosition.value = devModeStartPoint;
            path.value = [
                devModeStartPoint
            ];
        } else {
            userPosition.value = point;
            path.value = [...path.value, point];
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
            const coords: TrackedPoint = {
                lat: devModeStartPoint.lat + Math.random() * 0.0012,
                lng: devModeStartPoint.lng + Math.random() * 0.0012,
                timestamp: Date.now()
            };

            userPosition.value = coords;
            path.value = [...path.value, coords];
        }, 1000);
        return;
    } else {
        watchId = navigator.geolocation.watchPosition((position) => {
            const point: TrackedPoint = {
                lat: position.coords.latitude,
                lng: position.coords.longitude,
                timestamp: Date.now()
            };

            userPosition.value = point;
            path.value = [...path.value, point];
        }, (error) => {
            console.error("Error getting location: ", error);
        },
            {
                enableHighAccuracy: true,
                maximumAge: 0,
                timeout: 5000
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

    savePathToStorage(path.value);
}

const savePathToStorage = (path: TrackedPoint[]) => {
    const existing = JSON.parse(localStorage.getItem('trackedPaths') || '[]') as TrackedPath[];
    const newPath: TrackedPath = {
        id: crypto.randomUUID(),
        date: Date.now(),
        points: path,
    };

    try {
        localStorage.setItem('trackedPaths', JSON.stringify([...existing, newPath]));
    } catch (error) {
        console.error('Error saving path to localStorage', error);
    }
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

const calculateDistance = (path: TrackedPoint[]) => {
    let distance = 0;
    for (let i = 0; i < path.length - 1; i++) {
        const lat1 = path[i].lat;
        const lon1 = path[i].lng;
        const lat2 = path[i + 1].lat;
        const lon2 = path[i + 1].lng;

        const R = 6371e3; // metres
        const φ1 = lat1 * Math.PI / 180; // φ in radians
        const φ2 = lat2 * Math.PI / 180; // φ in radians
        const Δφ = (lat2 - lat1) * Math.PI / 180; // in radians
        const Δλ = (lon2 - lon1) * Math.PI / 180; // in radians

        const a = Math.sin(Δφ / 2) * Math.sin(Δφ / 2) +
            Math.cos(φ1) * Math.cos(φ2) *
            Math.sin(Δλ / 2) * Math.sin(Δλ / 2);
        const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));

        distance += R * c; // in metres
    }
    return distance;
}

const formatDistance = (distance: number) => {
    if (distance < 1000) {
        return `${Math.round(distance)} m`;
    } else {
        return `${(distance / 1000).toFixed(2)} km`;
    }
}
</script>