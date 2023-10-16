<template>
  <div>
    <div class="my-2">
      <c-map
        ref="c-map"
        :map="{
          zoom: options.zoomStarting,
          minZoom: options.zoomMin,
          maxZoom: options.zoomMax,
          center: options.center,
          bounds,
          maxBounds: options.bounds,
          attribution: map.attribution
        }"
        :label="{
          tooltip: { 'goToCurrentLocation': $t('tooltip.goToCurrentLocation') }
        }"
        style="height: 45vh;"
        @on-go-to-current-location="goToCurrentLocation"
        @on-map-ref="onMapRefEmit"
        @on-bounds-update="boundsUpdated"
      />

      <b-form-text id="password-help-block">
        {{ $t('geometry.mapHelpText') }}
      </b-form-text>
    </div>
    <hr>

    <b-row
      class="mb-2 mt-4"
    >
      <b-col
        sm="12"
        md="4"
      >
        <b-form-group
          :label="$t('geometry.zoom.zoomStartingLabel')"
          label-class="text-primary"
          class="rounded-left"
        >
          <b-form-input
            v-model="options.zoomStarting"
            number
            readonly
            type="number"
          />
        </b-form-group>
      </b-col>

      <b-col
        sm="12"
        md="4"
      >
        <b-form-group
          :label="$t('geometry.zoom.zoomMinLabel')"
          :description="`${options.zoomMin}`"
          label-class="text-primary"
          class="rounded-0"
        >
          <b-form-input
            v-model="options.zoomMin"
            number
            :min="1"
            :max="18"
            type="range"
          />
        </b-form-group>
      </b-col>

      <b-col
        sm="12"
        md="4"
      >
        <b-form-group
          :label="$t('geometry.zoom.zoomMaxLabel')"
          :description="`${options.zoomMax}`"
          label-class="text-primary"
        >
          <b-form-input
            v-model="options.zoomMax"
            number
            :min="1"
            :max="18"
            type="range"
          />
        </b-form-group>
      </b-col>

      <b-col
        sm="12"
        md="4"
      >
        <b-form-group
          :label="$t('geometry.bounds.lockBounds')"
          label-class="text-primary"
          class="rounded-left"
        >
          <b-form-checkbox
            v-model="options.lockBounds"
            name="lock-bounds"
            switch
            size="lg"
            @change="updateBounds"
          />
        </b-form-group>
      </b-col>

      <b-col
        sm="12"
        md="4"
      >
        <b-form-group
          :label="$t('geometry.onMarkerClick')"
          label-class="text-primary"
        >
          <b-form-select
            v-model="options.displayOption"
            :options="displayOptions"
          />
        </b-form-group>
      </b-col>
    </b-row>
  </div>
</template>

<script>
import base from '../base'
import { components } from '@cortezaproject/corteza-vue'
const { CMap } = components

export default {
  i18nOptions: {
    namespaces: 'block',
  },

  components: {
    CMap,
  },

  extends: base,

  data () {
    return {
      map: {
        show: false,
        zoom: 3,
        center: [30, 30],
        rotation: 0,
        attribution: '&copy; <a target="_blank" href="http://osm.org/copyright">OpenStreetMap</a>',
      },

      localValue: { coordinates: [] },
      center: [],
      bounds: null,
      mapRef: undefined,
    }
  },

  computed: {
    displayOptions () {
      return [
        { value: 'sameTab', text: this.$t('geometry.openInSameTab') },
        { value: 'newTab', text: this.$t('geometry.openInNewTab') },
        { value: 'modal', text: this.$t('geometry.openInModal') },
      ]
    },
  },

  beforeDestroy () {
    this.setDefaultValues()
  },

  methods: {
    updateCenter (coordinates) {
      let { lat = 0, lng = 0 } = coordinates || {}

      lat = Math.round(lat * 1e7) / 1e7
      lng = Math.round(lng * 1e7) / 1e7

      this.options.center = [lat, lng]
    },

    boundsUpdated (coordinates) {
      this.bounds = coordinates

      this.updateBounds(this.options.lockBounds)
    },

    zoomUpdated (zoom) {
      this.options.zoomStarting = zoom
    },

    updateBounds (value) {
      if (value) {
        const bounds = this.bounds || this.mapRef.mapObject.getBounds()
        const { _northEast, _southWest } = bounds

        this.options.bounds = [Object.values(_northEast), Object.values(_southWest)]
      } else {
        this.options.bounds = null
      }
    },

    goToCurrentLocation () {
      this.mapRef.mapObject.locate()
    },

    onLocationFound ({ latitude, longitude }) {
      const zoom = this.mapRef.mapObject._zoom >= 13 ? this.mapRef.mapObject._zoom : 13
      this.mapRef.mapObject.flyTo([latitude, longitude], zoom)
    },

    setDefaultValues () {
      this.map = {}
      this.localValue = {}
      this.center = []
      this.bounds = null
    },

    onMapRefEmit (map) {
      this.mapRef = map
    },
  },
}
</script>

<style></style>
