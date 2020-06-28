<template>
  <div>
    <el-card class="card">
      <div class="info">
        <p class="name">{{ country.name }}</p>
        <p>Capital: {{ country.capital }}</p>
        <p>Population: {{ numberWithCommas(country.population) }} ppl</p>
        <p>Borders: {{ (country.borders.length) ? country.borders : 'None' }}</p>
      </div>
      <div class="buttonContainer">
        <el-button
          class="button"
          icon="el-icon-zoom-in"
          type="primary"
          circle
          @click="flyToCountry"
        ></el-button>
      </div>
    </el-card>
  </div>
</template>

<script>
export default {
  name: "countryCard",
  props: {
    country: Object
  },
  methods: {
    /**
     * @param {Number} number to format
     * @returns {String} formatted with commas
     */
    // * Duplicate code with App.vue, should ideally be in a mixin
    numberWithCommas(number) {
      return number.toString().replace(/\B(?=(\d{3})+(?!\d))/g, ",");
    },
    flyToCountry() {
      this.$emit("flyToLocation");
    }
  }
};
</script>

<style>
.el-card__body {
  display: flex;
  flex-direction: row;
}

.buttonContainer {
  /* border: 1px solid green; */
  flex: 1;
}

.button {
  float: right;
}
</style>
