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
import { onMounted } from 'vue';

let map;

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

  var jsonPath = "/public/bornes.json";
  fetch(jsonPath)
  .then(response => response.json())
  .then(data => {
    var markers = L.markerClusterGroup();
    // var marker = L.geoJSON(data);
    var layerGroup = L.geoJSON(data, {
      onEachFeature: function (feature, layer) {
        layer.bindPopup('<h2>'+feature.properties.nom_station+'</h2><p>Adresse: '+feature.properties.adresse_station+'</p>');
      }
    });
    markers.addLayer(layerGroup);
    map.addLayer(markers);
  
  })
  .catch(error => console.error('Erreur lors de la récupération du fichier JSON :', error));
});
</script>

<style scoped>
#map {
  width: 100%;
  height: 100vh;
}
</style>