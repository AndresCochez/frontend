<script setup>
import { ref, computed, onMounted, onUnmounted, watch } from 'vue'

const props = defineProps({
  apiBase: { type: String, default: '' },
  tab: { type: String, default: 'verwerken' }
})

const API = props.apiBase
const orders = ref([])
const loading = ref(false)
const error = ref('')
let pollTimer = null

function setLastTab(t){ try{ localStorage.setItem('lastTab', t) }catch(e){} }

// Confirmed pending tracking
const confirmedPendingIds = ref(JSON.parse(localStorage.getItem('confirmedPendingIds') || '[]'))
const confirmedSet = computed(() => new Set(confirmedPendingIds.value))
function refreshConfirmed(){
  try{
    confirmedPendingIds.value = JSON.parse(localStorage.getItem('confirmedPendingIds') || '[]')
  }catch{
    confirmedPendingIds.value = []
  }
}

// Groupings
const unassigned = computed(() => orders.value.filter(o => {
  const s = String(o.status || '').trim()
  // Show here if no status OR status not in known buckets OR (pending but not confirmed yet)
  if(!s) return true
  if(s === 'pending') return !confirmedSet.value.has(o._id)
  return !(s === 'shipped' || s === 'cancelled')
}))

const pending = computed(() => orders.value.filter(o => o.status === 'pending' && confirmedSet.value.has(o._id)))
const shipped = computed(() => orders.value.filter(o => o.status === 'shipped'))
const cancelled = computed(() => orders.value.filter(o => o.status === 'cancelled'))

async function load(){
  loading.value = true
  error.value = ''
  try{
    const res = await fetch(`${API}/api/orders`)
    if(!res.ok) throw new Error('Kon bestellingen niet laden')
    orders.value = await res.json()
  }catch(e){
    error.value = e.message || 'Er ging iets mis'
  }finally{
    loading.value = false
  }
}

function startPolling(){
  if(pollTimer) return
  pollTimer = setInterval(load, 5000)
}

function stopPolling(){
  if(pollTimer){
    clearInterval(pollTimer)
    pollTimer = null
  }
}

onMounted(() => {
  load()
  startPolling()
  window.addEventListener('pending-confirmed-updated', refreshConfirmed)
})

onUnmounted(() => {
  stopPolling()
  window.removeEventListener('pending-confirmed-updated', refreshConfirmed)
})

watch(() => props.tab, () => {
  // can hook tab-specific behavior later
})
</script>

<template>
  <div class="toolbar" style="justify-content: space-between">
    <div><span class="badge">API: {{ API || '(same-origin)' }}</span></div>
    <div></div>
  </div>

  <p v-if="error" style="color:#b91c1c">{{ error }}</p>

  <template v-if="tab === 'overzicht'">
    <h3>Overzicht</h3>
    <table v-if="unassigned.length" class="table">
      <thead>
        <tr>
          <th>Bestelling</th>
          <th>Datum</th>
          <th>Klant</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="o in unassigned" :key="o._id">
          <td data-label="Bestelling"><a :href="'#/order/'+o._id">#{{ o._id?.slice(-6) || o._id }}</a></td>
          <td data-label="Datum">{{ new Date(o.date).toLocaleString() }}</td>
          <td data-label="Klant">{{ o.customer?.name || '-' }}</td>
        </tr>
      </tbody>
    </table>
    <p v-else>Geen bestellingen.</p>
  </template>

  <template v-else-if="tab === 'verwerken'">
    <h3>Te verwerken</h3>
    <table class="table" v-if="pending.length" @click.capture="setLastTab('verwerken')">
      <thead>
        <tr>
          <th>Bestelling</th>
          <th>Datum</th>
          <th>Klant</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="o in pending" :key="o._id">
          <td data-label="Bestelling"><a :href="'#/order/'+o._id">#{{ o._id?.slice(-6) || o._id }}</a></td>
          <td data-label="Datum">{{ new Date(o.date).toLocaleString() }}</td>
          <td data-label="Klant">{{ o.customer?.name || '-' }}</td>
        </tr>
      </tbody>
    </table>
    <p v-else>Geen te verwerken bestellingen.</p>
  </template>

  <template v-else-if="tab === 'verzonden'">
    <h3>Verzonden</h3>
    <table class="table" v-if="shipped.length" @click.capture="setLastTab('shipped')">
      <thead>
        <tr>
          <th>Bestelling</th>
          <th>Datum</th>
          <th>Klant</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="o in shipped" :key="o._id">
          <td data-label="Bestelling"><a :href="'#/order/'+o._id">#{{ o._id?.slice(-6) || o._id }}</a></td>
          <td data-label="Datum">{{ new Date(o.date).toLocaleString() }}</td>
          <td data-label="Klant">{{ o.customer?.name || '-' }}</td>
        </tr>
      </tbody>
    </table>
    <p v-else>Geen verzonden bestellingen.</p>
  </template>

  <template v-else-if="tab === 'geannuleerd'">
    <h3>Geannuleerd</h3>
    <table class="table" v-if="cancelled.length" @click.capture="setLastTab('cancelled')">
      <thead>
        <tr>
          <th>Bestelling</th>
          <th>Datum</th>
          <th>Klant</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="o in cancelled" :key="o._id">
          <td data-label="Bestelling"><a :href="'#/order/'+o._id">#{{ o._id?.slice(-6) || o._id }}</a></td>
          <td data-label="Datum">{{ new Date(o.date).toLocaleString() }}</td>
          <td data-label="Klant">{{ o.customer?.name || '-' }}</td>
        </tr>
      </tbody>
    </table>
    <p v-else>Geen geannuleerde bestellingen.</p>
  </template>
</template>