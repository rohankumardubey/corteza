<template>
<l-map
  ref="map"
  :zoom="map.zoom"
  :center="map.center"
  :min-zoom="map.zoomMin"
  :max-zoom="map.zoomMax"
  :bounds="map.bounds"
  :max-bounds="map.maxBounds"
  style="height: 75vh; width: 100%; cursor: pointer;"
  :style="styles"
  @click="placeMarker"
  @locationfound="onLocationFound"
  @update:zoom="onZoom"
  @update:center="onCenter"
  @update:bounds="onBoundsUpdate"
>
  <l-tile-layer
    url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"
    :attribution="map.attribution"
  />

  <l-polygon
    v-for="(geometry, i) in polygonGeometries"
    :key="`polygon-${i}`"
    :lat-lngs="geometry.map(value => value.geometry)"
    :color="polygonColors[i]"
  />

  <l-marker
    v-for="(marker, i) in markers"
    :key="i"
    :lat-lng="marker"
    :opacity="localValueIndex === undefined || i == localValueIndex ? 1.0 : 0.6"
    @click="onRemoveMarker(i)"
  />

  <slot name="markers"></slot>

  <l-control class="leaflet-bar">
    <a
      v-if="!hideCurrentLocationButton"
      :title="label.tooltip && label.tooltip.goToCurrentLocation"
      role="button"
      class="d-flex justify-content-center align-items-center"
      @click="onGoToCurrentLocation"
    >
      <font-awesome-icon
        :icon="['fas', 'location-arrow']"
        class="text-primary"
      />
    </a>
  </l-control>
</l-map>
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
    
    label: {
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

    localValueIndex:{
      type: Number,
      default: undefined
    },

    styles: {
      type: Object,
      default: () => ({})
    },

    polygonGeometries: {
      type: Array,
      default: () => ([])
    },

    polygonColors: {
      type: Array,
      default: () => ([])
    },
  },

  components: { LControl, CInputSearch, LPolygon },
  
  data () {
    return {
      geoSearch: {
        query: '',
        provider: new OpenStreetMapProvider(),
        results: [],
      },
    }
  },

  mounted() {
    this.$emit('on-map-ref', this.$refs.map);
  },

  methods: {
    onLocationFound (coords) {
      this.$emit('on-location-found', coords)
    },

    onRemoveMarker(index) {
      this.$emit('on-remove-marker', index)
    },

    onGoToCurrentLocation () {
      this.$emit('on-go-to-current-location')
    },

    placeMarker(e) {
      this.$emit('on-place-marker', e)
    },

    onZoom (e) {
      this.$emit('on-zoom', e)
    },

    onCenter (e) {
      this.$emit('on-center', e)
    },

    onBoundsUpdate (e) {
      this.$emit('on-bounds-update', e)
    }
  },
};
</script>
