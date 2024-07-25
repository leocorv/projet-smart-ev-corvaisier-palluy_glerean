<link rel="stylesheet" href="https://unpkg.com/leaflet@1.0.1/dist/leaflet.css" />

<template>
  <div id="map"></div>
</template>

<script setup>
import 'leaflet/dist/leaflet.css';
import L from 'leaflet';
import 'leaflet.markercluster/dist/leaflet.markercluster.js';
import 'leaflet.markercluster/dist/MarkerCluster.Default.css';
import 'leaflet.markercluster/dist/MarkerCluster.css';
import { onMounted, watch, defineExpose } from 'vue';
import * as turf from '@turf/turf'
import dataBornes from '/public/bornes.json' with { type: 'json' };

const props = defineProps(["start", "end", "isStartFocused","isEndFocused", "autonomie"]);


var greenIcon = new L.Icon({
  iconUrl: 'https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-2x-green.png',
  shadowUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/0.7.7/images/marker-shadow.png',
  iconSize: [25, 41],
  iconAnchor: [12, 41],
  popupAnchor: [1, -34],
  shadowSize: [41, 41]
});

var borneIcon = new L.Icon({
  iconUrl: 'https://cdn-icons-png.flaticon.com/128/9877/9877537.png',
  iconSize: [32, 32], 
  iconAnchor: [16, 32], 
  popupAnchor: [0, -32]
});

let map;
var mapBoxAccessToken = "pk.eyJ1IjoiYWdsZXJlYW4iLCJhIjoiY2x0ZnltbXoyMHNtbzJybGo0aXNpZG1vMSJ9.O-55x2qOPBPkSxOzkLK53g";

const geocoding = (address) => {
  let value_encoded = encodeURIComponent(address);
  var mapBoxAccessToken = "pk.eyJ1IjoiYWdsZXJlYW4iLCJhIjoiY2x0ZnltbXoyMHNtbzJybGo0aXNpZG1vMSJ9.O-55x2qOPBPkSxOzkLK53g";
  return fetch(
    `https://api.mapbox.com/geocoding/v5/mapbox.places/${value_encoded}.json?access_token=${mapBoxAccessToken}`
  )
    .then(response => response.json())
    .then((data) => data.features[0].center)
    .catch((error) => {
      console.error("Erreur de géocodage inversé :", error);
    });
}

const itineraryMapBox = async (startCoordinates,endCoordinates,steps = []) => {
  let directionsUrl;
  if(steps.length != 0) {
    const all_steps = steps.map(step => `${step}`).join(';');
    directionsUrl = `https://api.mapbox.com/directions/v5/mapbox/driving/${startCoordinates[0]},${startCoordinates[1]};${all_steps};${endCoordinates[0]},${endCoordinates[1]}?geometries=geojson&access_token=${mapBoxAccessToken}`;
  } else {
    directionsUrl = `https://api.mapbox.com/directions/v5/mapbox/driving/${startCoordinates[0]},${startCoordinates[1]};${endCoordinates[0]},${endCoordinates[1]}?geometries=geojson&access_token=${mapBoxAccessToken}`;
  }
  const response = await fetch(directionsUrl);
  const data = await response.json();
  return data;
}

const displayRoute = async (data) => {
    // Supprime l'itinéraire précédent s'il existe
    if (window.routeLayer) {
      map.removeLayer(window.routeLayer);
    }

    const routeGeoJSON = data.routes[0].geometry;
    window.routeLayer = L.geoJSON(routeGeoJSON, {
      style: { color: "#3388ff", weight: 4, opacity: 1 }
    }).addTo(map);

    map.fitBounds(window.routeLayer.getBounds());
};


const reverseGeocoding = (coordinates) => {
      return fetch(
        `https://api.mapbox.com/geocoding/v5/mapbox.places/${coordinates.lng},${coordinates.lat}.json?access_token=${mapBoxAccessToken}`
      )
      .then((response) => response.json())
      .then((data) => {
        return data.features[0].place_name;
      })
      .catch((error) => {
        console.error("Erreur de géocodage inversé :", error);
      });
}
var markers_bornes_recharge = [];
var markerStart;
const putStart = async (value) => {
    if(value === ""){
      if(markerStart != undefined) {
        map.removeLayer(markerStart);
        if (window.routeLayer) {
          map.removeLayer(window.routeLayer);
        }
      }
      markers_bornes_recharge.forEach(borne => {
        map.removeLayer(borne);
      });
    }else {
      let coordinates = await geocoding(value);
      if (markerStart != undefined) {
          map.removeLayer(markerStart);
          markerStart = L.marker({lat: coordinates[1], lng: coordinates[0]}).addTo(map);
      }else {
        markerStart = L.marker({lat: coordinates[1], lng: coordinates[0]}).addTo(map);
      }
      markers_bornes_recharge.forEach(borne => {
        map.removeLayer(borne);
      });
    }
}

