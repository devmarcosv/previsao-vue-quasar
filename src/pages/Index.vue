<template>
  <q-page class="text-white">
    <q-header>
    </q-header>
    <div class="container form-city" v-if="editing">
    </div>
    <div
      class="main-temperature row column justify-between items-center circle"
    >
      <div class="temperatures-today row items-center">
        <span class="temp">{{ Math.round(this.weather.main.temp) }}°</span>

        <div class="others row column">
          <p>{{ Math.round(this.forecasters.list[0].temp.max) }}°</p>
          <p>{{ Math.round(this.forecasters.list[0].temp.min) }}°</p>
        </div>

      </div>
      <img
        v-if="this.weather.weather[0].icon"
        :src="getIconUrl(this.weather.weather[0].icon)"
        alt="Weather"
      />
      <p class="weather-description">
        {{ this.weather.weather[0].description }}
      </p>
      <p class="feels-like">
        Feels Like {{ Math.round(this.weather.main.feels_like) }}°
      </p>
    </div>

    <div class="prevision container row justify-between">


      <div
        v-for="(prevision, index) in this.forecasters.list.slice(1, 5)"
        :key="index"
        class="prevision-day"
      >
      <span> --------------------- </span>
        <img
          v-if="prevision.weather[0].icon"
          :src="getIconUrl(prevision.weather[0].icon)"
          alt="Weather"
        />
        <p class="max">{{ Math.round(prevision.temp.max) }}°</p>
        <hr />
        <p class="min">{{ Math.round(prevision.temp.min) }}°</p>
      </div>
    </div>

    <div class="info row justify-between container">
      <div class="xesque">
        <p class="value">{{ this.weather.wind.speed }} m/s</p>
        <p>Vento</p>
      </div>
      <div>
        <p class="value">{{ Math.round(this.weather.main.humidity) }}%</p>
        <p>Humidade</p>
      </div>
      <div>
        <p class="value">{{ this.weather.main.pressure }} hpa</p>
        <p>pressão</p>
      </div>
    </div>

    <div class="weather-forecast container">
      <p class="date">Próximos dias</p>

      <q-markup-table class="table transparent no-shadow text-white">
        <tbody>
          <tr
            v-for="(range, index) in this.forecast.list.slice(0, 9)"
            :key="index"
            class="no-border"
          >
            <td class="text-left">{{ getHours(range.dt_txt) }}</td>
            <td class="text-right">
              <img
                v-if="range.weather[0].icon"
                :src="getIconUrl(range.weather[0].icon)"
                alt="Weather"
              />
            </td>
            <td class="text-right">{{ Math.floor(range.main.temp) }}°</td>
            <td class="text-right">{{ range.main.humidity }}%</td>
            <td class="text-right">{{ range.wind.speed }} m/s</td>
          </tr>
        </tbody>
      </q-markup-table>
    </div>
  </q-page>
</template>

<script>
/* eslint-disable no-console */

import Chart from 'chart.js';
import { date, LocalStorage, Notify } from 'quasar';

import cityList from '../assets/city.list.min.json';
import globals from '../globals';

Chart.defaults.global.defaultFontColor = '#fff';
Chart.defaults.global.defaultFontSize = 14;

const London = {
  id: 2643743,
  name: 'Londres',
  country: 'GB',
  coord: {
    lon: -0.1258,
    lat: 51.5085,
  },
};

