<template>
  <div class="h-screen relative">
    <GeoErrorModal 
      @closeGeoError="closeGeoError" 
      v-if="geoError" :geoErrorMessage="geoErrorMessage" 
    />

    <AllMapFeatures 
      @getGeoLocation="getGeoLocation" 
      @plotResult="plotResult"
      @toggleSResults="toggleSResults"
      @removeResult="removeResult"
      :coords="coords" 
      :fetchCoords="fetchCoords"
      :searchResults="searchResults"

    />

    <div id="leafletMap" class="h-full z-[1]"></div>
  </div>
</template>

<script>
import leaflet from 'leaflet';
import { onMounted, ref } from "vue";
import GeoErrorModal from "@/components/GeoErrorModal.vue";
import AllMapFeatures from "@/components/AllMapFeatures.vue";


export default {
  name: 'HomeView',
  components: { GeoErrorModal, AllMapFeatures },
  setup() {
    let leafletMap;
    onMounted(() => {
      // initialise map fom leaflet library
      leafletMap = leaflet.map('leafletMap').setView([51.505, -0.09], 13);
    
      // adding map tile layers
      leaflet.tileLayer(
        `https://api.mapbox.com/styles/v1/{id}/tiles/{z}/{x}/{y}?access_token=${process.env.VUE_APP_API_KEY}`, 
        {
          attribution: 'Map data &copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors, Imagery © <a href="https://www.mapbox.com/">Mapbox</a>',
          maxZoom: 18,
          id: 'mapbox/streets-v11',
          tileSize: 512,
          zoomOffset: -1,
          accessToken: process.env.VUE_APP_API_KEY
        }).addTo(leafletMap);

        // close search results whne we move around map
        leafletMap.on("moveend", () => {
          closeSearchResults();
        })
        // initiate function in mount hook
        getGeoLocation();

    })

    // store users coordinates
    const coords = ref(null);
    // const used for a little animation on location icon while api loads
    const fetchCoords = ref(null);
    const geoMarker = ref(null);

    const geoError = ref(null);
    const geoErrorMessage = ref(null);

    const getGeoLocation = () => {
      console.log('called---' + coords.value)
      // check if coords has value, then remove users location
      if (coords.value) {
        coords.value =  null;
        sessionStorage.removeItem('coords');
        //also remove geoMarker from map
        leafletMap.removeLayer(geoMarker.value);
        return;
      }

      // first check if we have coords stored in session
      if (sessionStorage.getItem('coords')) {
        console.log('session storage')
        coords.value = JSON.parse(sessionStorage.getItem("coords"));
        console.log(coords.value)
        plotGeoLocation(coords.value);
        return;
      }

      /* geolocation api */
      
      // not found then call  to obtain users location
      fetchCoords.value = true;
      // getCurrentPosition except two call back functions
      navigator.geolocation.getCurrentPosition(setCoords, getLocErr)
    };

    const setCoords = (position) => {
      // we have users coordinates in position
      console.log('check me ' + position)
      // first stop fetchCoords since we now have data
      fetchCoords.value = null;
      //store users coord in session storage so we dont need to always call api each time user refreshes page
      const setSessionUserCoords = {
        lat: position.coords.latitude,
        lng: position.coords.longitude
      }
      sessionStorage.setItem('coords', JSON.stringify(setSessionUserCoords));

      // set ref coords value
      coords.value = setSessionUserCoords;

      // next we need to plot this users location on our map
      plotGeoLocation(coords.value);
    }

    const getLocErr = (err) => {
      fetchCoords.value = null;
      geoError.value = true;
      geoErrorMessage.value = err.message;
    }

    const plotGeoLocation = () => {
      // now create a new custom marker
      const customMarker = leaflet.icon({
        iconUrl: require('../assets/red-map-maker.svg'),
        iconSize: [30,30],
      });

      // create new marker with coords from user and custom icons
      geoMarker.value = leaflet
        .marker([coords.value.lat, coords.value.lng], {icon: customMarker})
        .addTo(leafletMap);

      // set our map view to current user location
      leafletMap.setView([coords.value.lat, coords.value.lng], 10);

    };

    // close the error modal
    const closeGeoError = () => {
      geoError.value = null;
      geoErrorMessage.value = null;
    }

    const resultMarker = ref(null);

    const plotResult = (coords) => {
      if(resultMarker.value) {
        leafletMap.removeLayer(resultMarker.value);
      }
      // now create a new custom marker
      const customMarker = leaflet.icon({
        iconUrl: require('../assets/blue-map-marker.svg'),
        iconSize: [30,30],
      });

      // create new marker with coords from user and custom icons
      resultMarker.value = leaflet
        .marker([coords.coordinates[1], coords.coordinates[0]], {icon: customMarker})
        .addTo(leafletMap);

      // set our map view to current user location
      leafletMap.setView([coords.coordinates[1], coords.coordinates[0]], 13);
      closeSearchResults();
    };

    const searchResults = ref(null);
    const toggleSResults = () => {
      searchResults.value = !searchResults.value;
    };

    const closeSearchResults = () => {
      searchResults.value = null;
    };

    const removeResult = () => {
      leafletMap.removeLayer(resultMarker.value);
    }

    return { 
      coords, 
      fetchCoords, 
      geoMarker, 
      closeGeoError, 
      geoError, 
      geoErrorMessage, 
      getGeoLocation,
      plotResult,
      searchResults,
      toggleSResults,
      closeSearchResults,
      removeResult
    };

  },
};
</script>
