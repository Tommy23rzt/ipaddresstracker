<template>
  <div class="flex flex-col h-screen max-h-screen">
    <!-- Search / Results -->
    <div
      class="z-20 flex justify-center relative bg-hero-pattern bg-cover px-4 pt-8 pb-32"
    >
      <!-- Search Input -->
      <div class="w-full max-w-screen-sm">
        <h1 class="text-white text-center text-3xl pb-4">IP Address Tracker</h1>
        <form class="flex" @submit.prevent="getIpInfo">
          <input
            v-model="queryIp"
            class="flex-1 py-3 px-2 rounded-tl-md rounded-bl-md focus:outline-none"
            type="text"
            aria-label="IP address"
            placeholder="Search for any IP address or leave empty to get your ip info"
          />
          <button
            type="submit"
            :disabled="loading"
            aria-label="Search IP"
            class="cursor-pointer
          bg-black
          text-white
          px-4
          rounded-tr-md
          rounded-br-md
          flex
          items-center
          disabled:opacity-50
          fas fa-chevron-right"
          ></button>
        </form>
        <p v-if="errorMessage" class="text-red-200 text-sm mt-2">
          {{ errorMessage }}
        </p>
        <div class="mt-3 flex justify-center">
          <button
            type="button"
            :disabled="locating"
            class="inline-flex items-center gap-2 rounded-md bg-white/90 px-4 py-2 text-sm font-medium text-slate-800 shadow hover:bg-white disabled:opacity-50"
            @click="locateUser"
          >
            <i class="fas fa-location-crosshairs"></i>
            {{ locating ? "Locating..." : "Use my location" }}
          </button>
        </div>
      </div>
      <!-- IP Info -->
      <IPInfo v-if="ipInfo" v-bind:ipInfo="ipInfo" />
    </div>

    <!-- Map -->
    <div id="mapid" class="h-full z-10"></div>
  </div>
</template>

<script>
import IPInfo from "../components/IPInfo.vue";
import leaflet from "leaflet";
import { onMounted, onUnmounted, ref } from "vue";
import axios from "axios";

const API_URL = "https://ipwho.is/";

export default {
  name: "HomeView",
  components: { IPInfo },
  setup() {
    // create map variable
    let mymap;
    let marker;
    let locationMarker;
    let locationCircle;
    // data
    const queryIp = ref("");
    const ipInfo = ref(null);
    const errorMessage = ref("");
    const loading = ref(false);
    const locating = ref(false);
    // mounted lifecycle hook, creates the map
    onMounted(() => {
      mymap = leaflet.map("mapid").setView([42.5145, -83.0147], 9);
      leaflet
        .tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
          maxZoom: 19,
          attribution: "© OpenStreetMap",
        })
        .addTo(mymap);
    });
    // removes the map when the view is destroyed
    onUnmounted(() => {
      if (mymap) {
        mymap.remove();
        mymap = null;
      }
    });
    // gets ip information from API
    const getIpInfo = async () => {
      errorMessage.value = "";
      loading.value = true;
      try {
        const ip = queryIp.value.trim();
        const { data: result } = await axios.get(
          `${API_URL}${encodeURIComponent(ip)}`
        );
        if (!result.success) {
          throw new Error(result.message || "Invalid IP address");
        }
        ipInfo.value = {
          address: result.ip,
          state: [result.city, result.region, result.country]
            .filter(Boolean)
            .join(", "),
          timezone: result.timezone ? result.timezone.utc : "",
          isp: result.connection ? result.connection.isp : "",
          lat: result.latitude,
          lng: result.longitude,
        };
        if (marker) {
          marker.setLatLng([ipInfo.value.lat, ipInfo.value.lng]);
        } else {
          marker = leaflet
            .marker([ipInfo.value.lat, ipInfo.value.lng])
            .addTo(mymap);
        }
        mymap.setView([ipInfo.value.lat, ipInfo.value.lng], 13);
      } catch (err) {
        errorMessage.value =
          (err.response && err.response.data && err.response.data.message) ||
          err.message;
      } finally {
        loading.value = false;
      }
    };
    // uses the browser geolocation to move the map to the real position
    const locateUser = () => {
      errorMessage.value = "";
      if (!navigator.geolocation) {
        errorMessage.value = "Geolocalizzazione non supportata dal browser.";
        return;
      }
      locating.value = true;
      navigator.geolocation.getCurrentPosition(
        ({ coords }) => {
          locating.value = false;
          if (!mymap) return;
          const position = [coords.latitude, coords.longitude];
          if (locationMarker) {
            locationMarker.setLatLng(position);
          } else {
            locationMarker = leaflet
              .marker(position)
              .addTo(mymap)
              .bindPopup("La tua posizione (approssimativa)");
          }
          if (locationCircle) {
            locationCircle.setLatLng(position).setRadius(coords.accuracy);
          } else {
            locationCircle = leaflet
              .circle(position, {
                radius: coords.accuracy,
                color: "#2563eb",
                fillColor: "#3b82f6",
                fillOpacity: 0.15,
              })
              .addTo(mymap);
          }
          mymap.setView(position, 15);
        },
        (err) => {
          locating.value = false;
          const messages = {
            1: "Permesso di geolocalizzazione negato.",
            2: "Posizione non disponibile.",
            3: "Timeout durante la geolocalizzazione.",
          };
          errorMessage.value =
            messages[err.code] || "Impossibile ottenere la posizione.";
        },
        { enableHighAccuracy: true, timeout: 10000, maximumAge: 60000 }
      );
    };
    return {
      queryIp,
      ipInfo,
      getIpInfo,
      locateUser,
      errorMessage,
      loading,
      locating,
    };
  },
};
</script>
