<script setup lang="ts">
import { ref, onMounted, watch } from 'vue';

// Type definitions
interface ForecastDay {
  day: string;
  temp: string;
  condition: string;
  icon: string;
}

interface HourForecast {
  time: string;
  temp: string;
}

interface LocationSuggestion {
  name: string;
  region: string;
  country: string;
  fullName: string;
}

// Weather data state
const currentTemp = ref(0);
const currentCondition = ref('');
const currentIcon = ref('');
const wind = ref('');
const humidity = ref('');
const location = ref('');
const date = ref('');
const searchQuery = ref('');
const isLoading = ref(true);
const errorMessage = ref('');

// Suggestions data
const suggestions = ref<LocationSuggestion[]>([]);
const showSuggestions = ref(false);
const isFetchingSuggestions = ref(false);
const suggestionsTimeout = ref<number | null>(null);

// Animation states
const contentLoaded = ref(false);
const activeTab = ref('weekly');

// Forecast data
const forecast = ref<ForecastDay[]>([]);
const hourlyForecast = ref<HourForecast[]>([]);

// Format date
const formatDate = (dateString: string) => {
  const options: Intl.DateTimeFormatOptions = { 
    weekday: 'long', 
    year: 'numeric', 
    month: 'long', 
    day: 'numeric' 
  };
  return new Date(dateString).toLocaleDateString('en-US', options);
};

// Get day of week
const getDayOfWeek = (dateString: string) => {
  const options: Intl.DateTimeFormatOptions = { weekday: 'long' };
  return new Date(dateString).toLocaleDateString('en-US', options);
};

// Sort days to start from Monday
const sortDaysFromMonday = (forecastData: ForecastDay[]) => {
  const dayOrder = ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday'];
  return [...forecastData].sort((a, b) => {
    return dayOrder.indexOf(a.day) - dayOrder.indexOf(b.day);
  });
};

// Fetch suggestions from API
const fetchSuggestions = async (query: string) => {
  if (!query.trim() || query.length < 3) {
    suggestions.value = [];
    showSuggestions.value = false;
    return;
  }
  
  isFetchingSuggestions.value = true;
  
  try {
    const apiKey = 'e007277ac91a4ff88ad83238252602';
    const response = await fetch(`https://api.weatherapi.com/v1/search.json?key=${apiKey}&q=${query}`);
    
    if (!response.ok) {
      throw new Error('Failed to fetch suggestions');
    }
    
    const data = await response.json();
    
    suggestions.value = data.map((item: any) => ({
      name: item.name,
      region: item.region,
      country: item.country,
      fullName: `${item.name}, ${item.country}`
    }));
    
    showSuggestions.value = suggestions.value.length > 0;
  } catch (error) {
    console.error('Error fetching suggestions:', error);
    suggestions.value = [];
    showSuggestions.value = false;
  } finally {
    isFetchingSuggestions.value = false;
  }
};

// Fetch weather data from API
const fetchWeatherData = async (query = 'Siem Reap, Cambodia') => {
  contentLoaded.value = false;
  showSuggestions.value = false;
  isLoading.value = true;
  errorMessage.value = '';
  
  try {
    const apiKey = 'e007277ac91a4ff88ad83238252602';
    const response = await fetch(`https://api.weatherapi.com/v1/forecast.json?key=${apiKey}&q=${query}&days=7&aqi=no&alerts=no`);
    
    if (!response.ok) {
      throw new Error('Failed to fetch weather data');
    }
    
    const data = await response.json();
    
    // Update current weather data
    currentTemp.value = data.current.temp_c;
    currentCondition.value = data.current.condition.text;
    currentIcon.value = `https:${data.current.condition.icon}`;
    wind.value = `${data.current.wind_kph} km/h ${data.current.wind_dir}`;
    humidity.value = `${data.current.humidity}%`;
    location.value = `${data.location.name}, ${data.location.country}`;
    date.value = formatDate(data.location.localtime);
    
    // Update forecast data
    const forecastData: ForecastDay[] = data.forecast.forecastday.map((day: any) => ({
      day: getDayOfWeek(day.date),
      temp: `${Math.round(day.day.avgtemp_c)}°`,
      condition: day.day.condition.text,
      icon: `https:${day.day.condition.icon}`
    }));
    
    // Sort days to start from Monday
    forecast.value = sortDaysFromMonday(forecastData);
    
    // Update hourly forecast for today with 2-hour intervals
    hourlyForecast.value = data.forecast.forecastday[0].hour
      .filter((_hourData: any, index: number) => index % 2 === 0) // Every 2 hours
      .map((hourData: any) => ({
        time: new Date(hourData.time).toLocaleTimeString('en-US', { hour: '2-digit', hour12: true }),
        temp: `${Math.round(hourData.temp_c)}°`
      }))
      .slice(0, 8); // Limit to 8 entries
      
    // Add a slight delay to show animation
    setTimeout(() => {
      contentLoaded.value = true;
    }, 300);
    
  } catch (error) {
    console.error('Error fetching weather data:', error);
    errorMessage.value = 'Failed to load weather data. Please try again.';
  } finally {
    isLoading.value = false;
  }
};

