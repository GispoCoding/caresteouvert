<template>
  <MglMap
    :center="mapCenter"
    :zoom="mapZoom"
    :map-style="mapStyle"
    @load="load"
    @update:center="updateMapCenter"
    @update:zoom="updateMapZoom"
  >
    <MglMarker
      v-if="place"
      :coordinates="place.geometry.coordinates"
      :offset="{ x: 0, y: -15 }"
      color="red"
    />
    <MglNavigationControl />
    <MglGeolocateControl
      :positionOptions="{ enableHighAccuracy: true }"
    />
    <MglVectorLayer
      v-for="layer in layers"
      :key="layer.id"
      :clear-source="false"
      :layer-id="layer.id"
      :layer="layer"
      :source="poiSource"
      source-id="poi"
      @mouseenter="mouseenter"
      @click="clickPoi"
      @mouseleave="mouseleave"
    />
  </MglMap>
</template>

<script>
import 'mapbox-gl/dist/mapbox-gl.css';
import { mapState } from 'vuex';
import * as config from '../config.json';
import {
  MglGeolocateControl,
  MglMap,
  MglMarker,
  MglNavigationControl,
  MglVectorLayer,
} from 'vue-mapbox/dist/vue-mapbox.umd';

const source = "public.poi_osm_light";

function getLayers(theme) {
  const conditionalOpacity = [
    'interpolate', ['linear'],
    ['zoom'],
    0, 0,
    12, [
      'case',
      ["in", ["get", "status"], ["literal", ["unknown", "partial", "closed"]]], 0,
      1
    ],
    14, [
      'case',
      ["in", ["get", "status"], ["literal", ["unknown", "partial", "closed"]]], 0,
      1
    ],
    15, 1
  ];

  return [
    {
      id: "poi-background",
      type: "circle",
      "source-layer": source,
      layout: {
        'circle-sort-key': [
          'case',
          ["in", ["get", "status"], ["literal", ["open", "open_adapted"]]], 2,
          ["in", ["get", "status"], ["literal", ["closed"]]], 1,
          0
        ]
      },
      paint: {
        'circle-color': 'white',
        'circle-opacity': conditionalOpacity,
        'circle-stroke-opacity': conditionalOpacity,
        'circle-stroke-width': [
          'case',
          ["in", ["get", "status"], ["literal", ["open", "open_adapted"]]], 4,
          2.5
        ],
        'circle-stroke-color': [
          'case',
          ["in", ["get", "status"], ["literal", ["open", "open_adapted"]]], theme.success,
          ["in", ["get", "status"], ["literal", ["closed"]]], theme.error,
          "#9E9E9E"
        ],
        'circle-radius': [
          'interpolate',
          ['linear'],
          ['zoom'],
          12, 0,
          14, 2,
          15, [
            'case',
            ["in", ["get", "status"], ["literal", ["unknown", "partial", "closed"]]], 4,
            6
          ],
          19, 13
        ]
      }
    },
    {
      id: "poi-icon",
      type: "symbol",
      "source-layer": source,
      minzoom: 15,
      layout: {
        'symbol-sort-key': [
          'case',
          ["in", ["get", "status"], ["literal", ["open", "open_adapted"]]], 0,
          ["in", ["get", "status"], ["literal", ["closed"]]], 1,
          2
        ],
        "icon-image": [
          "coalesce",
          ['image', ['get', 'cat']],
          ['image', ['get', 'normalized_cat']],
          ['image', 'other']
        ],
        "icon-size": [
          'interpolate',
          ['linear'],
          ['zoom'],
          14, 0.3,
          19, 0.9
        ],
        "text-anchor": "top",
        "text-field": ["get", "name"],
        "text-font": [
          "Noto Sans Regular"
        ],
        "text-max-width": 9,
        "text-offset": [
          'interpolate',
          ['linear'],
          ['zoom'],
          14, ['literal', [0, 0]],
          19, ['literal', [0, 1.5]]
        ],
        "text-padding": 2,
        "text-size": 12
      },
      paint: {
        "icon-opacity": conditionalOpacity,
        "text-opacity": [
          'interpolate', ['linear'],
          ['zoom'],
          0, 0,
          16, 0,
          17, 1
        ],
        "text-color": "#666",
        "text-halo-blur": 0.5,
        "text-halo-color": "#ffffff",
        "text-halo-width": 1
      }
    }
  ];
}

export default {
  components: {
    MglGeolocateControl,
    MglMap,
    MglMarker,
    MglNavigationControl,
    MglVectorLayer,
  },

  props: {
    featuresAndLocation: {
      type: String,
      required: false,
      default: ''
    },

    mapStyle: {
      type: Object,
      required: true
    },

    mapCenter: {
      type: Object,
      required: true
    },

    mapZoom: {
      type: Number,
      required: true
    },

    filter: {
      type: String,
      required: true
    },

    sidebar: {
      type: Boolean,
      required: true
    }
  },

  mounted() {
    this.$on('updateMapBounds', (bbox) => {
      this.map.fitBounds(bbox, { duration: 0 });
    });
  },

  computed: {
    ...mapState(['place']),

    poiSource() {
      return {
        minzoom: config.minZoomPoi,
        maxzoom: config.maxZoomPoi,
        tiles: [config.tilesUrl]
      };
    },

    layers() {
      const [ category, subcategory ] = this.filter.split('/');
      return getLayers(this.$vuetify.theme.themes.light).map((layer) => {
        const newLayer = { ...layer, filter: ['all'] };
        if (subcategory) {
          newLayer.filter.push(['==', 'cat', subcategory]);
        } else if (category !== '') {
          newLayer.filter.push(['==', 'normalized_cat', category]);
        }
        return newLayer;
      });
    }
  },

  watch: {
    place(place) {
      if (place) {
        const isPlaceUnderUI = (x, y) => {
          const height = document.body.clientHeight;
          const width = document.body.clientWidth;
          const offsetSidebar = this.sidebar ? 300 : 0;
          const offsetPlace = this.$vuetify.breakpoint.smAndDown ? 0 : 300;
          return (x < offsetSidebar || x > width - offsetPlace || y > height || y < 0);
        }
        const { x, y } = this.map.project(place.geometry.coordinates);
        if (isPlaceUnderUI(x, y)) {
          this.map.panTo(place.geometry.coordinates);
        }
      }
    }
  },

  methods: {
    load({ map }) {
      this.map = map;
    },

    updateMapCenter(mapCenter) {
      this.$emit('update:mapCenter', mapCenter);
    },

    updateMapZoom(mapZoom) {
      this.$emit('update:mapZoom', mapZoom);
    },

    mouseenter(e) {
      e.map.getCanvas().style.cursor = 'pointer';
    },

    mouseleave(e) {
      e.map.getCanvas().style.cursor = '';
    },

    clickPoi(e) {
      const id = e.mapboxEvent.features[0].properties.fid;
      if (this.$route.name === 'place' && this.$route.params.id === id) {
        return;
      }
      this.$router.push({
        name: 'place',
        params: {
          id,
          featuresAndLocation: this.featuresAndLocation
        }
      });
    }
  }
}
</script>

<style>
.mgl-map-wrapper {
  width: 100vw;
  height: 100vh;
  position: relative;
}

.mgl-map-wrapper .mapboxgl-map {
  height: 100%;
  left: 0;
  position: absolute;
  top: 0;
  width: 100%;
}
</style>
