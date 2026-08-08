<script setup>
import { ref, onMounted, inject, computed } from 'vue';

const API_URL = inject('API_URL');
const addAlert = inject('addAlert');

const proveedoresList = ref([]);
const medicamentosList = ref([]);

// Formulario de Compra
const selectedProveedorId = ref('');
const selectedMedId = ref('');
const purchaseItems = ref([]);

// Formulario de Lote temporal para el producto seleccionado
const activeMed = ref(null);
const batchExpiry = ref('');
const batchProduction = ref('');
const batchPrice = ref(0);
const batchQty = ref(1);

const loading = ref(false);

const loadData = async () => {
  try {
    const provRes = await fetch(`${API_URL}/proveedores?limite=100`);
    if (provRes.ok) {
      const d = await provRes.json();
      proveedoresList.value = d.datos || [];
      if (proveedoresList.value.length > 0) {
        selectedProveedorId.value = proveedoresList.value[0].id_proveedor;
      }
    }

    const medRes = await fetch(`${API_URL}/medicamentos?limite=100`);
    if (medRes.ok) {
      const d = await medRes.json();
      medicamentosList.value = d.datos || [];
    }
  } catch (error) {
    console.error('Error al cargar datos en compras:', error);
  }
};

onMounted(() => {
  loadData();
});

const activeMedicamentosList = computed(() => {
  return medicamentosList.value.filter(m => m.estado_medicamento);
});

const onMedSelect = () => {
  if (!selectedMedId.value) return;
  const med = medicamentosList.value.find(m => m.id_medicamento === Number(selectedMedId.value));
  if (med) {
    selectMedForPurchase(med);
  }
};

const selectMedForPurchase = (med) => {
  activeMed.value = med;
  batchPrice.value = Number(med.precio_venta) * 0.6; // Valor inicial sugerido del lote (60% del precio venta)
  batchQty.value = 10; // Cantidad sugerida inicial
  batchExpiry.value = '';
  batchProduction.value = '';
};

const addItemToPurchase = () => {
  if (!activeMed.value || !batchExpiry.value || batchQty.value <= 0 || batchPrice.value < 0) {
    alert('Por favor complete todos los datos del lote.');
    return;
  }

  // Verificar si ya está en la compra
  const existing = purchaseItems.value.find(item => item.id_medicamento === activeMed.value.id_medicamento);
  if (existing) {
    alert('Este medicamento ya está agregado a la lista de compra actual.');
    return;
  }

  purchaseItems.value.push({
    id_medicamento: activeMed.value.id_medicamento,
    nombre_medicamento: activeMed.value.nombre_medicamento,
    cantidad: Number(batchQty.value),
    precio_lote: Number(batchPrice.value),
    fecha_vencimiento: batchExpiry.value,
    fecha_produccion: batchProduction.value || null,
    subtotal: Number(batchPrice.value) // El costo del lote completo
  });

  // Reset de selección temporal
  activeMed.value = null;
  selectedMedId.value = '';
};

const removeItem = (medId) => {
  purchaseItems.value = purchaseItems.value.filter(item => item.id_medicamento !== medId);
};

// Sumar subtotales de la compra
const purchaseTotal = computed(() => {
  return purchaseItems.value.reduce((acc, curr) => acc + curr.subtotal, 0);
});

