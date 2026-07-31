<script setup>
import { ref, onMounted, inject } from 'vue';

const API_URL = inject('API_URL');

const props = defineProps({
  defaultMode: { type: String, default: 'detalle_venta' } // 'detalle_venta' | 'detalle_compra' | 'detalle_pagos'
});

const mode = ref(props.defaultMode);

// --- ESTADOS ---
const list = ref([]);
const loading = ref(true);
const totalCount = ref(0);

// Catálogos locales para traducción
const medicamentosList = ref([]);
const proveedoresList = ref([]);
const metodosPagoList = ref([]);

const loadCatalogs = async () => {
  try {
    const medRes = await fetch(`${API_URL}/medicamentos?limite=200`);
    if (medRes.ok) {
      const d = await medRes.json();
      medicamentosList.value = d.datos || [];
    }
    const provRes = await fetch(`${API_URL}/proveedores?limite=200`);
    if (provRes.ok) {
      const d = await provRes.json();
      proveedoresList.value = d.datos || [];
    }
    const metRes = await fetch(`${API_URL}/metodos-pago?limite=200`);
    if (metRes.ok) {
      const d = await metRes.json();
      metodosPagoList.value = d.datos || [];
    }
  } catch (e) {
    console.error('Error al cargar catálogos:', e);
  }
};

const loadDetallesVenta = async () => {
  loading.value = true;
  try {
    const res = await fetch(`${API_URL}/detalle-venta?limite=100`);
    if (res.ok) {
      const data = await res.json();
      list.value = data.datos || [];
      totalCount.value = data.total || 0;
    }
  } catch (e) {
    console.error(e);
  } finally {
    loading.value = false;
  }
};

const loadDetallesCompra = async () => {
  loading.value = true;
  try {
    const res = await fetch(`${API_URL}/detalle-compra?limite=100`);
    if (res.ok) {
      const data = await res.json();
      list.value = data.datos || [];
      totalCount.value = data.total || 0;
    }
  } catch (e) {
    console.error(e);
  } finally {
    loading.value = false;
  }
};

const loadDetallesPagos = async () => {
  loading.value = true;
  try {
    const res = await fetch(`${API_URL}/detalles-metodos-pago?limite=100`);
    if (res.ok) {
      const data = await res.json();
      list.value = data.datos || [];
      totalCount.value = data.total || 0;
    }
  } catch (e) {
    console.error(e);
  } finally {
    loading.value = false;
  }
};

const refreshData = async () => {
  if (mode.value === 'detalle_venta') {
    await loadDetallesVenta();
  } else if (mode.value === 'detalle_compra') {
    await loadDetallesCompra();
  } else {
    await loadDetallesPagos();
  }
};

onMounted(async () => {
  await loadCatalogs();
  await refreshData();
});

const getMedicamentoNombre = (id) => {
  const m = medicamentosList.value.find(item => item.id_medicamento === id);
  return m ? m.nombre_medicamento : `Medicamento #${id}`;
};

const getProveedorNombre = (id) => {
  const p = proveedoresList.value.find(item => item.id_proveedor === id);
  return p ? p.nombre_proveedor : `Proveedor #${id}`;
};

const getMetodoPagoNombre = (id) => {
  const m = metodosPagoList.value.find(item => item.id_metodo_pago === id);
  return m ? m.nombre_metodo_pago : `Método #${id}`;
};

const formatCurrency = (val) => {
  return new Intl.NumberFormat('es-GT', { style: 'currency', currency: 'GTQ' }).format(val);
};
</script>

