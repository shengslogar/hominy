<template>
  <div class="weather-container">
    <transition name="slideYFade">
      <Spinner v-if="coords.updated === null || weather.updated === null" class="weather-spinner"/>
      <div v-else class="weather">
        <div class="weather-now"
             @click="$emit('temperatureClicked')">
          <span class="weather-now-temp">
            {{ Math.round(weather.temperature) }}
          </span>
          <span class="weather-now-units">
            &deg;{{ weather.units }}
          </span>
        </div>
        <div class="weather-location">
          <span class="weather-location-city" :title="coordsTooltipText">
            {{ weather.location }}
          </span>
        </div>
      </div>
    </transition>
  </div>
</template>

<script>
import Spinner from '@/components/Spinner';

export default {
  name: 'Weather',
  components: {
    Spinner,
  },
  props: {
    weatherUnits: {
      default: 'K',
      type: String,
    },
    cache: null,
  },
  data() {
    return {
      coords: {
        lat: null,
        long: null,
        updated: null,
        loading: true,
      },
      weather: {
        apiKey: 'd9b473a4fb6873a891ca291f10853854',
        temperature: null,
        units: null,
        description: '',
        location: '',
        updated: null,
        loading: true,
      },
    };
  },
  methods: {
    async getWeather() {
      this.weather.loading = true;

      const request = await fetch(`https://api.openweathermap.org/data/2.5/weather?lat=${this.coords.lat}&lon=${this.coords.long}&appid=${this.weather.apiKey}`)
        .then(response => response.json());

      this.weather.rawTemp = request.main.temp;
      this.weather.description = request.weather[0].description;
      this.weather.location = request.name;
      this.weather.loading = false;
      this.weather.updated = (+new Date());

      this.recomputeWeatherTemp();
    },
    recomputeWeatherTemp() {
      this.weather.units = this.weatherUnits;

      switch (this.weatherUnits) {
        case 'F':
          this.weather.temperature = (9 / 5) * (this.weather.rawTemp - 273) + 32
          break;
        case 'C':
          this.weather.temperature = this.weather.rawTemp - 273;
          break;

        case 'K':
        default:
          this.weather.temperature = this.weather.rawTemp;
          break;
      }
    },
    async getLocation() {
      this.coords.loading = true;

      await new Promise((resolve, reject) => {
        navigator.geolocation.getCurrentPosition(
          (response) => {
            this.coords.lat = response.coords.latitude;
            this.coords.long = response.coords.longitude;
            this.coords.updated = +(new Date());
            this.coords.loading = false;

            if (resolve)
              resolve.apply(this.coords);
          },
          () => {
            this.coords.loading = false;

            if (reject) {
              reject.apply();
            }
          });
      });
    },
    async refreshWeather() {
      await this.getLocation();
      await this.getWeather();

      this.$emit('updateCache', {
        coords: this.coords,
        weather: this.weather,
      });

      setTimeout(() => {
        this.refreshWeather();
      }, 60000);
    },
  },
  computed: {
    coordsTooltipText() {
      return `(${Math.round(this.coords.long)}, ${Math.round(this.coords.lat)})`;
    }
  },
  watch: {
    weatherUnits() {
      this.recomputeWeatherTemp();
    },
  },
  created() {
    // cached data?
    if (this.cache !== null) {
      Object.assign(this.weather, this.cache.weather);
      this.recomputeWeatherTemp(); // just in case, fringe case where setting changed but cache didn't update
      Object.assign(this.coords, this.cache.coords);

      // wait a second -> save some api limits!
      setTimeout(() => {
        this.refreshWeather();
      }, 1000);
    } else {
      this.refreshWeather();
    }
  },
}
</script>

<style scoped>
.slideYFade-enter-active,
.slideYFade-leave-active {
  transition-duration : .4s;
}

.slideYFade-enter,
.slideYFade-leave-to {
  opacity   : 0;
  transform : translateY(10px);
}

@keyframes weather-container-before--entrance {
  0% {
    opacity : 0;
    transform: scaleX(.1);
  }
}

.weather-container {
  text-align : center;
  padding    : 30px 0 0;
  margin-top : 15px;
  position   : relative;
}

.weather-container::before {
  content    : '';
  display    : block;
  position   : absolute;
  background : rgba(255, 255, 255, .5);
  height     : 1px;
  top        : -.5px;
  left       : 0;
  right      : 0;
  animation  : weather-container-before--entrance .3s ease;
}

.weather {
  display         : flex;
  align-items     : center;
  justify-content : center;
}

.weather > * {
  flex : 1 1 auto;
}

.weather-spinner {
  position        : absolute;
  top             : 0;
  bottom          : 0;
  display         : flex;
  align-items     : center;
  justify-content : center;
  left            : 0;
  right           : 0;
}

.weather-now,
.weather-location {
  padding: 0 1rem;
  flex-basis: 50%;
}

.weather-now {
  font-size: 1.75rem;
  font-weight: 300;
}

.weather-location {
  font-size: 1rem;
  border-left : 1px solid rgba(255, 255, 255, .5);
}
</style>
