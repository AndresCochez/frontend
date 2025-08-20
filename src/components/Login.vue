<script setup>
import { ref, watch } from 'vue'
const emit = defineEmits(['logged-in'])
const username = ref('')
const password = ref('')
const error = ref('')

function submit() {
  if (username.value.trim() === 'Admin' && password.value === 'Admin') {
    error.value = ''
    emit('logged-in')
  } else {
    error.value = 'Ongeldige inloggegevens'
  }
}
watch([username, password], () => { if (error.value) error.value = '' })
</script>

<template>
  <h2>Inloggen</h2>
  <div class="toolbar">
    <input placeholder="Gebruikersnaam" v-model="username" @keyup.enter="submit" />
    <input placeholder="Wachtwoord" type="password" v-model="password" @keyup.enter="submit" />
    <button @click="submit">Inloggen</button>
  </div>
  <p v-if="error" style="color:#b91c1c">{{ error }}</p>
</template>
