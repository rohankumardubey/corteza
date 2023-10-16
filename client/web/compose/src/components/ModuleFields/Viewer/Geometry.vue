<template>
  <div>
    <span
      v-for="(c, index) of localValue"
      :key="index"
      :class="{ 'd-block': field.options.multiDelimiter === '\n' }"
    >
      <a
        class="text-primary pointer"
        @click.stop="openMap(index)"
      >
        {{ c.lat }}, {{ c.lng }}
        <font-awesome-icon
          :icon="['fas', 'map-marked-alt']"
        />
        {{ index !== localValue.length - 1 ? field.options.multiDelimiter : '' }}
      </a>
    </span>

    <b-modal
      v-model="map.show"
      :title="field.label || field.name"
      size="lg"
      body-class="p-0"
      hide-footer
    >
      <div
        v-if="!field.options.hideGeoSearch"
        class="geosearch-container"
      >
        <c-input-search
          v-model="geoSearch.query"
          :placeholder="$t('geosearchInputPlaceholder')"
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

      <c-map
        ref="c-map"
        :field="field"
        :map="map"
        :local-value-index="localValueIndex"
        :markers="localValue"
        :hide-current-location-button="!field.options.hideCurrentLocationButton"
        :label="{
          tooltip: { 'goToCurrentLocation': $t('tooltip.goToCurrentLocation') }
        }"
        @on-location-found="onLocationFound"
        @on-go-to-current-location="goToCurrentLocation"
        @on-map-ref="onMapRefEmit"
      />
    </b-modal>

    <errors :errors="errors" />
  </div>
</template>
<script>
import base from './base'
import { latLng } from 'leaflet'
import { OpenStreetMapProvider } from 'leaflet-geosearch'
import { components } from '@cortezaproject/corteza-vue'
import { isNumber } from 'lodash'
const { CInputSearch, CMap } = components

export default {
  i18nOptions: {
    namespaces: 'field',
    keyPrefix: 'kind.geometry',
  },

  components: {
    CInputSearch,
    CMap,
  },

  extends: base,

  data () {
    return {
      map: {
        show: false,
        zoom: 14,
        center: [30, 30],
        rotation: 0,
        attribution: '&copy; <a target="_blank" rel="noopener noreferrer" href="http://osm.org/copyright">OpenStreetMap</a>',
      },
      localValueIndex: undefined,

      geoSearch: {
        query: '',
        provider: new OpenStreetMapProvider(),
        results: [],
      },

      mapRef: undefined,
    }
  },

  computed: {
    localValue () {
      if (this.field.isMulti) {
        return this.value.map(v => {
          return this.getLatLng(JSON.parse(v || '{"coordinates":[]}').coordinates || [])
        }).filter(c => c)
      } else {
        return [this.getLatLng(JSON.parse(this.value || '{"coordinates":[]}').coordinates || [])].filter(c => c)
      }
    },
  },

  beforeDestroy () {
    this.setDefaultValues()
  },

  methods: {
    onMapRefEmit (map) {
      this.mapRef = map
    },

    openMap (index) {
      this.localValueIndex = index
      this.map.center = this.localValue[index] || this.field.options.center
      this.map.zoom = index >= 0 ? 13 : this.field.options.zoom
      this.map.show = true

      setTimeout(() => {
        this.mapRef.mapObject.invalidateSize()
      }, 100)
    },

    getLatLng (coordinates = [undefined, undefined]) {
      const [lat, lng] = coordinates

      if (isNumber(lat) && isNumber(lng)) {
        return latLng(lat, lng)
      }
    },

    goToCurrentLocation () {
      this.mapRef.mapObject.locate()
    },

    onLocationFound ({ latitude, longitude }) {
      const zoom = this.mapRef.mapObject._zoom >= 13 ? this.mapRef.mapObject._zoom : 13
      this.mapRef.mapObject.flyTo([latitude, longitude], zoom)
    },

    placeGeoSearchMarker (result) {
      const zoom = this.mapRef.mapObject._zoom >= 15 ? this.mapRef.mapObject._zoom : 15
      this.mapRef.mapObject.flyTo([result.latlng.lat, result.latlng.lng], zoom, { animate: false })
      this.geoSearch.results = []
    },

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
        this.toastErrorHandler(this.$t('notification:field-geometry.geolocationErrors.locationSearchFailed'))()
      })
    },

    setDefaultValues () {
      this.map = {}
      this.localValueIndex = undefined
      this.geoSearch = {}
    },
  },
}
</script>
