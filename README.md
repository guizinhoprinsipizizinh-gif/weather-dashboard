const WEATHER_ICONS = {
  0: '☀️',
  1: '🌤️',
  2: '⛅',
  3: '☁️',
  45: '🌫️',
  48: '🌫️',
  51: '🌦️',
  53: '🌦️',
  55: '🌧️',
  56: '🌧️',
  57: '🌧️',
  61: '🌦️',
  63: '🌧️',
  65: '🌧️',
  66: '🌧️',
  67: '🌧️',
  71: '🌨️',
  73: '🌨️',
  75: '🌨️',
  77: '🌨️',
  80: '🌦️',
  81: '🌧️',
  82: '🌧️',
  85: '🌨️',
  86: '🌨️',
  95: '⛈️',
  96: '⛈️',
  99: '⛈️'
};

const STORAGE_KEY = 'weather-dashboard-favorites';
const DEFAULT_CITY = 'Lisbon';

const state = {
  unit: 'c',
  favorites: getFavorites(),
  currentLocation: null
};

const cityNameEl = document.querySelector('#city-name');
const tempValueEl = document.querySelector('#current-temp');
const feelsLikeEl = document.querySelector('#feels-like');
const humidityEl = document.querySelector('#humidity');
const windSpeedEl = document.querySelector('#wind-speed');
const sunriseEl = document.querySelector('#sunrise');
const weatherSummaryEl = document.querySelector('#weather-summary');
const weatherIconEl = document.querySelector('#weather-icon');
const forecastListEl = document.querySelector('#forecast-list');
const favoriteListEl = document.querySelector('#favorite-list');
const cityInputEl = document.querySelector('#city-input');
const searchFormEl = document.querySelector('#search-form');
const locationBtnEl = document.querySelector('#location-btn');
const unitButtons = document.querySelectorAll('.unit');

function getFavorites() {
  const saved = localStorage.getItem(STORAGE_KEY);
  if (!saved) return ['Lisbon'];

  try {
    const parsed = JSON.parse(saved);
    return Array.isArray(parsed) && parsed.length ? parsed : ['Lisbon'];
  } catch (error) {
    return ['Lisbon'];
  }
}

function saveFavorites() {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(state.favorites));
}

function toLocalTime(dateString) {
  const date = new Date(dateString);
  return date.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
}

function toDayName(dateString) {
  const date = new Date(dateString);
  return date.toLocaleDateString([], { weekday: 'short' });
}

function formatTemperature(value) {
  if (state.unit === 'f') {
    return `${Math.round((value * 9) / 5 + 32)}°F`;
  }
  return `${Math.round(value)}°C`;
}

function formatWind(value) {
  return `${Math.round(value)} km/h`;
}

function getWeatherLabel(code) {
  const map = {
    0: 'Clear sky',
    1: 'Mostly clear',
    2: 'Partly cloudy',
    3: 'Cloudy',
    45: 'Foggy',
    48: 'Foggy',
    51: 'Light drizzle',
    53: 'Drizzle',
    55: 'Heavy drizzle',
    56: 'Freezing drizzle',
    57: 'Freezing drizzle',
    61: 'Light rain',
    63: 'Rain',
    65: 'Heavy rain',
    66: 'Freezing rain',
    67: 'Freezing rain',
    71: 'Light snow',
    73: 'Snow',
    75: 'Heavy snow',
    77: 'Snow grains',
    80: 'Rain showers',
    81: 'Heavy showers',
    82: 'Violent showers',
    85: 'Snow showers',
    86: 'Heavy snow showers',
    95: 'Thunderstorm',
    96: 'Thunderstorm with hail',
    99: 'Thunderstorm with hail'
  };

  return map[code] || 'Weather';
}

function renderFavorites() {
  favoriteListEl.innerHTML = '';

  state.favorites.forEach((city) => {
    const btn = document.createElement('button');
    btn.type = 'button';
    btn.className = 'favorite-btn';
    btn.textContent = city;
    btn.addEventListener('click', () => fetchWeatherByCity(city));
    favoriteListEl.appendChild(btn);
  });
}

function renderForecast(weatherData) {
  const daily = weatherData.daily;
  forecastListEl.innerHTML = '';

  daily.time.slice(0, 5).forEach((day, index) => {
    const item = document.createElement('div');
    item.className = 'forecast-item';

    const code = daily.weather_code[index];
    item.innerHTML = `
      <div class="forecast-day">${toDayName(day)}</div>
      <div class="forecast-icon">${WEATHER_ICONS[code] || '🌤️'}</div>
      <div class="forecast-temp">
        <span class="max">${state.unit === 'f' ? Math.round((daily.temperature_2m_max[index] * 9) / 5 + 32) : Math.round(daily.temperature_2m_max[index])}°</span>
        <span class="min">${state.unit === 'f' ? Math.round((daily.temperature_2m_min[index] * 9) / 5 + 32) : Math.round(daily.temperature_2m_min[index])}°</span>
      </div>
    `;

    forecastListEl.appendChild(item);
  });
}

function setActiveUnit(unit) {
  state.unit = unit;
  unitButtons.forEach((button) => {
    const active = button.dataset.unit === unit;
    button.classList.toggle('active', active);
  });

  const currentTemp = document.querySelector('#current-temp').dataset.temp;
  if (currentTemp) {
    const currentValue = Number(currentTemp);
    tempValueEl.textContent = state.unit === 'f'
      ? `${Math.round((currentValue * 9) / 5 + 32)}`
      : `${Math.round(currentValue)}`;
    document.querySelector('#current-unit').textContent = state.unit === 'f' ? '°F' : '°C';

    const feelsLike = document.querySelector('#feels-like').dataset.value;
    if (feelsLike) {
      const feels = Number(feelsLike);
      feelsLikeEl.textContent = state.unit === 'f'
        ? `${Math.round((feels * 9) / 5 + 32)}°F`
        : `${Math.round(feels)}°C`;
    }
  }
}

