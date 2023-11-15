<template>
  <div class="position-relative" v-if="mapOptions.show">
    <div
      v-if="!hideGeoSearch"
      class="geosearch-container"
      @mouseover="disableMap"
      @mouseleave="enableMap"
    >
      <c-input-search
        v-model="geoSearch.query"
        :placeholder="labels.geosearchInputPlaceholder"
        :autocomplete="'off'"
        :debounce="300"
        @input="onGeoSearch"
      />

      <div class="geosearch-results">
        <div
          v-for="(result, idx) in geoSearch.results"
          :key="idx"
          class="geosearch-result"
          @click="placeGeoSearchMarker(result)"
        >
          {{ result.label }}
        </div>
      </div>
    </div>

    <l-map
      ref="map"
      :zoom="mapOptions.zoom"
      :center="mapOptions.center"
      :min-zoom="mapOptions.minZoom"
      :max-zoom="mapOptions.maxZoom"
      :bounds="mapOptions.bounds"
      :max-bounds="mapOptions.maxBounds"
      style="cursor: pointer;"
      :style="mapStyle"
      @click="onMapClick"
      @locationfound="onLocationFound"
      @update:zoom="onZoom"
      @update:center="onCenter"
      @update:bounds="onBoundsUpdate"
    >
      <l-tile-layer
        url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"
        :attribution="mapOptions.attribution"
      />
    
      <l-polygon
        v-for="(geometry, i) in polygons"
        :key="i"
        :lat-lngs="geometry.map(value => value.geometry)"
        :color="geometry[0].color"
      />
    
      <l-marker
        v-for="(marker, i) in markers"
        :key="`marker-${i}`"
        :lat-lng="marker"
        :icon="getIcon(marker.color)"
        :opacity="activeMarkerIndex === undefined || i == activeMarkerIndex ? 1.0 : 0.6"
        @click="onMarkerClick(i, marker)"
      />
    
      <l-control class="leaflet-bar">
        <a
          v-if="!hideCurrentLocationButton"
          :title="labels.tooltip && labels.tooltip.goToCurrentLocation"
          role="button"
          class="d-flex justify-content-center align-items-center"
          @click="goToCurrentLocation"
        >
          <font-awesome-icon
            :icon="['fas', 'location-arrow']"
            class="text-primary"
          />
        </a>
      </l-control>
    </l-map>
  </div>
</template>

<script>
import { OpenStreetMapProvider } from 'leaflet-geosearch'
import { divIcon } from 'leaflet'
import { LControl, LPolygon } from 'vue2-leaflet'
import CInputSearch from '../input/CInputSearch.vue'

