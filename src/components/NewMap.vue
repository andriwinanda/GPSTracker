<template>
  <div class="main">
    <div class="flex">
      <!-- Map Display here -->
      <div class="map-holder">
        <div id="mapContainer"></div>
      </div>

      <!-- Coordinates Display here -->
      <div class="dislpay-arena">
        <div class="coordinates-header">
          <h2>GPS Tracker</h2>
          <template v-if="isArrive">
            <h3>Arrived at Destination</h3>
            <line-chart :chartData="datacollection"></line-chart>
          </template>

          <button
            type="button"
            :disabled="loading"
            :class="{ disabled: loading }"
            class="location-btn"
            @click="startDriving()"
          >
            Start
          </button>
          <button
            type="button"
            :disabled="loading"
            :class="{ disabled: loading }"
            class="location-btn"
            @click="pause()"
          >
            Pause
          </button>
          <button
            type="button"
            :disabled="loading"
            :class="{ disabled: loading }"
            class="location-btn"
            @click="stop()"
          >
            Stop
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import mapboxgl from "mapbox-gl";
import bearing from "@turf/bearing";
import "mapbox-gl/dist/mapbox-gl.css";
import routeMap from "../routeMap.json";
import lineChart from "./LineChart.js";
const Routes = routeMap[0].geometry;
import moment from 'moment';
export default {
  components: { lineChart },
  data() {
    return {
      loading: false,
      location: "",
      access_token:
        "pk.eyJ1IjoiZmFraHIiLCJhIjoiY2pseXc0djE0MHBibzN2b2h4MzVoZjk4aSJ9.ImbyLtfsfSsR_yyBluR8yQ",
      origin: {
        place: "",
        coord: [98.68393647698798, 3.575295945789321],
      },
      destination: {
        place: "",
        coord: [98.677243, 3.576319],
      },
      map: {},
      currentPolyline: {
        coordinates: [],
        type: "LineString",
      },
      speed: [],
      timeStamp: [],
      index: 0,

      isArrive: false,
      driving: null,
      datacollection: {
        labels: [],
        datasets: [
          {
            label: "Speed (m/s)",
            backgroundColor: "#4cd964",
            data: [],
          },
        ],
      },
    };
  },
  mounted() {
    this.createMap();
  },
  beforeDestroy() {
    clearInterval(this.driving);
  },
  methods: {
    async createMap() {
      try {
        mapboxgl.accessToken = this.access_token;
        this.map = new mapboxgl.Map({
          container: "mapContainer",
          style: "mapbox://styles/mapbox/streets-v11",
          center: this.origin.coord,
          zoom: 10,
          transition: {
            duration: 300,
            delay: 0,
          },
        });
        const nav = new mapboxgl.NavigationControl();
        this.map.addControl(nav, "top-right");
        this.map.resize();
        this.map.loadImage(
          "https://cdn.pixabay.com/photo/2014/04/02/16/28/car-307371_1280.png",
          (error, image) => {
            if (error) throw error;

            // Add the image to the map style.
            this.map.addImage("car", image);
          }
        );
      } catch (err) {
        console.log("map error", err);
      }
    },

    async startDriving() {
      if (this.isArrive) this.stop();
      this.map.fitBounds([this.origin.coord, this.destination.coord], {
        maxZoom: 14,
        padding: 5,
      });
      this.driving = setInterval(() => {
        if (this.index >= Routes.length) {
          this.index = 0;
          this.isArrive = true;
          clearInterval(this.driving);
        } else {
          this.isArrive = false;
          this.updatePoint(Routes[this.index].coordinate);
          this.updatePolyline();
          this.index++;
          this.currentPolyline.coordinates.push(Routes[this.index].coordinate);
          this.datacollection.datasets[0].data.push(Routes[this.index].speed);
          this.datacollection.labels.push(moment(new Date()).format("DD-MM-YY hh:mm:ss "));
        }
      }, 3000);
    },

    pause() {
      clearInterval(this.driving);
    },
    stop() {
      clearInterval(this.driving);
      this.index = 0;
      this.currentPolyline.coordinates = [];
      this.timeStamp = [];
      this.speed = [];
      this.updatePoint(this.origin.coord);
      this.updatePolyline();
    },
    async updateMarker(coor) {
      marker.setLngLat(coor);
      marker.addTo(this.map);
      requestAnimationFrame(marker);
    },

    async updatePoint(coor) {
      const point = {
        type: "FeatureCollection",
        features: [
          {
            type: "Feature",
            properties: {},
            geometry: {
              type: "Point",
              coordinates: coor,
            },
          },
        ],
      };
      if (this.map.getSource("point")) {
        this.map.removeLayer("point");
        this.map.removeSource("point");
      }
      point.features[0].properties.bearing = bearing(
        this.currentPolyline.coordinates[this.index - 2] || this.origin.coord,
        coor
      );

      this.map.addLayer({
        id: "point",
        source: {
          type: "geojson",
          data: point,
        },
        type: "symbol",
        layout: {
          // This icon is a part of the Mapbox Streets style.
          // To view all images available in a Mapbox style, open
          // the style in Mapbox Studio and click the "Images" tab.
          // To add a new image to the style at runtime see
          // https://docs.mapbox.com/mapbox-gl-js/example/add-image/
          "icon-image": "car",
          "icon-size": 0.02,
          "icon-rotate": ["get", "bearing"],
          "icon-rotation-alignment": "map",
          "icon-allow-overlap": true,
          "icon-ignore-placement": false,
        },
      });
    },
    async updatePolyline() {
      if (this.map.getSource("route")) {
        this.map.removeLayer("route");
        this.map.removeSource("route");
      }
      this.map.addLayer({
        id: "route",
        type: "line",
        source: {
          type: "geojson",
          data: {
            type: "Feature",
            properties: {},
            geometry: this.currentPolyline,
          },
        },
        layout: {
          "line-join": "round",
          "line-cap": "round",
        },
        paint: {
          "line-color": "#d80739",
          "line-width": 4,
          "line-opacity": 0.5,
        },
      });
    },
  },
};
</script>

