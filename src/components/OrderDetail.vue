<script setup>
import { ref, onMounted, computed } from 'vue'

const props = defineProps({
  apiBase: { type: String, default: '' }
})

const API = props.apiBase || (localStorage.getItem('API_URL')||'').trim() || 'http://localhost:5000'
const orderId = ref(null)
const order = ref(null)
const loading = ref(true)
const error = ref('')
const success = ref('')
const cleared = ref(false)

function isConfirmedPending(id){
  try{
    const arr = JSON.parse(localStorage.getItem('confirmedPendingIds') || '[]')
    return Array.isArray(arr) && arr.includes(id)
  }catch{ return false }
}

const showPendingButton = computed(() => {
  if(!order.value) return false
  const s = String(order.value.status || '').trim()
  if(s !== 'pending') return true
  // already pending: only show if it's NOT yet confirmed (i.e., still in Overzicht)
  return !isConfirmedPending(order.value._id)
})


function statusToTab(s){
  switch(s){
    case 'pending': return 'pending'
    case 'ready': return 'ready'
    case 'shipped': return 'shipped'
    case 'cancelled': return 'cancelled'
    default: return ''
  }
}

function statusLabel(s){
  switch(s){
    case 'pending': return 'Te verwerken'
    case 'shipped': return 'Verzonden'
    case 'cancelled': return 'Geannuleerd'
    default: return s
  }
}

function parseHash() {
  const h = window.location.hash || ''
  const m = h.match(/^#\/order\/(.+)$/)
  orderId.value = m ? m[1] : null
}

async function load() {
  loading.value = true
  error.value = ''
  try {
    const res = await fetch(`${API}/api/orders`)
    if (!res.ok) throw new Error('Kon bestellingen niet laden')
    const items = await res.json()
    order.value = items.find(o => o._id === orderId.value) || null
    try {
      const cur=(localStorage.getItem('lastTab')||'').trim();
      if(!cur && order.value && order.value.status){ localStorage.setItem('lastTab', order.value.status) }
    } catch {}
    if (!order.value) error.value = 'Bestelling niet gevonden'
  } catch (e) {
    error.value = e.message || 'Er ging iets mis'
  } finally {
    loading.value = false
  }
}

async function markStatus(id, status){
  try{
    error.value = ''; success.value = '';
    // Primair: POST
    let res = await fetch(`${API}/api/orders/${id}/status`, {
      method:'POST',
      headers:{'Content-Type':'application/json'},
      body: JSON.stringify({ status })
    });
    // Fallback: PATCH
    if(!res.ok){
      res = await fetch(`${API}/api/orders/${id}/status`, {
        method:'PATCH',
        headers:{'Content-Type':'application/json'},
        body: JSON.stringify({ status })
      });
    }
    if(!res.ok) throw new Error('Kon status niet updaten');

    if(order.value && order.value._id===id){ order.value.status = status }

    // Lokale tracker (ongewijzigd gedrag)
    try {
      const key = 'confirmedPendingIds';
      const arr = JSON.parse(localStorage.getItem(key) || '[]');
      if(status === 'pending'){
        if(!arr.includes(id)) arr.push(id);
      } else {
        const i = arr.indexOf(id); if(i !== -1) arr.splice(i,1);
      }
      localStorage.setItem(key, JSON.stringify(arr));
      window.dispatchEvent(new CustomEvent('pending-confirmed-updated'));
    // Direct terug naar het vorige tabblad (zonder succesmessage)
    success.value = '';
    const last = (localStorage.getItem('lastTab')||'').trim();
    const fallback = statusToTab(status) || 'overzicht';
    window.location.hash = last ? `#/${last}` : `#/${fallback}`;
    } catch {}

      }catch(e){
    error.value = e.message || 'Er ging iets mis bij status wijzigen';
  }
}

function goBack() { const t=(localStorage.getItem('lastTab')||'').trim(); window.location.hash = t? `#/${t}` : ''; }

async function deleteOrder(id){
  try{
    const res = await fetch(`${API}/api/orders/${id}`, { method:'DELETE' })
    if(!res.ok) throw new Error('Kon bestelling niet verwijderen')
    // Geen succesmessage tonen en meteen terugnavigeren
    success.value = ''
    order.value = null
    const last = (localStorage.getItem('lastTab')||'').trim()
    // Forceer terug naar geannuleerd als je van daar kwam; anders naar het laatst gebruikte tab of overzicht
    const target = (last === 'geannuleerd' || last === 'cancelled') ? 'geannuleerd' : (last || 'overzicht')
    window.location.hash = `#/${target}`
  }catch(e){
    error.value = e.message || 'Er ging iets mis bij verwijderen'
  }

}

onMounted(() => {
  parseHash()
  load()
  window.addEventListener('hashchange', () => { parseHash(); load() })
})
</script>

<template>
  <div class="toolbar" style="justify-content: space-between">
    <button class="secondary" @click="goBack">← Terug naar overzicht</button>
    <div v-if="order && !cleared">
      <strong>Bestelling</strong> <code>{{ order._id }}</code>
    </div>
  </div>

  <div v-if="loading">Laden…</div>
  <p v-if="success" class="badge" style="background: var(--accent-weak); color: var(--accent)">✔ {{ success }}</p>
  <p v-else-if="error" style="color:#b91c1c">{{ error }}</p>

  <div v-else-if="order && !cleared" class="card">
    <h2>Details</h2>
    <div class="toolbar">
      <button v-if="showPendingButton" @click="markStatus(order._id, 'pending')">Zet naar te verwerken</button>
      <button v-if="order?.status!=='shipped'" @click="markStatus(order._id, 'shipped')">Markeer verzonden</button>
      <button v-if="order?.status!=='cancelled'" class="secondary" @click="markStatus(order._id, 'cancelled')">Annuleer</button>
      <button class="secondary" @click="deleteOrder(order._id)">Verwijder</button>
    </div>

    <div class="grid-2">
      <div>
        <h3>Algemeen</h3>
        <p><strong>Datum:</strong> {{ new Date(order.date).toLocaleString() }}</p>
        <p><strong>Status:</strong> {{ order.status }}</p>
        <p><strong>Totaalprijs:</strong> € {{ Math.round(order.price || 0) }}</p>
      </div>
      <div>
        <h3>Klant</h3>
        <p><strong>Naam:</strong> {{ order.customer?.name || '-' }}</p>
        <p><strong>Adres:</strong> {{ order.customer?.address?.street || '-' }}, {{ order.customer?.address?.city || '-' }}</p>
      </div>
    </div>

    <h3>Configuratie</h3>
<ul>
  <li><strong>Bolletje:</strong> {{ order.scoop?.flavor }} </li>
  <li><strong>Hoorntje:</strong> {{ order.cone?.style }} </li>
  <li><strong>Sprinkels:</strong> {{ order.sprinkles?.level }} </li>
</ul>
  </div>
</template>

<style scoped>
.grid-2{ display:grid; grid-template-columns: 1fr 1fr; gap: 1rem; }
code{ font-size: .9rem; }
</style>