<template>
  <div>
    <!-- Tabs Internas -->
    <div style="display: flex; gap: 8px; margin-bottom: 24px; border-bottom: 1px solid var(--border-color); padding-bottom: 12px;">
      <button class="btn" :class="mode === 'detalle_venta' ? 'btn-primary' : 'btn-secondary'" @click="mode = 'detalle_venta'; refreshData();">
        Líneas de Detalles de Ventas
      </button>
      <button class="btn" :class="mode === 'detalle_compra' ? 'btn-primary' : 'btn-secondary'" @click="mode = 'detalle_compra'; refreshData();">
        Líneas de Detalles de Compras
      </button>
      <button class="btn" :class="mode === 'detalle_pagos' ? 'btn-primary' : 'btn-secondary'" @click="mode = 'detalle_pagos'; refreshData();">
        Líneas de Detalles Métodos de Pago
      </button>
    </div>

    <!-- Spinner -->
    <div v-if="loading" style="text-align: center; padding: 40px; color: var(--text-secondary);">
      Consultando base de datos relacional...
    </div>

    <div v-else>
      
      <!-- 1. TABLA DETALLES DE VENTAS -->
      <div class="table-responsive" v-if="mode === 'detalle_venta'">
        <table class="table">
          <thead>
            <tr>
              <th>ID Línea</th>
              <th>Venta Referenciada</th>
              <th>Medicamento</th>
              <th>Lote Asignado</th>
              <th>Cantidad</th>
              <th>Subtotal</th>
              <th>Estado Registro</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="item in list" :key="item.id_detalle_venta">
              <td>#DV-{{ item.id_detalle_venta }}</td>
              <td><span style="font-weight: 700; color: var(--primary-color);">Venta #000{{ item.id_venta }}</span></td>
              <td style="font-weight: 600;">{{ getMedicamentoNombre(item.id_medicamento) }}</td>
              <td><code>{{ item.id_lote ? '#L-' + item.id_lote : 'Sin Lote' }}</code></td>
              <td>{{ item.cantidad_detalle_venta }} unidades</td>
              <td style="font-weight: 700;">{{ formatCurrency(item.subtotal_detalle_venta) }}</td>
              <td>
                <span class="status-badge" :class="item.estado_detalle_venta ? 'active' : 'inactive'">
                  {{ item.estado_detalle_venta ? 'Válido' : 'Cancelado' }}
                </span>
              </td>
            </tr>
            <tr v-if="list.length === 0">
              <td colspan="7" style="text-align: center; padding: 40px; color: var(--text-muted);">
                No hay líneas de detalles de ventas registradas.
              </td>
            </tr>
          </tbody>
        </table>
      </div>

      <!-- 2. TABLA DETALLES DE COMPRAS -->
      <div class="table-responsive" v-if="mode === 'detalle_compra'">
        <table class="table">
          <thead>
            <tr>
              <th>ID Línea</th>
              <th>Compra Referenciada</th>
              <th>Proveedor</th>
              <th>Medicamento</th>
              <th>Lote Generado</th>
              <th>Cantidad</th>
              <th>Subtotal Compra</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="item in list" :key="item.id_detalle_compra">
              <td>#DC-{{ item.id_detalle_compra }}</td>
              <td><span style="font-weight: 700; color: var(--success-color);">Compra #000{{ item.id_compra }}</span></td>
              <td style="font-weight: 600;">{{ getProveedorNombre(item.id_proveedor) }}</td>
              <td>{{ getMedicamentoNombre(item.id_medicamento) }}</td>
              <td><code>#L-{{ item.id_lote }}</code></td>
              <td>{{ item.cantidad_compra }} unidades</td>
              <td style="font-weight: 700;">{{ formatCurrency(item.subtotal_compra) }}</td>
            </tr>
            <tr v-if="list.length === 0">
              <td colspan="7" style="text-align: center; padding: 40px; color: var(--text-muted);">
                No hay líneas de detalles de compras registradas.
              </td>
            </tr>
          </tbody>
        </table>
      </div>

      <!-- 3. TABLA DETALLES DE METODOS DE PAGO -->
      <div class="table-responsive" v-if="mode === 'detalle_pagos'">
        <table class="table">
          <thead>
            <tr>
              <th>ID Línea</th>
              <th>Venta Asociada</th>
              <th>Método de Pago</th>
              <th>Monto Pagado</th>
              <th>Estado Registro</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="item in list" :key="item.id_detalle_metodos_pago">
              <td>#DMP-{{ item.id_detalle_metodos_pago }}</td>
              <td><span style="font-weight: 700; color: var(--primary-color);">Venta #000{{ item.id_venta }}</span></td>
              <td style="font-weight: 600; color: var(--accent-color);">{{ getMetodoPagoNombre(item.id_metodo_pago) }}</td>
              <td style="font-weight: 700;">{{ formatCurrency(item.cantidad_detalle_metodos_pago) }}</td>
              <td>
                <span class="status-badge" :class="item.estado_detalle_metodos_pago ? 'active' : 'inactive'">
                  {{ item.estado_detalle_metodos_pago ? 'Válido' : 'Anulado' }}
                </span>
              </td>
            </tr>
            <tr v-if="list.length === 0">
              <td colspan="5" style="text-align: center; padding: 40px; color: var(--text-muted);">
                No hay líneas de métodos de pago registradas.
              </td>
            </tr>
          </tbody>
        </table>
      </div>

    </div>
  </div>
</template>
