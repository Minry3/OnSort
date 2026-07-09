<script setup>
import { ref } from 'vue'
import { BForm, BFormGroup, BFormInput, BButton, BAlert, BSpinner } from 'bootstrap-vue-next'
import axios from 'axios';

const ville = ref(''); 
const chargement = ref(false);
const erreur = ref('');
const resultat = ref(null);

async function verifier(event) {
  event.preventDefault();
  chargement.value = true;
  erreur.value = '';
  resultat.value = null;

  try {
    const url = `https://geocoding-api.open-meteo.com/v1/search?name=${ville.value}&count=1&language=fr`
    const reponseApi = await axios.get(url);
    const data = reponseApi.data; 

    if (!data.results || data.results.length === 0) {
      erreur.value = "Ville introuvable. Essaie une autre orthographe !";
      chargement.value = false;
      return;
    }

    const lat = data.results[0].latitude;
    const lon = data.results[0].longitude;
    const nomVille = data.results[0].name;

    const meteoUrl = `https://api.open-meteo.com/v1/forecast?latitude=${lat}&longitude=${lon}&current=temperature_2m,surface_temperature`;
    const meteoReponse = await axios.get(meteoUrl);
    const meteoData = meteoReponse.data; 

    const tempAir = Math.round(meteoData.current.temperature_2m);
    const tempSol = Math.round(meteoData.current.surface_temperature);

    let message = '';
    let theme = '';

    if (tempSol < 35) {
      message = "✅ C'est bon ! Balade en toute sécurité.";
      theme = "success";
    } else if (tempSol < 40) {
      message = "⚠️ Attention, sol chaud. Privilégiez l'herbe !";
      theme = "warning";
    } else {
      message = "🚨 Danger ! Risque de brûlures sévères.";
      theme = "danger";
    }

    resultat.value = {
      ville: nomVille,
      air: tempAir,
      sol: tempSol,
      msg: message,
      theme: theme
    }
  } catch (error) {
    erreur.value = "Oups, impossible de joindre la météo.";
  } finally {
    chargement.value = false; 
  }
}
</script>

<template>
  <div class="container mt-5" style="max-width: 500px;">
    <h2>🐾 On sort ?</h2>
    <p>La chaleur du sol peut être très dangereuse pour les petites pattes de vos toutous. Vérifiez la température avant de le sortir !</p>

    <BForm @submit="verifier">
      <BFormGroup id="ville-group" label="Ville :" label-for="ville" class="mb-3">
        <BFormInput 
          id="ville" 
          v-model="ville" 
          placeholder="Votre ville (Ex: Paris)" 
          required 
        />
      </BFormGroup>

      <BButton type="submit" variant="primary" class="w-100" :disabled="chargement">
        <BSpinner v-if="chargement" small class="me-2"/>
        {{ chargement ? 'Calcul des températures...' : 'Vérifier la sécurité' }}
      </BButton>
    </BForm>

    <BAlert v-if="erreur" model-value="true" variant="danger" class="mt-4">
      {{ erreur }}
    </BAlert>

    <div v-if="resultat" class="card mt-4" :class="`border-${resultat.theme}`">
      <div class="card-body">
        <h4>{{ resultat.ville }}</h4>
        <hr>
        <div class="d-flex justify-content-around text-center mb-3">
          <div>
            <span class="text-muted">Air</span>
            <h5>{{ resultat.air }}°C</h5>
          </div>
          <div>
            <span class="text-muted">Sol</span>
            <h5>{{ resultat.sol }}°C</h5>
          </div>
        </div>
        <BAlert model-value="true" :variant="resultat.theme" class="mb-0">
          {{ resultat.msg }}
        </BAlert>
      </div>
    </div>
  </div>
</template>