function updateWeatherUI(data) {
  const { current, daily, timezone } = data;
  const tempValue = current.temperature_2m;
  const feels = current.apparent_temperature;
  const weatherCode = current.weather_code;

  cityNameEl.textContent = `${data.location.name}, ${data.location.country || data.location.admin1 || ''}`.trim();
  tempValueEl.dataset.temp = tempValue;
  tempValueEl.textContent = state.unit === 'f'
    ? Math.round((tempValue * 9) / 5 + 32)
    : Math.round(tempValue);
  document.querySelector('#current-unit').textContent = state.unit === 'f' ? '°F' : '°C';

  weatherIconEl.textContent = WEATHER_ICONS[weatherCode] || '☀️';
  weatherSummaryEl.textContent = getWeatherLabel(weatherCode);

  feelsLikeEl.dataset.value = feels;
  feelsLikeEl.textContent = state.unit === 'f'
    ? `${Math.round((feels * 9) / 5 + 32)}°F`
    : `${Math.round(feels)}°C`;

  humidityEl.textContent = `${Math.round(current.relative_humidity_2m)}%`;
  windSpeedEl.textContent = formatWind(current.wind_speed_10m);
  sunriseEl.textContent = toLocalTime(daily.sunrise[0]);

  renderForecast(data);
}

async function fetchWeatherByCity(city) {
  try {
    const geoRes = await fetch(`https://geocoding-api.open-meteo.com/v1/search?name=${encodeURIComponent(city)}&count=1&language=en&format=json`);
    const geoData = await geoRes.json();

    if (!geoData.results || !geoData.results.length) {
      throw new Error('City not found');
    }

    const result = geoData.results[0];
    const weatherRes = await fetch(
      `https://api.open-meteo.com/v1/forecast?latitude=${result.latitude}&longitude=${result.longitude}&current=temperature_2m,relative_humidity_2m,apparent_temperature,weather_code,wind_speed_10m&daily=weather_code,temperature_2m_max,temperature_2m_min,sunrise,sunset&timezone=auto&forecast_days=5`
    );

    const weatherData = await weatherRes.json();
    weatherData.location = {
      name: result.name,
      country: result.country,
      admin1: result.admin1 || ''
    };

    updateWeatherUI(weatherData);
    cityInputEl.value = city;

    if (!state.favorites.includes(city)) {
      state.favorites = [city, ...state.favorites].slice(0, 5);
      saveFavorites();
      renderFavorites();
    }
  } catch (error) {
    alert('Unable to load weather for that city. Please try another one.');
    console.error(error);
  }
}

async function fetchWeatherByCoords(lat, lon) {
  try {
    const res = await fetch(
      `https://api.open-meteo.com/v1/forecast?latitude=${lat}&longitude=${lon}&current=temperature_2m,relative_humidity_2m,apparent_temperature,weather_code,wind_speed_10m&daily=weather_code,temperature_2m_max,temperature_2m_min,sunrise,sunset&timezone=auto&forecast_days=5`
    );

    const data = await res.json();
    const cityName = `Your location`;
    data.location = {
      name: cityName,
      country: '',
      admin1: ''
    };

    updateWeatherUI(data);
  } catch (error) {
    alert('Unable to get weather for your current location.');
    console.error(error);
  }
}

searchFormEl.addEventListener('submit', async (event) => {
  event.preventDefault();
  const city = cityInputEl.value.trim();

  if (!city) return;
  await fetchWeatherByCity(city);
});

locationBtnEl.addEventListener('click', () => {
  if (!navigator.geolocation) {
    alert('Geolocation is not supported in your browser.');
    return;
  }

  navigator.geolocation.getCurrentPosition(
    async (position) => {
      const { latitude, longitude } = position.coords;
      await fetchWeatherByCoords(latitude, longitude);
    },
    () => {
      alert('Location access was denied. Please enter a city manually.');
    }
  );
});

unitButtons.forEach((button) => {
  button.addEventListener('click', () => {
    setActiveUnit(button.dataset.unit);
    const currentValue = Number(tempValueEl.dataset.temp || 0);
    tempValueEl.textContent = state.unit === 'f'
      ? Math.round((currentValue * 9) / 5 + 32)
      : Math.round(currentValue);
    document.querySelector('#current-unit').textContent = state.unit === 'f' ? '°F' : '°C';

    const feels = Number(feelsLikeEl.dataset.value || 0);
    feelsLikeEl.textContent = state.unit === 'f'
      ? `${Math.round((feels * 9) / 5 + 32)}°F`
      : `${Math.round(feels)}°C`;

    renderForecast({
      daily: {
        time: Array.from({ length: 5 }, (_, index) => new Date(Date.now() + index * 86400000).toISOString()),
        weather_code: [0, 1, 2, 3, 61],
        temperature_2m_max: [20, 21, 19, 22, 23],
        temperature_2m_min: [12, 13, 11, 14, 15]
      }
    });
  });
});

function initialize() {
  renderFavorites();
  setActiveUnit('c');
  fetchWeatherByCity(DEFAULT_CITY);
}

initialize();
