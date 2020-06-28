/* eslint-disable no-console */
<template>
  <div id="app">
    <div class="wrapper-col">
      <div class="header">
        <el-checkbox type="checkbox" v-model="sortByPop" label="Sort by Population" border />
        <el-input v-model="filter" placeholder="filter countries" class="filterBox" clearable></el-input>
        <el-button @click="resizeMap">Resize Map</el-button>
        <el-button>Show Vessel Tracks [NYI]</el-button>
      </div>
      <div class="wrapper-row">
        <div class="aside">
          <p>Average pop: {{ numberWithCommas(averagePop) }} ppl</p>
          <p v-for="(count, i) in countryList" :key="i">
            <country-card :country="count" @flyToLocation="flyToCountry(count)" />
          </p>
        </div>
        <div class="main" id="MapContainer"></div>
      </div>
    </div>
  </div>
</template>

<script>
// Modules/Components requirements
import CountryCard from "./components/countryCard.vue";
import Papa from "papaparse";
import axios from "axios";
import { schemeCategory10 } from "d3-scale-chromatic";

// * STATIC DATA
const urlCountries = "https://restcountries.eu/rest/v2/all";

const urlVessels =
  "https://kpler.github.io/kp-recruitment/e0e0648191289bb0cf43090d052121c58f0af8f5/data-fs-exercise.csv";

const mapboxgl = require("mapbox-gl/dist/mapbox-gl.js");
mapboxgl.accessToken =
  "pk.eyJ1IjoibWFydGluY2FybGVzIiwiYSI6ImNrNGJzZDJwYzBibHoza3Bpb2JiNnN2cWEifQ.5r0EGkw-vnQn1Gmwy7OQrQ";