// Handle manual search
const handleSearch = () => {
  if (searchQuery.value.trim().length >= 3) {
    fetchWeatherData(searchQuery.value);
  }
};

// Select suggestion
const selectSuggestion = (suggestion: LocationSuggestion) => {
  searchQuery.value = suggestion.fullName;
  showSuggestions.value = false;
  fetchWeatherData(suggestion.fullName);
};

// Watch for changes in the search query to update suggestions only
watch(searchQuery, (newQuery) => {
  // Clear any existing timeouts
  if (suggestionsTimeout.value) {
    clearTimeout(suggestionsTimeout.value);
  }
  
  // Skip if empty
  if (!newQuery.trim()) {
    showSuggestions.value = false;
    return;
  }
  
  // Set a timeout to fetch suggestions
  suggestionsTimeout.value = window.setTimeout(() => {
    fetchSuggestions(newQuery);
  }, 300); // 300ms delay for suggestions
});

// Handle click outside to close suggestions
const handleClickOutside = (event: MouseEvent) => {
  if (!(event.target as Element).closest('.search-container')) {
    showSuggestions.value = false;
  }
};

// Initialize with default location and event listener
onMounted(() => {
  fetchWeatherData();
  document.addEventListener('click', handleClickOutside);
});
</script>

<template>
  <div class="weather-app min-h-screen bg-gradient-to-br from-[#1a1a2e] to-[#16213e] text-white p-4 md:p-8 flex items-center justify-center">
    <div class="weather-card w-full max-w-6xl backdrop-blur-lg bg-[#1a1a2a]/60 rounded-3xl overflow-hidden shadow-2xl border border-[#ffffff15] p-6 md:p-8">
      <!-- Search Bar with Suggestions -->
      <div class="mb-8 search-container relative">
        <div class="relative rounded-full overflow-hidden shadow-lg transition-all duration-300 hover:shadow-[0_0_15px_rgba(255,255,255,0.2)]">
          <input 
            type="text" 
            v-model="searchQuery"
            @keyup.enter="handleSearch"
            placeholder="Search location..."
            class="w-full bg-[#2a2a3a]/80 text-white px-6 py-4 pr-12 focus:outline-none focus:ring-0 transition-all duration-300"
          />
          <button 
            @click="handleSearch"
            class="absolute right-4 top-1/2 transform -translate-y-1/2 text-gray-400 hover:text-white transition-colors duration-300"
            :class="{ 'pointer-events-none': isLoading }"
          >
            <span class="material-icons">{{ isLoading ? 'refresh' : 'search' }}</span>
          </button>
        </div>
        
        <!-- Suggestions Dropdown -->
        <div 
          v-if="showSuggestions" 
          class="absolute z-10 mt-2 w-full bg-[#2a2a3a]/90 backdrop-blur rounded-xl overflow-hidden shadow-2xl max-h-60 overflow-y-auto border border-[#ffffff15] animate-fade-in"
        >
          <div class="py-2 px-4 text-gray-300 text-xs uppercase font-semibold border-b border-gray-700/50">
            Suggestions
          </div>
          <div 
            v-if="isFetchingSuggestions"
            class="flex items-center justify-center p-4"
          >
            <span class="material-icons animate-spin mr-2">refresh</span>
            <span>Loading suggestions...</span>
          </div>
          <div 
            v-else-if="suggestions.length === 0"
            class="p-4 text-center text-gray-400"
          >
            No locations found
          </div>
          <div 
            v-else
            v-for="suggestion in suggestions" 
            :key="suggestion.fullName"
            @click="selectSuggestion(suggestion)"
            class="px-4 py-3 hover:bg-[#ffffff15] cursor-pointer transition-colors duration-200"
          >
            <div class="font-medium">{{ suggestion.name }}</div>
            <div class="text-sm text-gray-400">{{ suggestion.region }}, {{ suggestion.country }}</div>
          </div>
        </div>
      </div>

      <!-- Error message if any -->
      <div v-if="errorMessage" class="bg-red-500/80 backdrop-blur-sm p-4 rounded-lg mb-6 text-center animate-fade-in">
        {{ errorMessage }}
      </div>

      <div v-if="isLoading && !location" class="flex justify-center items-center h-96">
        <div class="loader">
          <div class="cloud"></div>
          <div class="sun"></div>
        </div>
      </div>

      <div v-else class="space-y-8">
        <!-- Header with location and date -->
        <div 
          class="flex flex-col md:flex-row justify-between items-start md:items-center mb-6 space-y-4 md:space-y-0"
          :class="{ 'animate-fade-in-up': contentLoaded }"
        >
          <div class="flex items-center gap-2">
            <span class="material-icons animate-pulse-slow text-blue-400">location_on</span>
            <span class="text-2xl font-light">{{ location }}</span>
          </div>
          <span class="text-gray-300">{{ date }}</span>
        </div>

        <!-- Main Weather Info -->
        <div 
          class="grid grid-cols-1 md:grid-cols-2 gap-8 mb-8 bg-[#ffffff08] backdrop-blur-sm p-6 rounded-2xl shadow-inner border border-[#ffffff10]"
          :class="{ 'animate-fade-in-up': contentLoaded }"
          style="animation-delay: 0.1s;"
        >
          <div>
            <div class="text-8xl font-extralight mb-4 bg-gradient-to-r from-blue-300 to-purple-400 text-transparent bg-clip-text">
              {{ currentTemp }}°<span class="text-2xl">C</span>
            </div>
            <div class="text-4xl text-gray-300 mb-8 font-light">{{ currentCondition }}</div>
            <div class="flex flex-col sm:flex-row gap-4 sm:gap-8">
              <div class="flex items-center gap-2 bg-[#ffffff08] p-3 rounded-xl">
                <span class="material-icons text-blue-400">air</span>
                <span>Wind</span>
                <span class="text-gray-300 ml-1">{{ wind }}</span>
              </div>
              <div class="flex items-center gap-2 bg-[#ffffff08] p-3 rounded-xl">
                <span class="material-icons text-blue-400">water_drop</span>
                <span>Humidity</span>
                <span class="text-gray-300 ml-1">{{ humidity }}</span>
              </div>
            </div>
          </div>
          <div class="flex justify-center items-center">
            <!-- Weather Icon -->
            <div class="relative" v-if="currentIcon">
              <img :src="currentIcon" alt="Weather icon" class="w-48 h-48 filter drop-shadow-glow animate-float" />
            </div>
            <div class="relative text-8xl animate-float" v-else>
              ⛈️
            </div>
          </div>
        </div>

        <!-- Forecast Tabs -->
        <div 
          class="bg-[#ffffff08] backdrop-blur-sm rounded-2xl overflow-hidden border border-[#ffffff10] shadow-lg"
          :class="{ 'animate-fade-in-up': contentLoaded }"
          style="animation-delay: 0.2s;"
        >
          <!-- Tab Navigation -->
          <div class="flex border-b border-[#ffffff15]">
            <button 
              @click="activeTab = 'weekly'" 
              class="flex-1 py-4 font-medium transition-all duration-300"
              :class="activeTab === 'weekly' ? 'bg-[#ffffff15] text-white' : 'text-gray-400 hover:bg-[#ffffff08]'"
            >
              Weekly Forecast
            </button>
            <button 
              @click="activeTab = 'hourly'" 
              class="flex-1 py-4 font-medium transition-all duration-300"
              :class="activeTab === 'hourly' ? 'bg-[#ffffff15] text-white' : 'text-gray-400 hover:bg-[#ffffff08]'"
            >
              Hourly Forecast
            </button>
          </div>
          
          <!-- Weekly Forecast Tab -->
          <div v-show="activeTab === 'weekly'" class="p-6">
            <div class="grid grid-cols-2 sm:grid-cols-4 md:grid-cols-7 gap-4">
              <div 
                v-for="(day, index) in forecast" 
                :key="day.day" 
                class="bg-[#ffffff08] rounded-xl p-4 text-center transform transition-all duration-300 hover:bg-[#ffffff15] hover:scale-105 hover:shadow-lg"
                :style="`animation-delay: ${0.1 + index * 0.05}s`"
                :class="{ 'animate-fade-in-up': contentLoaded }"
              >
                <div class="text-lg mb-2 font-medium">{{ day.day }}</div>
                <div class="text-3xl mb-2 bg-gradient-to-r from-blue-300 to-purple-400 text-transparent bg-clip-text font-light">{{ day.temp }}</div>
                <div class="h-16 flex justify-center items-center">
                  <img :src="day.icon" alt="Weather icon" class="w-16 h-16" />
                </div>
                <div class="text-gray-400 text-sm">{{ day.condition }}</div>
              </div>
            </div>
          </div>
          
          <!-- Hourly Forecast Tab -->
          <div v-show="activeTab === 'hourly'" class="p-6">
            <div class="grid grid-cols-2 sm:grid-cols-4 md:grid-cols-8 gap-4">
              <div 
                v-for="(hour, index) in hourlyForecast" 
                :key="hour.time" 
                class="bg-[#ffffff08] rounded-xl p-4 text-center transform transition-all duration-300 hover:bg-[#ffffff15] hover:scale-105"
                :style="`animation-delay: ${0.1 + index * 0.05}s`"
                :class="{ 'animate-fade-in-up': contentLoaded }"
              >
                <div class="text-xl mb-2 bg-gradient-to-r from-blue-300 to-purple-400 text-transparent bg-clip-text font-light">{{ hour.temp }}</div>
                <div class="text-gray-400">{{ hour.time }}</div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.weather-app {
  min-height: 100vh;
  background-size: 400% 400%;
  animation: gradient 15s ease infinite;
}

