<script setup>
  import MapItinerary from '../views/MapItinerary.vue'
</script>

<template>
  <div class="row">
    <div class="col-3" id="form-content">
      <form class="mt-3" id="itineraryForm">
        <h2>Calculer un itinéraire</h2>
        <div class="mb-3">
          <label for="start" class="form-label">Adresse de départ:</label>
          <input id="start" v-model="start" type="text" class="form-control" list="start-addresses" @input="searchAddress('start', $event.target.value)" @focusout="isStartFocused = false" @focusin="isStartFocused = true" required>
          <datalist id="start-addresses">
            <option v-for="address in startSuggestions" :value="address" :key="address">{{ address }}</option>
          </datalist>
        </div>

        <div class="mb-3">
          <label for="end" class="form-label">Adresse d'arrivée:</label>
          <input id="end" v-model="end" type="text" class="form-control" list="end-addresses" @input="searchAddress('end', $event.target.value)" @focusout="isEndFocused = false" @focusin="isEndFocused = true" required>
          <datalist id="end-addresses">
            <option v-for="address in endSuggestions" :value="address" :key="address">{{ address }}</option>
          </datalist>
        </div>     

        <div class="mb-3">
          <label for="vehicle-select" class="form-label">Choisissez un véhicule:</label>
          <select @change="vehicle_autonomie()" class="form-control mr-sm-2" id="vehicle-select" v-model="selectedVehicle">
            <option selected>-</option>
            <option v-for="vehicle in vehicles" :value="vehicle.make + ' ' + vehicle.model" :key="vehicle.make + vehicle.model">
              {{ vehicle.make }} {{ vehicle.model }}
            </option>
          </select>
        </div>
        <div class="mb-3">
          <button type="button" class="btn btn-primary" @click.prevent="this.$refs.mapItinerary.itineraryCalculator"> Get Route </button>
        </div>
      </form>
    </div>
    <div class="map-view col-9">
      <MapItinerary :submitMap="submitMap" :start="start" :end="end" :autonomie="autonomie" :is-start-focused="isStartFocused" :is-end-focused="isEndFocused" ref="mapItinerary"/>
    </div>
  </div>  
</template>

<style scoped>
  .row {
    --bs-gutter-x: 0rem;
  }
  .btn-primary {
    width: 100%;
    background-color: black;
    border: 1px solid black;
    transition: opacity .8s;
  }
  .btn-primary:hover {
    opacity: .5;
  }
  .form-control {
    border: solid 1px grey;
  }
  .form-control:focus {
    box-shadow: 0 0 5pt 2pt #D3D3D3;
  }
  #form-content {
    padding: 1rem;
  }
  #itinerary-result {
    display: none;
  }
</style>

<script>

var mapBoxAccessToken = "pk.eyJ1IjoiYWdsZXJlYW4iLCJhIjoiY2x0ZnltbXoyMHNtbzJybGo0aXNpZG1vMSJ9.O-55x2qOPBPkSxOzkLK53g";
  
import vehicleData from '../assets/vehicle.json';

  export default {
    components: {
      MapItinerary
    },
    data() {
      return {
        start: '',
        end: '',
        isStartFocused: false,
        isEndFocused: false,
        selectedVehicle: '',
        vehicles: vehicleData.vehicles,
        autonomie: ''
      };
    },
    mounted() {
      if (this.vehicles.length > 0) {
        // Sélectionne par défaut l'option "-"
        this.selectedVehicle = "-";
      }
    },
    methods: {
      searchAddress(type, query) {
        if (query.length < 1) {
          if (type === 'start') {
            this.startSuggestions = [];
          } else if (type === 'end') {
            this.endSuggestions = [];
          }
          return;
        }

        fetch(`https://api-adresse.data.gouv.fr/search/?q=${encodeURIComponent(query)}&limit=15`)
          .then(response => response.json())
          .then(data => {
            let addresses = data.features.map(feature => feature.properties.label);

            if (type === 'start') {
              this.startSuggestions = addresses;
            } else if (type === 'end') {
              this.endSuggestions = addresses;
            }

            // Si aucune adresse n'est trouvée
            if (addresses.length === 0 || (addresses.length === 1 && addresses[0] !== query)) {
              if (type === 'start') {
                this.startSuggestions = addresses.push(query);
              } else if (type === 'end') {
                this.endSuggestions = addresses.push(query);
              }
            }
          })
          .catch(error => {
            console.error('Erreur lors de la recherche d\'adresses:', error);
          });
      },
      vehicle_autonomie() {
        // console.log(this.selectedVehicle);
        if (this.selectedVehicle === '-') {
          return;
        }
        let vehicle = this.vehicles.find(vehicle => vehicle.make + ' ' + vehicle.model === this.selectedVehicle);
        this.autonomie = vehicle.autonomie;
        // console.log(this.autonomie);
      },
      submitForm() {
        if (this.start === '' || this.end === '') {
          alert('Veuillez renseigner une adresse de départ et une adresse d\'arrivée.');
          return;
        }

        if (this.start === this.end) {
          alert('L\'adresse de départ et l\'adresse d\'arrivée ne peuvent pas être identiques.');
          return;
        }

        this.$emit('submitMap', {
          start: this.start,
          end: this.end,
          vehicle: this.selectedVehicle
        });
      }
    }
  };
</script>