<style>
.main {
  padding: 25px;
}
.flex {
  display: flex;
  flex-wrap: wrap;
  flex-direction: row;
  justify-content: space-between;
}
.map-holder {
  width: 65%;
}
#mapContainer {
  width: 100%;
  height: 80vh;
}
.dislpay-arena {
  background: #ffffff;
  box-shadow: 0px -3px 10px rgba(0, 58, 78, 0.1);
  border-radius: 5px;
  padding: 20px 30px;
  width: 25%;
}
.coordinates-header {
  margin-bottom: 50px;
}
.coordinates-header h3 {
  color: #1f2a53;
  font-weight: 600;
}

.coordinates-header p {
  color: rgba(13, 16, 27, 0.75);
  font-weight: 600;
  font-size: 0.875rem;
}
.location-control {
  height: 30px;
  background: #ffffff;
  border: 1px solid rgba(31, 42, 83, 0.25);
  box-shadow: 0px 0px 10px rgba(73, 165, 198, 0.1);
  border-radius: 4px;
  padding: 0px 10px;
  width: 90%;
}
.location-control:focus {
  outline: none;
}
.location-btn {
  margin: 15px 5px 5px 5px;
  padding: 10px 15px;
  background: #d80739;
  box-shadow: 0px 4px 10px rgba(73, 165, 198, 0.1);
  border-radius: 5px;
  border: 0;
  cursor: pointer;
  color: #ffffff;
  font-size: 0.875rem;
  font-weight: 600;
}


@media screen and (max-width: 768px) {
  .dislpay-arena {
    width: 100%;
  }
  #mapContainer {
    width: 100%;
  }
}
</style>