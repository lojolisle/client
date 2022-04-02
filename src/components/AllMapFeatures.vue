<template>
  <div
    class="w-full md:w-auto absolute md:top-[40px] md:left-[60px] z-[2] flex gap-4 px-6 py-8 md:px-0 md:py-0 bg-transparent"
  >
    <!-- MapBox Search API Section-->
    <div class="relative flex-1 md:min-w-[350px]">
      <!-- Map Search Input -->
      <input
        class="pl-9 pr-4 py-3 text-[14px] focus:outline-none w-full shadow-md rounded-md"
        type="text"
        placeholder="Search..."
        v-model="searchQuery"
        @input="search"
        @focus="$emit('toggleSResults')"
      />
      <!-- Map Search Icon -->
      <div class="absolute top-0 left-[8px] h-full flex items-center">
        <i class="fas fa-search"></i>
      </div>
      <!-- Map Search Results -->
      <div class="absolute mt-[8px] w-full">
        <!-- Search Query Results -->
         <div
            v-if="searchQuery && searchResults"
            class="h-[200px] overflow-scroll bg-white rounded-md"
         >
            <!-- Loading Spinner only show whensearch result is null-->
            <LoadingSpinner v-if="!searchData"/>
            <div v-else>
               <div class="px-4 py-2 flex gap-x-2 cursor-pointer hover:bg-slate-600 hover:text-white"
               v-for="(result, index) in searchData"
               :key="index"
               @click="selectResult(result)"
               >
                  <i class="fas fa-map-marker-alt"></i>
                  <p class="text-xs">{{result.place_name_en}}</p>
               </div>
            </div>
        </div>

        <!-- Selected Search Result Section -->
        <div v-if="selectedResult" class="mt-[8px] px-4 py-3 bg-white rounded-md">
          <i @click="removeResult" class="flex justify-end far fa-times-circle"></i>
          <h1 class="text-lg">{{ selectedResult.text }}</h1>
          <p class="text-xs mb-1">
            {{ selectedResult.properties.address }}
          </p>
          <p class="text-xs">Category: {{ selectedResult.properties.category }}</p>
          <br/>
          <hr>
          <div v-if="weatherQueryResult">
            <h1>Current Weather at {{selectedResult.city}}</h1>
            <img :src="weatherIcon">
            <h1 class="text-lg">{{ weatherData.weather[0].description }}</h1>
            <p class="text-xs mb-1">
              Temperature: {{ weatherData.main.temp }} &deg; C, Feels lIke {{ weatherData.main.feels_like }}
            </p>
            <p class="text-xs mb-1">   
              Wind: {{ weatherData.wind.speed }}, Humidity: {{ weatherData.main.humidity }}
            </p> 
          </div>
        </div>
      </div>
    </div>

    <!-- Geolocation API section -->
    <div
      class="px-4 bg-white flex items-center shadow-md rounded-md min-h-[45px]"
      @click="$emit('getGeolocation')"
      :class="{ 'bg-slate-600': coords }"
    >
      <i
        class="fas fa-location-arrow 'text-slate-600' text-[18px]"
        :class="{ 'text-white': coords, 'animate-pulse': fetchCoords }"
      ></i>
    </div>
  </div>
</template>

<script>
import { ref } from "vue";
import axios from "axios";
import LoadingSpinner from "./LoadingSpinner.vue";

export default {
  props: ["fetchCoords", "coords", "searchResults"],
  components: { LoadingSpinner },
  setup(props, { emit }) {
    const searchQuery = ref(null);
    const searchData = ref(null);
    const queryTimeout = ref(null);
    const selectedResult = ref(null);

    // data for weather results storage
    const weatherQueryResult = ref(null);
    const weatherData = ref(null);
    const weatherIcon = ref(null);

    // function to call ap1/search end point
    const search = () => {
      clearTimeout(queryTimeout.value);
      // reset data on a new search
      searchData.value = null;
      queryTimeout.value = setTimeout(async () => {
        // Only make search, if there is value in query input
        if (searchQuery.value !== "") {
          const params = new URLSearchParams({
            fuzzyMatch: true,
            language: "en",
            limit: 10,
            proximity: props.coords ? `${props.coords.lng},${props.coords.lat}` : "0,0",
          });
          const getData = await axios.get(`http://localhost:3000/api/search/${searchQuery.value}?${params}`);
          searchData.value = getData.data.features;
          console.log(searchData.value)
          
        }
      }, 750);
    };

    const selectResult = (result) => {
      selectedResult.value = result;
      emit("plotResult", result.geometry);
      // call weather API from selected results coordinates
      getWeatherDetails(result.geometry);
    };

    const removeResult = () => {
      selectedResult.value = null;
      emit("removeResult");
    };

    const getWeatherDetails = async (geometry) => {
      //  const params = new URLSearchParams({
      //       lat: geometry.coordinates[1],
      //       lon: geometry.coordinates[0],
      //     });
        //weatherQueryResult.value = await axios.get(`http://localhost:3000/api/weather/weather?${params}`);
         weatherQueryResult.value = await axios.get(`https://api.openweathermap.org/data/2.5/weather?lat=${geometry.coordinates[1]}&lon=${geometry.coordinates[0]}&appid=91624a5b1a7de946621c04e9fbe3ffe8&units=metric`);
         weatherData.value = weatherQueryResult.value.data;
         weatherIcon.value = `http://openweathermap.org/img/wn/${weatherData.value.weather[0].icon }@2x.png`;
    }

    return {
      searchQuery,
      search,
      searchData,
      selectResult,
      selectedResult,
      removeResult,
      weatherQueryResult,
      weatherData,
      weatherIcon
    };
  },
};
</script>