// Guardar Registro de Compra completo en BD
const savePurchase = async () => {
  if (purchaseItems.value.length === 0) {
    alert('No hay artículos agregados a la compra.');
    return;
  }
  if (!selectedProveedorId.value) {
    alert('Seleccione un proveedor.');
    return;
  }

  loading.value = true;
  try {
    // 1. Guardar cabecera de Compra
    const purchasePayload = {
      id_proveedor: Number(selectedProveedorId.value),
      fecha_compra: new Date().toISOString().split('T')[0],
      total_compra: Number(purchaseTotal.value),
      estado_compra: true
    };

    const compRes = await fetch(`${API_URL}/compras`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(purchasePayload)
    });

    if (!compRes.ok) {
      const err = await compRes.json();
      throw new Error(err.message || 'Error al guardar cabecera de compra');
    }

    const savedPurchase = await compRes.json();

    // 2. Procesar cada medicamento comprado (Crear Lote y Detalle Compra)
    for (const item of purchaseItems.value) {
      // 2.1. Crear el Lote en BD
      const lotePayload = {
        id_medicamento: item.id_medicamento,
        fecha_vencimiento: item.fecha_vencimiento,
        fecha_produccion: item.fecha_produccion || undefined,
        precio_lote: Number(item.precio_lote),
        existencia_lote: Number(item.cantidad),
        estado_lote: true
      };

      const loteRes = await fetch(`${API_URL}/lotes`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(lotePayload)
      });

      if (!loteRes.ok) {
        throw new Error(`Error al registrar lote para medicamento ID ${item.id_medicamento}`);
      }

      const savedLote = await loteRes.json();

      // 2.2. Crear el Detalle de la Compra
      const detailPayload = {
        id_compra: savedPurchase.id_compra,
        id_proveedor: Number(selectedProveedorId.value),
        id_medicamento: item.id_medicamento,
        id_lote: savedLote.id_lote,
        cantidad_compra: item.cantidad,
        subtotal_compra: item.subtotal,
        estado_compra: true
      };

      const detRes = await fetch(`${API_URL}/detalles-compra`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(detailPayload)
      });

      if (!detRes.ok) {
        throw new Error(`Error al registrar detalle de compra para medicamento ID ${item.id_medicamento}`);
      }

      // 2.3. Actualizar la existencia total del medicamento en el catálogo
      // Buscamos los datos actuales del medicamento
      const currentMedRes = await fetch(`${API_URL}/medicamentos/${item.id_medicamento}`);
      if (currentMedRes.ok) {
        const currentMed = await currentMedRes.json();
        const newStock = Number(currentMed.existencia_total_medicamento || 0) + Number(item.cantidad);
        
        await fetch(`${API_URL}/medicamentos/${item.id_medicamento}`, {
          method: 'PATCH',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ existencia_total_medicamento: newStock })
        });
      }
    }

    addAlert(`Compra #${savedPurchase.id_compra} guardada correctamente. Stock actualizado.`, 'success');
    purchaseItems.value = [];
    loadData();

  } catch (error) {
    alert(error.message || 'Error al guardar la compra');
    console.error(error);
  } finally {
    loading.value = false;
  }
};

const formatCurrency = (val) => {
  return new Intl.NumberFormat('es-GT', { style: 'currency', currency: 'GTQ' }).format(val);
};
</script>