export default {
  props: {
    hideCurrentLocationButton: {
      type: Boolean,
      default: false
    },
    
    labels: {
      type: Object,
      required: true
    },
    
    map: {
      type: Object
    },
    
    markers: {
      type: Array,
      default: () => ([])
    },

    activeMarkerIndex:{
      type: Number,
      default: undefined
    },

    mapStyle: {
      type: String,
      default: "height: 75vh; width: 100%;"
    },

    polygons: {
      type: Array,
      default: () => ([])
    },

    hideGeoSearch: {
      type: Boolean,
      default: false
    },

    disable: {
      type: Boolean,
      default: false
    }
  },

  components: {
    LControl,
    CInputSearch,
    LPolygon
  },
  
  data () {
    return {
      geoSearch: {
        query: '',
        provider: new OpenStreetMapProvider(),
        results: [],
      },
    }
  },

  computed: {
    mapOptions () {
      const defaultOptions = {
        show: true,
        zoom: 3,
        center: [30, 30],
        rotation: 0,
        attribution: '&copy; <a target="_blank" href="http://osm.org/copyright">OpenStreetMap</a>',
      }

      return {
        ...defaultOptions,
        ...this.map
      }
    }
  },

  watch: {
    mapOptions: {
      handler (map) {
        if (map.show && this.$refs.map) {
          this.$refs.map.mapObject.invalidateSize()
        }
      }
    }
  },

  mounted () {
    this.onBoundsUpdate(this.$refs.map.mapObject.getBounds())
  },

  methods: {
    onGeoSearch (query) {
      if (!query) {
        this.geoSearch.results = []
        return
      }

      this.geoSearch.provider.search({ query }).then(results => {
        this.geoSearch.results = results.map(result => ({
          ...result,
          latlng: {
            lat: result.raw.lat,
            lng: result.raw.lon,
          },
        }))
      }).catch(() => {
        this.$emit('trigger-geosearch-error')
      })
    },

    placeGeoSearchMarker (result) {
      const zoom = this.$refs.map.mapObject._zoom >= 15 ? this.$refs.map.mapObject._zoom : 15
      this.$refs.map.mapObject.flyTo([result.latlng.lat, result.latlng.lng], zoom, { animate: false })
      this.geoSearch.results = []
      this.onMapClick(result)
    },

    onLocationFound ({ latitude, longitude }) {
      const zoom = this.mapRef.mapObject._zoom >= 13 ? this.mapRef.mapObject._zoom : 13
      this.$refs.map.mapObject.flyTo([latitude, longitude], zoom)
    },

    disableMap () {
      if (this.disable) this.$refs.map.mapObject._handlers.forEach(handler => handler.disable())
    },

    enableMap () {
      if (this.disable) this.$refs.map.mapObject._handlers.forEach(handler => handler.enable())
    },

    onMarkerClick(index, marker) {
      this.$emit('on-marker-marker', { index, marker })
    },

    goToCurrentLocation () {
      this.$refs.map.mapObject.locate()
    },

    onMapClick(e) {
      this.$emit('on-map-click', e)
    },


    getIcon (markerColor) {
      let html = "<img src='data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABkAAAApCAYAAADAk4LOAAAFgUlEQVR4Aa1XA5BjWRTN2oW17d3YaZtr2962HUzbDNpjszW24mRt28p47v7zq/bXZtrp/lWnXr337j3nPCe85NcypgSFdugCpW5YoDAMRaIMqRi6aKq5E3YqDQO3qAwjVWrD8Ncq/RBpykd8oZUb/kaJutow8r1aP9II0WmLKLIsJyv1w/kqw9Ch2MYdB++12Onxee/QMwvf4/Dk/Lfp/i4nxTXtOoQ4pW5Aj7wpici1A9erdAN2OH64x8OSP9j3Ft3b7aWkTg/Fm91siTra0f9on5sQr9INejH6CUUUpavjFNq1B+Oadhxmnfa8RfEmN8VNAsQhPqF55xHkMzz3jSmChWU6f7/XZKNH+9+hBLOHYozuKQPxyMPUKkrX/K0uWnfFaJGS1QPRtZsOPtr3NsW0uyh6NNCOkU3Yz+bXbT3I8G3xE5EXLXtCXbbqwCO9zPQYPRTZ5vIDXD7U+w7rFDEoUUf7ibHIR4y6bLVPXrz8JVZEql13trxwue/uDivd3fkWRbS6/IA2bID4uk0UpF1N8qLlbBlXs4Ee7HLTfV1j54APvODnSfOWBqtKVvjgLKzF5YdEk5ewRkGlK0i33Eofffc7HT56jD7/6U+qH3Cx7SBLNntH5YIPvODnyfIXZYRVDPqgHtLs5ABHD3YzLuespb7t79FY34DjMwrVrcTuwlT55YMPvOBnRrJ4VXTdNnYug5ucHLBjEpt30701A3Ts+HEa73u6dT3FNWwflY86eMHPk+Yu+i6pzUpRrW7SNDg5JHR4KapmM5Wv2E8Tfcb1HoqqHMHU+uWDD7zg54mz5/2BSnizi9T1Dg4QQXLToGNCkb6tb1NU+QAlGr1++eADrzhn/u8Q2YZhQVlZ5+CAOtqfbhmaUCS1ezNFVm2imDbPmPng5wmz+gwh+oHDce0eUtQ6OGDIyR0uUhUsoO3vfDmmgOezH0mZN59x7MBi++WDL1g/eEiU3avlidO671bkLfwbw5XV2P8Pzo0ydy4t2/0eu33xYSOMOD8hTf4CrBtGMSoXfPLchX+J0ruSePw3LZeK0juPJbYzrhkH0io7B3k164hiGvawhOKMLkrQLyVpZg8rHFW7E2uHOL888IBPlNZ1FPzstSJM694fWr6RwpvcJK60+0HCILTBzZLFNdtAzJaohze60T8qBzyh5ZuOg5e7uwQppofEmf2++DYvmySqGBuKaicF1blQjhuHdvCIMvp8whTTfZzI7RldpwtSzL+F1+wkdZ2TBOW2gIF88PBTzD/gpeREAMEbxnJcaJHNHrpzji0gQCS6hdkEeYt9DF/2qPcEC8RM28Hwmr3sdNyht00byAut2k3gufWNtgtOEOFGUwcXWNDbdNbpgBGxEvKkOQsxivJx33iow0Vw5S6SVTrpVq11ysA2Rp7gTfPfktc6zhtXBBC+adRLshf6sG2RfHPZ5EAc4sVZ83yCN00Fk/4kggu40ZTvIEm5g24qtU4KjBrx/BTTH8ifVASAG7gKrnWxJDcU7x8X6Ecczhm3o6YicvsLXWfh3Ch1W0k8x0nXF+0fFxgt4phz8QvypiwCCFKMqXCnqXExjq10beH+UUA7+nG6mdG/Pu0f3LgFcGrl2s0kNNjpmoJ9o4B29CMO8dMT4Q5ox8uitF6fqsrJOr8qnwNbRzv6hSnG5wP+64C7h9lp30hKNtKdWjtdkbuPA19nJ7Tz3zR/ibgARbhb4AlhavcBebmTHcFl2fvYEnW0ox9xMxKBS8btJ+KiEbq9zA4RthQXDhPa0T9TEe69gWupwc6uBUphquXgf+/FrIjweHQS4/pduMe5ERUMHUd9xv8ZR98CxkS4F2n3EUrUZ10EYNw7BWm9x1GiPssi3GgiGRDKWRYZfXlON+dfNbM+GgIwYdwAAAAASUVORK5CYII=' alt='marker' />"

      if (markerColor) {
        html = `<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 34.892337" height="60" width="40" style="margin-top: -40px;margin-left: -15px;height: 35px;">
          <g transform="translate(-814.59595,-274.38623)">
            <g transform="matrix(1.1855854,0,0,1.1855854,-151.17715,-57.3976)">
              <path d="m 817.11249,282.97118 c -1.25816,1.34277 -2.04623,3.29881 -2.01563,5.13867 0.0639,3.84476 1.79693,5.3002 4.56836,10.59179 0.99832,2.32851 2.04027,4.79237 3.03125,8.87305 0.13772,0.60193 0.27203,1.16104 0.33416,1.20948 0.0621,0.0485 0.19644,-0.51262 0.33416,-1.11455 0.99098,-4.08068 2.03293,-6.54258 3.03125,-8.87109 2.77143,-5.29159 4.50444,-6.74704 4.56836,-10.5918 0.0306,-1.83986 -0.75942,-3.79785 -2.01758,-5.14062 -1.43724,-1.53389 -3.60504,-2.66908 -5.91619,-2.71655 -2.31115,-0.0475 -4.4809,1.08773 -5.91814,2.62162 z" style="fill:${markerColor};stroke:${markerColor};"/>
              <circle r="3.0355" cy="288.25278" cx="823.03064" id="path3049" style="display:inline;fill:#FFFFFF;"/>
            </g>
          </g>
        </svg>`
      }

      return divIcon({
        className: 'marker-pin',
        html
      })
    },

    onZoom (e) {
      this.$emit('on-zoom', e)
    },

    onCenter (e) {
      this.$emit('on-center', e)
    },

    onBoundsUpdate (value) {
      value = value || this.$refs.map.mapObject.getBounds()

      this.$emit('on-bounds-update', value)
    }
  },
}
</script>

<style scoped>
.geosearch-container {
  position: absolute;
  display: block;
  height: auto;
  width: 50%;
  max-width: 400px;
  cursor: auto;
  z-index: 10000;
  left: 25%;
  top: 10px;
}

.geosearch-results {
  margin: 1px;
  border-radius: 2px;
  background-color: var(--white);
  max-height: 50%;
  overflow: auto;
}

.geosearch-result {
  border-radius: 2px;
  line-height: 32px;
  padding: 0 8px;
  font-size: 12px;
  white-space: nowrap;
}

.geosearch-result:hover {
  background-color: var(--gray-200);
  cursor: pointer;
}
</style>