@keyframes gradient {
  0% {
    background-position: 0% 50%;
  }
  50% {
    background-position: 100% 50%;
  }
  100% {
    background-position: 0% 50%;
  }
}

.weather-card {
  min-height: 800px;
}

.loader {
  position: relative;
  width: 150px;
  height: 150px;
}

.cloud {
  position: absolute;
  top: 50%;
  left: 50%;
  width: 60px;
  height: 20px;
  background: #fff;
  border-radius: 10px;
  transform: translate(-50%, -50%);
  box-shadow: 0 0 40px rgba(255, 255, 255, 0.5);
  animation: cloud 3s ease-in-out infinite;
}

.cloud:before, .cloud:after {
  content: '';
  position: absolute;
  background: #fff;
  border-radius: 50%;
}

.cloud:before {
  width: 30px;
  height: 30px;
  top: -20px;
  left: 10px;
}

.cloud:after {
  width: 20px;
  height: 20px;
  top: -10px;
  left: 40px;
}

.sun {
  position: absolute;
  top: 50%;
  left: 50%;
  width: 40px;
  height: 40px;
  background: #ffeb3b;
  border-radius: 50%;
  transform: translate(-50%, -50%);
  box-shadow: 0 0 40px #ffeb3b;
  animation: sun 6s ease-in-out infinite;
}