<template>
  <div class="pos-container">
    <!-- Mitad Izquierda: Configuración de la Compra e Ingreso de Lotes -->
    <div class="pos-left">
      
      <!-- Panel de Proveedor -->
      <div class="stats-card">
        <h4 class="pos-section-title">Proveedor del Pedido</h4>
        <div class="form-group" style="margin-bottom: 0;">
          <label class="form-label">Seleccionar Proveedor</label>
          <select class="form-control" v-model="selectedProveedorId">
            <option v-for="prov in proveedoresList" :key="prov.id_proveedor" :value="prov.id_proveedor">
              {{ prov.nombre_proveedor }}
            </option>
          </select>
        </div>
      </div>

      <!-- Seleccionar Medicamento -->
      <div class="stats-card">
        <h4 class="pos-section-title">Seleccionar Medicamento a Adquirir</h4>
        <div class="form-group" style="margin-bottom: 0;">
          <label class="form-label">Seleccionar Medicamento *</label>
          <select class="form-control" v-model="selectedMedId" @change="onMedSelect">
            <option value="" disabled>-- Seleccione un medicamento del catálogo --</option>
            <option v-for="med in activeMedicamentosList" :key="med.id_medicamento" :value="med.id_medicamento">
              {{ med.nombre_medicamento }} (Stock: {{ med.existencia_total_medicamento }})
            </option>
          </select>
        </div>
      </div>

      <!-- Configuración del Lote para el Producto Seleccionado -->
      <div class="stats-card" v-if="activeMed" style="border: 1px solid var(--primary-color); background-color: rgba(59, 130, 246, 0.03);">
        <h4 class="pos-section-title" style="color: var(--primary-color);">Configurar Lote: {{ activeMed.nombre_medicamento }}</h4>
        
        <div class="form-row">
          <div class="form-group">
            <label class="form-label">Fecha de Producción</label>
            <input type="date" class="form-control" v-model="batchProduction">
          </div>
          <div class="form-group">
            <label class="form-label">Fecha de Vencimiento *</label>
            <input type="date" class="form-control" v-model="batchExpiry" required>
          </div>
        </div>

        <div class="form-row">
          <div class="form-group">
            <label class="form-label">Costo Total del Lote (Q) *</label>
            <input type="number" step="0.01" class="form-control" v-model="batchPrice" min="0" required>
          </div>
          <div class="form-group">
            <label class="form-label">Cantidad Adquirida (Unidades) *</label>
            <input type="number" class="form-control" v-model="batchQty" min="1" required>
          </div>
        </div>

        <div style="display: flex; gap: 8px; justify-content: flex-end; margin-top: 10px;">
          <button class="btn btn-secondary" @click="activeMed = null; selectedMedId = ''">Cancelar</button>
          <button class="btn btn-primary" @click="addItemToPurchase">Agregar a la Lista de Compra</button>
        </div>
      </div>

      <!-- Detalle de la Compra actual -->
      <div class="table-responsive">
        <div style="padding: 16px 20px; font-weight: 700; border-bottom: 1px solid var(--border-color); font-size: 15px;">
          Artículos en el Pedido de Compra
        </div>
        <table class="table">
          <thead>
            <tr>
              <th>Artículo</th>
              <th>Fecha Vencimiento</th>
              <th style="text-align: center;">Cantidad Comprada</th>
              <th style="text-align: right;">Costo Lote</th>
              <th style="text-align: right;">Subtotal</th>
              <th style="text-align: right;">Acciones</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="item in purchaseItems" :key="item.id_medicamento">
              <td style="font-weight: 600;">{{ item.nombre_medicamento }}</td>
              <td><code>{{ item.fecha_vencimiento }}</code></td>
              <td style="text-align: center;">{{ item.cantidad }}</td>
              <td style="text-align: right;">{{ formatCurrency(item.precio_lote) }}</td>
              <td style="text-align: right; font-weight: 700;">{{ formatCurrency(item.subtotal) }}</td>
              <td style="text-align: right;">
                <button class="btn btn-danger btn-sm" @click="removeItem(item.id_medicamento)">&times;</button>
              </td>
            </tr>
            <tr v-if="purchaseItems.length === 0">
              <td colspan="6" style="text-align: center; padding: 40px; color: var(--text-muted);">
                No hay medicamentos agregados al pedido de compra. Seleccione uno arriba.
              </td>
            </tr>
          </tbody>
        </table>
      </div>

    </div>

    <!-- Mitad Derecha: Resumen de Compra -->
    <div class="pos-right">
      <h4 class="pos-section-title">Resumen de Importación</h4>
      
      <div class="pos-summary">
        <div class="pos-summary-row">
          <span>Total Artículos</span>
          <span>{{ purchaseItems.length }}</span>
        </div>
        <div class="pos-summary-row total">
          <span>Costo Total</span>
          <span>{{ formatCurrency(purchaseTotal) }}</span>
        </div>
      </div>

      <button class="btn btn-success" style="width:100%; margin-top: 24px; font-size:16px; padding: 12px;"
              :disabled="purchaseItems.length === 0 || loading" 
              @click="savePurchase">
        {{ loading ? 'Registrando...' : 'Registrar Compra e Inventario' }}
      </button>
    </div>
  </div>
</template>
