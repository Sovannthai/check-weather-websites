<script setup lang="ts">
import WeatherCard from './components/WeatherCard.vue'
</script>

<template>
  <div class="min-h-screen bg-black p-8">
    <WeatherCard />
  </div>
</template>

<style>
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600&display=swap');
@import url('https://fonts.googleapis.com/icon?family=Material+Icons');

body {
  font-family: 'Inter', sans-serif;
  margin: 0;
  padding: 0;
}
</style>