export default {
  name: "app",
  components: {
    CountryCard
  },
  data() {
    return {
      countries: [],
      vessels: [],
      map: undefined,
      markers: [],
      filter: "",
      sortByPop: false
    };
  },
  computed: {
    // Array for template cards
    countryList() {
      return this.sortByPop ? this.countriesByPop : this.filteredCountries;
    },
    // Filter the countries based on user input
    filteredCountries() {
      return this.countries.filter(count => {
        return count.name.toLowerCase().includes(this.filter.toLowerCase());
      });
    },
    // Compute array of countries sorted by population
    countriesByPop() {
      return [...this.filteredCountries].sort(
        (a, b) => b.population - a.population
      );
    },
    // Compute average population of filtered countries
    averagePop() {
      let sum = 0;
      this.filteredCountries.forEach(ct => {
        sum += ct.population;
      });
      return Math.round(sum / this.filteredCountries.length);
    },
    // Return unique array of vessel IDs
    vesselIDs() {
      return [
        ...new Set(this.vessels.map(ves => ves.vessel_id).filter(a => a))
      ];
    }
  },
  mounted() {
    this.initializeMap();
    // To ensure data is retrieved before drawing it
    Promise.all([this.parseVesselCSV(), this.getCountries()]).then(() => {
      this.drawAllTracks();
      this.addMarkers(this.countries);
    });
  },
  watch: {
    filter() {
      // 1. Clear all markers
      this.clearMarkers();
      // 2. Add new markers
      this.addMarkers(this.filteredCountries);
    }
  },
  methods: {
    /******************************
     **       DATA RETRIEVAL      *
     *****************************/
    // GET the countries from API and store in "countries"
    async getCountries() {
      try {
        const response = await axios.get(urlCountries);
        this.countries = response.data;
      } catch (error) {
        // console.log('error: ', error)
      }
    },
    // Parse the CSV file and store result in "vessels"
    parseVesselCSV() {
      return new Promise((resolve, reject) => {
        Papa.parse(urlVessels, {
          header: true,
          transformHeader: header => {
            // Remove empty space
            return header.trim();
          },
          download: true,
          error: err => {
            reject(err);
          },
          complete: results => {
            this.vessels = this.reorderCoords(results.data);
            resolve();
          }
        });
      });
    },
    /******************************
     **       MAP MANAGEMENT      *
     *****************************/
    initializeMap() {
      // Create the map
      this.map = new mapboxgl.Map({
        container: "MapContainer",
        zoom: 1,
        style: "mapbox://styles/mapbox/streets-v11"
      });
      // Add some navigation control to it
      const nav = new mapboxgl.NavigationControl({
        showCompass: true
      });
      this.map.addControl(nav, "top-left");
    },
    // Zoom out
    resizeMap() {
      this.map.setZoom(1);
    },
    /**
     * Goal: Focus on a country location
     * @param {Object} country
     */
    flyToCountry(country) {
      this.map.flyTo({
        center: this.reverseLatLng(country.latlng),
        zoom: 5
      });
    },
    /******************************
     **           MARKERS         *
     *****************************/
    /**
     * Goal: add Markers for specified countries
     * @param {Array} Countries to mark on the map
     */
    addMarkers(countries) {
      countries.forEach(count => {
        if (this.areLatLngValid(count.latlng)) {
          const marker = new mapboxgl.Marker().setLngLat(
            this.reverseLatLng(count.latlng)
          );
          this.markers.push(marker);
          marker.addTo(this.map);
        }
      });
    },
    // Remove all markers from the map
    clearMarkers() {
      this.markers.forEach(mk => {
        mk.remove();
      });
      this.markers = [];
    },
    /******************************
     **        VESSEL TRACKS      *
     *****************************/
    // Draw each vessel track
    drawAllTracks() {
      this.vesselIDs.forEach((id, i) => {
        this.drawVesselTrack(id, i);
      });
    },
    /**
     * @param {String} vessel_id
     * @param {Number} index for color
     */
    drawVesselTrack(shipId, index) {
      // Create route obj
      const route = {
        type: "FeatureCollection",
        features: [
          {
            type: "Feature",
            geometry: {
              type: "LineString",
              coordinates: this.sortCoordsByDate(shipId)
            }
          }
        ]
      };
      // Add Source to map
      this.map.addSource(shipId, {
        type: "geojson",
        data: route
      });
      // Add layer to map
      this.map.addLayer({
        id: shipId,
        source: shipId,
        type: "line",
        paint: {
          "line-width": 2,
          "line-color": schemeCategory10[index]
        }
      });
    },
    /******************************
     **        CONVENIENCE        *
     *****************************/
    /**
     * @param {String} ship id
     * @returns {Array} of coordinate pairs
     */
    sortCoordsByDate(shipId) {
      const shipData = this.vessels.filter(v => v.vessel_id === shipId);
      return shipData.map(p => {
        return [p.longitude, p.latitude];
      });
    },
    /**
     * @param {Array} parsed csv file
     * @returns {Array} data sorted by time
     */
    reorderCoords(vessels) {
      return vessels.sort((a, b) => {
        // Convert strings to date
        const dateA = new Date(a.received_time_utc);
        const dateB = new Date(b.received_time_utc);
        return dateA > dateB ? -1 : dateA < dateB ? 1 : 0;
      });
    },
    /**
     * @param {Array} [lat, lng]
     * @returns {Boolean}
     */
    areLatLngValid(coords) {
      // Check for missing coordinates
      if (!coords.length) return false;
      // Check for coordinates domain
      if (coords[0] > 90 || coords[0] < -90) return false;
      return true;
    },
    /**
     * @param {Array} [lat, lng]
     * @returns {Array} [lng, lat]
     */
    reverseLatLng(latlng) {
      return [...latlng].reverse();
    },
    /**
     * @param {Number} number to format
     * @returns {String} formatted with commas
     */
    // * Duplicate code with App.vue, should ideally be in a mixin
    numberWithCommas(number) {
      return number.toString().replace(/\B(?=(\d{3})+(?!\d))/g, ",");
    },
  }
}
</script>

<style src="element-ui/lib/theme-chalk/index.css"></style>
<style src="mapbox-gl/dist/mapbox-gl.css"></style>

<style>
#app {
  font-family: "Avenir", Helvetica, Arial, sans-serif;
  overflow: hidden;
}

p {
  margin: 6px;
  font-size: 14px;
}

.wrapper-col {
  display: flex;
  flex-direction: column;
}

.wrapper-row {
  display: flex;
  flex-direction: row;
}

.aside {
  width: 30%;
  height: 90vh;
  overflow-y: auto;
}

.main {
  height: 90vh;
  width: 100%;
}

.name {
  font-weight: bold;
}

.filterBox {
  width: 35vh;
  margin: 10px 0px;
}
</style>
