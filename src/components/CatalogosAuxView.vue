<script setup>
import { ref, onMounted, inject } from 'vue';

const API_URL = inject('API_URL');
const addAlert = inject('addAlert');

const props = defineProps({
  activeTabProp: { type: String, default: 'presentaciones' }
});

const activeTab = ref(props.activeTabProp); // 'presentaciones' | 'casas' | 'pagos'

// --- ESTADOS DE PRESENTACIONES ---
const presentacionesList = ref([]);
const presNombre = ref('');
const presEstado = ref(true);
const presEditingId = ref(null);

// --- ESTADOS DE CASAS MEDICAS ---
const casasMedicasList = ref([]);
const casaNombre = ref('');
const casaEstado = ref(true);
const casaEditingId = ref(null);

// --- ESTADOS DE METODOS DE PAGO ---
const metodosList = ref([]);
const pagoNombre = ref('');
const pagoCuenta = ref('');
const pagoEstado = ref(true);
const pagoEditingId = ref(null);

// --- PETICIONES API ---
const loadPresentaciones = async () => {
  try {
    const res = await fetch(`${API_URL}/presentaciones?limite=100`);
    if (res.ok) {
      const data = await res.json();
      presentacionesList.value = data.datos || [];
    }
  } catch (e) { console.error(e); }
};

const loadCasasMedicas = async () => {
  try {
    const res = await fetch(`${API_URL}/casas-medicas?limite=100`);
    if (res.ok) {
      const data = await res.json();
      casasMedicasList.value = data.datos || [];
    }
  } catch (e) { console.error(e); }
};

const loadMetodos = async () => {
  try {
    const res = await fetch(`${API_URL}/metodos-pago?limite=100`);
    if (res.ok) {
      const data = await res.json();
      metodosList.value = data.datos || [];
    }
  } catch (e) { console.error(e); }
};

onMounted(() => {
  loadPresentaciones();
  loadCasasMedicas();
  loadMetodos();
});

// --- GUARDAR Y ELIMINAR ACCIONES ---

// 1. Presentaciones
const savePresentacion = async () => {
  if (!presNombre.value) return;
  const payload = { nombre_presentacion: presNombre.value, estado_presentacion: presEstado.value };
  try {
    let res;
    if (presEditingId.value) {
      res = await fetch(`${API_URL}/presentaciones/${presEditingId.value}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });
    } else {
      res = await fetch(`${API_URL}/presentaciones`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });
    }
    if (res.ok) {
      addAlert(`Presentación guardada correctamente.`, 'success');
      presNombre.value = '';
      presEstado.value = true;
      presEditingId.value = null;
      loadPresentaciones();
    }
  } catch (e) { console.error(e); }
};

const deletePresentacion = async (id, name) => {
  if (confirm(`¿Eliminar la presentación "${name}"?`)) {
    try {
      const res = await fetch(`${API_URL}/presentaciones/${id}`, { method: 'DELETE' });
      if (res.ok || res.status === 204) {
        addAlert(`Presentación "${name}" eliminada.`, 'success');
        loadPresentaciones();
      }
    } catch (e) { console.error(e); }
  }
};

// 2. Casas Médicas
const saveCasaMedica = async () => {
  if (!casaNombre.value) return;
  const payload = { nombre_casa_medica: casaNombre.value, estado_casa_medica: casaEstado.value };
  try {
    let res;
    if (casaEditingId.value) {
      res = await fetch(`${API_URL}/casas-medicas/${casaEditingId.value}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });
    } else {
      res = await fetch(`${API_URL}/casas-medicas`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });
    }
    if (res.ok) {
      addAlert(`Casa médica guardada correctamente.`, 'success');
      casaNombre.value = '';
      casaEstado.value = true;
      casaEditingId.value = null;
      loadCasasMedicas();
    }
  } catch (e) { console.error(e); }
};

const deleteCasaMedica = async (id, name) => {
  if (confirm(`¿Eliminar la casa médica "${name}"?`)) {
    try {
      const res = await fetch(`${API_URL}/casas-medicas/${id}`, { method: 'DELETE' });
      if (res.ok || res.status === 204) {
        addAlert(`Casa médica "${name}" eliminada.`, 'success');
        loadCasasMedicas();
      }
    } catch (e) { console.error(e); }
  }
};