var markerEnd;
const putEnd = async (value) => {
    if(value === ""){
      if(markerEnd != undefined) {
        map.removeLayer(markerEnd);
        if (window.routeLayer) {
          map.removeLayer(window.routeLayer);
        }
      }
      markers_bornes_recharge.forEach(borne => {
        map.removeLayer(borne);
      });
    } else {
        let coordinates = await geocoding(value);
        if (markerEnd != undefined) {
          map.removeLayer(markerEnd);
          markerEnd = L.marker({lat: coordinates[1], lng: coordinates[0]}, {icon: greenIcon}).addTo(map); 
        }else {
          markerEnd = L.marker({lat: coordinates[1], lng: coordinates[0]}, {icon: greenIcon}).addTo(map); 
        }
        markers_bornes_recharge.forEach(borne => {
          map.removeLayer(borne);
        });
    }
}

const splitItinerary = (data) => {
  const route = {
    "type": "Feature",
    "properties": {},
    "geometry": {
      "type": "LineString",
      "coordinates": data.routes[0].geometry.coordinates,
    }
  };
  const autonomy = props.autonomie * 0.01; 
  const dividedRoute = turf.lineChunk(route, autonomy, { units: 'degrees' });
  let coos = [];
  dividedRoute.features.forEach((segment, index) => {
    coos.push([segment.geometry.coordinates[1][1],segment.geometry.coordinates[1][0]]);
  });
  return coos;
}
const getBornes = (tabCoos) => {
  var resTabBornes = [];

  var allTabBornes = turf.featureCollection(dataBornes.features.map(borne => {
    return turf.point([borne.geometry.coordinates[1],borne.geometry.coordinates[0]]);
  }));

  tabCoos.forEach(coos => {
    var tabBornes = [];
    var coosPoint = turf.point(coos);
    // console.log("coosPoint : ", coosPoint)

    var tabBornes = turf.nearestPoint(coosPoint, allTabBornes);
    // console.log("tabBornes : ", tabBornes)
    let coordinate = [tabBornes.geometry.coordinates[1],tabBornes.geometry.coordinates[0]];
    resTabBornes.push(coordinate);
    let marker_borne = L.marker(tabBornes.geometry.coordinates,{icon: borneIcon}).addTo(map);
    markers_bornes_recharge.push(marker_borne);
    // console.log("tabBornes coordinates : ", tabBornes.geometry.coordinates);
  });

  // console.log("resTabBornes : ", resTabBornes)
  return resTabBornes;
}

const itineraryCalculator = async () => {
  let startCoordinates = await geocoding(document.getElementById("start").value);
  let endCoordinates = await geocoding(document.getElementById("end").value);
  const data = await itineraryMapBox(startCoordinates,endCoordinates);
  let coos = splitItinerary(data); // Récupération des points d'arrêts.
  let bornes = getBornes(coos);
  let final_itinerary = await itineraryMapBox(startCoordinates,endCoordinates,bornes);
  displayRoute(final_itinerary);
}

onMounted(() => {

  map = L
    .map('map')
    .setView([46.36, 2.79], 6);

  L
    .tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
      maxZoom: 19,
      attribution:
        '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>',
    })
    .addTo(map);
    
    var markerEndClick;
    map.on('click', async function(e) {        
        if (document.getElementById("start").value == "" || document.getElementById("end").value == "") {
          
          if (window.routeLayer) {
            map.removeLayer(window.routeLayer);
          }

          if(document.getElementById("start").value === "") {
            markerStartClick = L.marker(e.latlng).addTo(map);   
      
            let markerAddress = await reverseGeocoding(e.latlng);
            let start = document.getElementById('start');
            start.value = markerAddress;
            
          } else {
            if(document.getElementById("end").value === "") {
              markerEndClick = L.marker(e.latlng, {icon: greenIcon}).addTo(map);   
    
              let markerAddress = await reverseGeocoding(e.latlng);
              let end = document.getElementById('end');
              end.value = markerAddress;
            
            } else {
              if(markerEndClick != undefined) {
                map.removeLayer(markerEndClick);
                markerEndClick = L.marker(e.latlng, {icon: greenIcon}).addTo(map);
                let markerAddress = await reverseGeocoding(e.latlng);
                let end = document.getElementById('end');
                end.value = markerAddress;
              }
            }
          }
        }
    });

    
  watch(()=>props.isStartFocused, (isStartFocused)=>{
    if( !isStartFocused ) {
      putStart(props.start);
    }
  });

  watch(()=>props.isEndFocused, (isEndFocused)=>{
    if( !isEndFocused ) {
      putEnd(props.end);
    }
  });


    
});
defineExpose({itineraryCalculator});
</script>

<style scoped>
#map {
  width: 100%;
  height: 100vh;
}
</style>