export default {
  name: 'PageIndex',
  data() {
    return {
      city: London,
      editing: false,
      submitting: false,
      model: London,
      cityOptions: [],
      weather: {
        main: {},
        wind: {},
        weather: [{}],
      },
      forecast: {
        list: [
          {
            rain: {},
            weather: [{}],
            main: {},
            wind: {},
          },
        ],
      },
      forecasters: {
        city: {},
        list: [
          {
            temp: {},
          },
        ],
      },
    };
  },
  methods: {
    filterFn(val, update, abort) {
      if (val.length < 2) {
        return abort();
      }

      update(() => {
        const needle = val.toLowerCase();

        this.cityOptions = cityList.filter(
          v => v.name.toLowerCase().indexOf(needle) !== -1,
        );
      });
    },
    submitCity() {
      this.submitting = true;

      LocalStorage.set('city', this.model);

      this.loadData();

      Notify.create({
        message: `ok '${this.model.name}, ${this.model.country}' city`,
      });

      this.editing = false;
      this.submitting = false;
    },
    getDay(index) {
      const newDate = date.addToDate(new Date(), { days: index });
      return date.formatDate(newDate, 'ddd');
    },
    getIconUrl(icon) {
      return `${globals.apiBaseUrl}/themes/openweathermap/assets/vendor/owm/img/widgets/${icon}.png`;
    },
    getHours(datetime) {
      if (datetime) {
        return date.formatDate(
          new Date(datetime).setUTCSeconds(this.weather.timezone),
          'HH:mm',
        );
      }
      return datetime;
    },
    loadWheater() {
      this.$axios
        .get(
          `${globals.apiBaseUrl}/data/2.5/weather/?id=${this.city.id}&${globals.apiParameters}`,
        )
        .then(response => {
          this.weather = response.data;
        })
        .catch(err => {
          console.log(err);
        });
    },
    loadForecast() {
      this.$axios
        .get(
          `${globals.apiBaseUrl}/data/2.5/forecast/?id=${this.city.id}&${globals.apiParameters}`,
        )
        .then(response => {
          this.forecast = response.data;
          this.drawChart(response.data.list);
        })
        .catch(err => {
          console.log(err);
        });
    },
    loadForecasters() {
      this.$axios
        .get(
          `${globals.apiBaseUrl}/data/2.5/forecast/daily/?id=${this.city.id}&${globals.apiParameters}`,
        )
        .then(response => {
          this.forecasters = response.data;
        })
        .catch(err => {
          console.log(err);
        });
    },
    drawChart(list) {
      const hours = [];
      const data = [];

      for (let i = 0; i < 6; i++) {
        hours.push(this.getHours(list[i].dt_txt));
        data.push(Math.round(list[i].main.temp));
      }

      const ctx = document
        .getElementById('weather-forecast-chart')
        .getContext('2d');
      new Chart(ctx, {
        type: 'line',
        data: {
          labels: hours,
          datasets: [
            {
              label: 'Temperature',
              borderColor: '#fff',
              fill: false,
              data,
            },
          ],
        },
        options: {
          scales: {
            yAxes: [
              {
                ticks: {
                  callback(value) {
                    return `${value}°`;
                  },
                },
              },
            ],
          },
        },
      });
    },
    loadData() {
      const city = LocalStorage.getItem('city');

      if (city) {
        this.city = city;
        this.model = city;
      }

      this.loadWheater();
      this.loadForecast();
      this.loadForecasters();
    },
  },
  mounted() {
    this.loadData();
  },
};
</script>

<style >
.q-page {
  padding-top: 80px;
  background: linear-gradient(-200deg, #595D5D, #9DA1A1);
  background-position: top left;
  background-size: cover;
}


.container {
  width: 90%;
  max-width: 960px;
  margin: 0 auto;
}

.padding-top-30 {
  padding-top: 30px;
}


.form-city {
  margin-bottom: 50px;
}

.main-temperature {
  width: 210px;
  height: 210px;
  margin: 0 auto;
  padding: 30px;
}

.temperatures-today .temp {
  font-size: 2.5em;
}

.temperatures-today .separator {
  margin-left: 5px;
  margin-right: 10px;
  font-size: 2.5em;
}

.main-temperature p {
  margin-bottom: 0;
}

.main-temperature img {
  width: 50px;
}

.main-temperature .weather-description {
  font-size: 1.1em;
  text-transform: capitalize;
}

.main-temperature .real-feel {
  font-size: 1.2em;
}

.prevision {
  margin: 50px auto;
}

.prevision p {
  margin-bottom: 0;
  font-size: 1.2em;
  text-align: center;
}

.prevision .day {
  font-weight: bold;
}

.prevision .min {
  color: #d1d1d1;
}

.prevision img {
  width: 50px;
  margin: 5px 0;
}

.info-info {
  margin-bottom: 50px;
}

.info .circle {
  width: 90px;
  height: 90px;
}

.info .circle p {
  margin-bottom: 0;
}

.info .circle .value {
  font-size: 1.1em;
}

.weather-forecast {
  margin-bottom: 30px;
}

.weather-forecast .date {
  font-size: 1.2em;
  font-weight: bold;
  letter-spacing: 1px;
}

.weather-forecast .table td {
  padding: 5px 10px;
  font-size: 1.1em;
  border: none;
}

.weather-forecast .table img {
  width: 30px;
}


</style>
