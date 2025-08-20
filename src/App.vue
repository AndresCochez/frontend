<script setup>
import { ref, computed, onMounted } from 'vue'
import Login from './components/Login.vue'
import OrdersView from './components/OrdersView.vue'
import OrderDetail from './components/OrderDetail.vue'

const API = (import.meta.env.VITE_API_URL || localStorage.getItem('API_URL') || 'http://localhost:5000')
  .toString()
  .trim()


const isLoggedIn = ref(localStorage.getItem('loggedIn') === 'true')
const activeTab = ref(isLoggedIn.value ? 'overzicht' : 'login')

function handleLogin(){
  isLoggedIn.value = true
  localStorage.setItem('loggedIn','true')
  activeTab.value = 'overzicht'
}

function logout(){
  isLoggedIn.value = false
  localStorage.removeItem('loggedIn')
  activeTab.value = 'login'
  window.location.hash = ''
}

// Hash router for detail
const route = ref(window.location.hash)
const isDetail = computed(() => /^#\/order\//.test(route.value || ''))
onMounted(() => {
  window.addEventListener('hashchange', () => { route.value = window.location.hash })
  window.addEventListener('goto-overzicht', () => { activeTab.value = 'overzicht' })
})
function goToOverview(){ window.location.hash = '' }
</script>

<template>
  <header class="header">
    <div class="header-inner">
      <div class="brand"><span class="dot"></span>Admin - Ben &amp; Jerry's</div>

      <div class="nav" v-if="isLoggedIn">
        <button :class="{secondary: activeTab !== 'overzicht'}" @click="activeTab = 'overzicht'">Overzicht</button>
        <button :class="{secondary: activeTab !== 'verwerken'}" @click="activeTab = 'verwerken'">Te verwerken</button>
        <button :class="{secondary: activeTab !== 'verzonden'}" @click="activeTab = 'verzonden'">Verzonden</button>
        <button :class="{secondary: activeTab !== 'geannuleerd'}" @click="activeTab = 'geannuleerd'">Geannuleerd</button>
      </div>

      <div class="toolbar">
        <button v-if="isLoggedIn" class="secondary" @click="logout">Uitloggen</button>
      </div>
    </div>
  </header>

  <main class="container">
    <div v-if="!isLoggedIn" class="card">
      <Login @logged-in="handleLogin"/>
    </div>

    <div v-else class="card">
      <OrderDetail v-if="isDetail" :apiBase="API" />
      <OrdersView v-else :apiBase="API" :tab="activeTab" />
    </div>
  </main>
</template>