// 3. Métodos de Pago
const saveMetodo = async () => {
  if (!pagoNombre.value) return;
  const payload = { 
    nombre_metodo_pago: pagoNombre.value, 
    cuenta_metodo_pago: pagoCuenta.value || undefined, 
    estado_metodo_pago: pagoEstado.value 
  };
  try {
    let res;
    if (pagoEditingId.value) {
      res = await fetch(`${API_URL}/metodos-pago/${pagoEditingId.value}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });
    } else {
      res = await fetch(`${API_URL}/metodos-pago`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });
    }
    if (res.ok) {
      addAlert(`Método de pago guardado.`, 'success');
      pagoNombre.value = '';
      pagoCuenta.value = '';
      pagoEstado.value = true;
      pagoEditingId.value = null;
      loadMetodos();
    }
  } catch (e) { console.error(e); }
};

const deleteMetodo = async (id, name) => {
  if (confirm(`¿Eliminar el método de pago "${name}"?`)) {
    try {
      const res = await fetch(`${API_URL}/metodos-pago/${id}`, { method: 'DELETE' });
      if (res.ok || res.status === 204) {
        addAlert(`Método de pago "${name}" eliminado.`, 'success');
        loadMetodos();
      }
    } catch (e) { console.error(e); }
  }
};
</script>

<template>
  <div>
    <!-- Tabs Auxiliares -->
    <div style="display: flex; gap: 8px; margin-bottom: 24px; border-bottom: 1px solid var(--border-color); padding-bottom: 12px;">
      <button class="btn" :class="activeTab === 'presentaciones' ? 'btn-primary' : 'btn-secondary'" @click="activeTab = 'presentaciones'">
        Presentaciones
      </button>
      <button class="btn" :class="activeTab === 'casas' ? 'btn-primary' : 'btn-secondary'" @click="activeTab = 'casas'">
        Casas Médicas
      </button>
      <button class="btn" :class="activeTab === 'pagos' ? 'btn-primary' : 'btn-secondary'" @click="activeTab = 'pagos'">
        Métodos de Pago
      </button>
    </div>

    <!-- 1. SECCIÓN PRESENTACIONES -->
    <div v-if="activeTab === 'presentaciones'" style="display: grid; grid-template-columns: 350px 1fr; gap: 24px;">
      <!-- Formulario Izquierda -->
      <div class="stats-card" style="height: fit-content;">
        <h4 class="pos-section-title">{{ presEditingId ? 'Editar Presentación' : 'Nueva Presentación' }}</h4>
        <div class="form-group">
          <label class="form-label">Nombre de Presentación *</label>
          <input type="text" class="form-control" v-model="presNombre" placeholder="Ej. Tableta, Jarabe, Ampolla">
        </div>
        <div class="form-group">
          <label class="form-check">
            <input type="checkbox" class="form-check-input" v-model="presEstado">
            <span class="form-check-label">Habilitado</span>
          </label>
        </div>
        <div style="display: flex; gap: 8px; margin-top: 10px;">
          <button class="btn btn-secondary" v-if="presEditingId" @click="presEditingId = null; presNombre = ''; presEstado = true;">Cancelar</button>
          <button class="btn btn-primary" @click="savePresentacion">Guardar</button>
        </div>
      </div>
      <!-- Listado Derecha -->
      <div class="table-responsive">
        <table class="table">
          <thead>
            <tr>
              <th>ID</th>
              <th>Nombre Presentación</th>
              <th>Estado</th>
              <th style="text-align: right;">Acciones</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="item in presentacionesList" :key="item.id_presentacion">
              <td>#000{{ item.id_presentacion }}</td>
              <td style="font-weight: 600;">{{ item.nombre_presentacion }}</td>
              <td>
                <span class="status-badge" :class="item.estado_presentacion ? 'active' : 'inactive'">
                  {{ item.estado_presentacion ? 'Activo' : 'Inactivo' }}
                </span>
              </td>
              <td style="text-align: right;">
                <div style="display: inline-flex; gap: 8px;">
                  <button class="btn btn-secondary btn-sm" @click="presEditingId = item.id_presentacion; presNombre = item.nombre_presentacion; presEstado = item.estado_presentacion;">
                    Editar
                  </button>
                  <button class="btn btn-danger btn-sm" @click="deletePresentacion(item.id_presentacion, item.nombre_presentacion)">
                    Eliminar
                  </button>
                </div>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <!-- 2. SECCIÓN CASAS MEDICAS -->
    <div v-if="activeTab === 'casas'" style="display: grid; grid-template-columns: 350px 1fr; gap: 24px;">
      <!-- Formulario Izquierda -->
      <div class="stats-card" style="height: fit-content;">
        <h4 class="pos-section-title">{{ casaEditingId ? 'Editar Casa Médica' : 'Nueva Casa Médica' }}</h4>
        <div class="form-group">
          <label class="form-label">Nombre Casa Médica *</label>
          <input type="text" class="form-control" v-model="casaNombre" placeholder="Ej. Bayer, Pfizer, Novartis">
        </div>
        <div class="form-group">
          <label class="form-check">
            <input type="checkbox" class="form-check-input" v-model="casaEstado">
            <span class="form-check-label">Habilitado</span>
          </label>
        </div>
        <div style="display: flex; gap: 8px; margin-top: 10px;">
          <button class="btn btn-secondary" v-if="casaEditingId" @click="casaEditingId = null; casaNombre = ''; casaEstado = true;">Cancelar</button>
          <button class="btn btn-primary" @click="saveCasaMedica">Guardar</button>
        </div>
      </div>
      <!-- Listado Derecha -->
      <div class="table-responsive">
        <table class="table">
          <thead>
            <tr>
              <th>ID</th>
              <th>Nombre Casa Médica</th>
              <th>Estado</th>
              <th style="text-align: right;">Acciones</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="item in casasMedicasList" :key="item.id_casa_medica">
              <td>#000{{ item.id_casa_medica }}</td>
              <td style="font-weight: 600;">{{ item.nombre_casa_medica }}</td>
              <td>
                <span class="status-badge" :class="item.estado_casa_medica ? 'active' : 'inactive'">
                  {{ item.estado_casa_medica ? 'Activo' : 'Inactivo' }}
                </span>
              </td>
              <td style="text-align: right;">
                <div style="display: inline-flex; gap: 8px;">
                  <button class="btn btn-secondary btn-sm" @click="casaEditingId = item.id_casa_medica; casaNombre = item.nombre_casa_medica; casaEstado = item.estado_casa_medica;">
                    Editar
                  </button>
                  <button class="btn btn-danger btn-sm" @click="deleteCasaMedica(item.id_casa_medica, item.nombre_casa_medica)">
                    Eliminar
                  </button>
                </div>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <!-- 3. SECCIÓN METODOS DE PAGO -->
    <div v-if="activeTab === 'pagos'" style="display: grid; grid-template-columns: 350px 1fr; gap: 24px;">
      <!-- Formulario Izquierda -->
      <div class="stats-card" style="height: fit-content;">
        <h4 class="pos-section-title">{{ pagoEditingId ? 'Editar Método' : 'Nuevo Método' }}</h4>
        <div class="form-group">
          <label class="form-label">Nombre del Método *</label>
          <input type="text" class="form-control" v-model="pagoNombre" placeholder="Ej. Efectivo, Tarjeta Visa, Transferencia">
        </div>
        <div class="form-group">
          <label class="form-label">Cuenta Relacionada</label>
          <input type="text" class="form-control" v-model="pagoCuenta" placeholder="Ej. Cuenta Monetaria #1234">
        </div>
        <div class="form-group">
          <label class="form-check">
            <input type="checkbox" class="form-check-input" v-model="pagoEstado">
            <span class="form-check-label">Habilitado</span>
          </label>
        </div>
        <div style="display: flex; gap: 8px; margin-top: 10px;">
          <button class="btn btn-secondary" v-if="pagoEditingId" @click="pagoEditingId = null; pagoNombre = ''; pagoCuenta = ''; pagoEstado = true;">Cancelar</button>
          <button class="btn btn-primary" @click="saveMetodo">Guardar</button>
        </div>
      </div>
      <!-- Listado Derecha -->
      <div class="table-responsive">
        <table class="table">
          <thead>
            <tr>
              <th>ID</th>
              <th>Nombre Método</th>
              <th>Cuenta Relacionada</th>
              <th>Estado</th>
              <th style="text-align: right;">Acciones</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="item in metodosList" :key="item.id_metodo_pago">
              <td>#000{{ item.id_metodo_pago }}</td>
              <td style="font-weight: 600;">{{ item.nombre_metodo_pago }}</td>
              <td><code>{{ item.cuenta_metodo_pago || 'Ninguna' }}</code></td>
              <td>
                <span class="status-badge" :class="item.estado_metodo_pago ? 'active' : 'inactive'">
                  {{ item.estado_metodo_pago ? 'Activo' : 'Inactivo' }}
                </span>
              </td>
              <td style="text-align: right;">
                <div style="display: inline-flex; gap: 8px;">
                  <button class="btn btn-secondary btn-sm" @click="pagoEditingId = item.id_metodo_pago; pagoNombre = item.nombre_metodo_pago; pagoCuenta = item.cuenta_metodo_pago || ''; pagoEstado = item.estado_metodo_pago;">
                    Editar
                  </button>
                  <button class="btn btn-danger btn-sm" @click="deleteMetodo(item.id_metodo_pago, item.nombre_metodo_pago)">
                    Eliminar
                  </button>
                </div>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>
