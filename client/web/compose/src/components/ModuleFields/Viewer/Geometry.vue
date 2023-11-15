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
      <c-map
        ref="c-map"
        :field="field"
        :map="map"
        :active-marker-index="localValueIndex"
        :markers="localValue"
        :hide-current-location-button="field.options.hideCurrentLocationButton"
        :hide-geo-search="!field.options.hideGeoSearch"
        :labels="{
          tooltip: { 'goToCurrentLocation': $t('tooltip.goToCurrentLocation') }
        }"
        @trigger-geosearch-error="triggerGeoSearchError"
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
const { CMap } = components

export default {
  i18nOptions: {
    namespaces: 'field',
    keyPrefix: 'kind.geometry',
  },

  components: {
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
    openMap (index) {
      this.localValueIndex = index
      this.map.center = this.localValue[index] || this.field.options.center
      this.map.zoom = index >= 0 ? 13 : this.field.options.zoom
      this.map.show = true
    },

    getLatLng (coordinates = [undefined, undefined]) {
      const [lat, lng] = coordinates

      if (isNumber(lat) && isNumber(lng)) {
        return latLng(lat, lng)
      }
    },

    triggerGeoSearchError () {
      this.toastErrorHandler(this.$t('notification:field-geometry.geolocationErrors.locationSearchFailed'))()
    },

    setDefaultValues () {
      this.map = {}
      this.localValueIndex = undefined
      this.geoSearch = {}
    },
  },
}
</script>
