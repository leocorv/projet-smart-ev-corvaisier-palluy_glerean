<script setup>
  import MapView from '../views/MapBornes.vue'
</script>

<template>
  <div class="row">
    <div class="col-3">
      <form @submit.prevent="submitForm" class="mt-3" style="padding-left: 1rem;">
        <h2> Chercher une borne </h2>
        <div class="mb-3">
          <label for="start" class="form-label">Adresse:</label>
          <input id="start" v-model="start" type="text" class="form-control" list="start-addresses" @input="searchAddress('start', $event.target.value)" required>
          <datalist id="start-addresses">
            <option v-for="address in startSuggestions" :value="address" :key="address">{{ address }}</option>
          </datalist>
        </div>    

        <button type="submit" class="btn btn-primary">Rechercher</button>
      </form>
      <hr>
      <a href="/itinerary"><button class="btn btn-primary">Calculer un itineraire 📍</button></a>
    </div>
    <div class="map-view col-9">
      <MapView :start="start" :end="end"/>
    </div>
  </div>  
</template>

<script>

</script>