@keyframes cloud {
  0%, 100% {
    transform: translate(-50%, -50%);
  }
  50% {
    transform: translate(-50%, -70%);
  }
}

@keyframes sun {
  0%, 100% {
    transform: translate(-80%, -50%);
    opacity: 0;
  }
  30%, 70% {
    transform: translate(-50%, -50%);
    opacity: 1;
  }
}

@keyframes spin {
  from {
    transform: translateY(-50%) rotate(0deg);
  }
  to {
    transform: translateY(-50%) rotate(360deg);
  }
}

.animate-spin {
  animation: spin 1s linear infinite;
}

@keyframes float {
  0%, 100% {
    transform: translateY(0);
  }
  50% {
    transform: translateY(-10px);
  }
}

.animate-float {
  animation: float 4s ease-in-out infinite;
}

.animate-pulse-slow {
  animation: pulse 3s cubic-bezier(0.4, 0, 0.6, 1) infinite;
}

@keyframes pulse {
  0%, 100% {
    opacity: 1;
  }
  50% {
    opacity: 0.5;
  }
}

@keyframes fade-in {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

.animate-fade-in {
  animation: fade-in 0.3s ease-in-out forwards;
}

@keyframes fade-in-up {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.animate-fade-in-up {
  opacity: 0;
  animation: fade-in-up 0.5s ease-out forwards;
}

.filter.drop-shadow-glow {
  filter: drop-shadow(0 0 10px rgba(255, 255, 255, 0.5));
}
</style>