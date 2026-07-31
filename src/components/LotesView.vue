<script setup>
import { ref, onMounted, inject } from 'vue';

const API_URL = inject('API_URL');
const addAlert = inject('addAlert');

const lotesList = ref([]);
const medicamentosList = ref([]);
const totalCount = ref(0);
const currentPage = ref(1);
const totalPages = ref(1);
const limitPerPage = ref(10);
const searchPattern = ref('');

const isModalOpen = ref(false);
const isEditing = ref(false);
const editingId = ref(null);

// Campos del formulario
const formMedicamentoId = ref('');
const formFechaVencimiento = ref('');
const formFechaProduccion = ref('');
const formPrecioLote = ref(0);
const formExistenciaLote = ref(0);
const formEstado = ref(true);

const loadMedicamentos = async () => {
  try {
    const res = await fetch(`${API_URL}/medicamentos?limite=100`);
    if (res.ok) {
      const data = await res.json();
      medicamentosList.value = data.datos || [];
    }
  } catch (error) {
    console.error('Error al cargar medicamentos:', error);
  }
};

const loadLotes = async () => {
  try {
    const res = await fetch(`${API_URL}/lotes?pagina=${currentPage.value}&limite=${limitPerPage.value}`);
    if (res.ok) {
      const data = await res.json();
      lotesList.value = data.datos || [];
      totalCount.value = data.total || 0;
      totalPages.value = data.paginas || 1;
    }
  } catch (error) {
    console.error('Error al cargar lotes:', error);
  }
};

const getMedicamentoNombre = (id) => {
  const med = medicamentosList.value.find(item => item.id_medicamento === id);
  return med ? med.nombre_medicamento : 'Desconocido';
};

onMounted(() => {
  loadMedicamentos();
  loadLotes();
});

const handleSearch = async () => {
  currentPage.value = 1;
  if (!searchPattern.value) {
    loadLotes();
    return;
  }
  // Búsqueda local cruzando lotes con nombres de medicamentos
  try {
    const res = await fetch(`${API_URL}/lotes?limite=200`);
    if (res.ok) {
      const data = await res.json();
      const allLotes = data.datos || [];
      const filtered = allLotes.filter(l => {
        const medName = getMedicamentoNombre(l.id_medicamento).toLowerCase();
        return medName.includes(searchPattern.value.toLowerCase()) ||
               l.fecha_vencimiento.includes(searchPattern.value);
      });
      lotesList.value = filtered.slice(0, limitPerPage.value);
      totalCount.value = filtered.length;
      totalPages.value = Math.ceil(filtered.length / limitPerPage.value) || 1;
    }
  } catch (error) {
    console.error('Error en búsqueda de lotes:', error);
  }
};

const openCreateModal = () => {
  isEditing.value = false;
  editingId.value = null;
  formMedicamentoId.value = medicamentosList.value.length > 0 ? medicamentosList.value[0].id_medicamento : '';
  formFechaVencimiento.value = '';
  formFechaProduccion.value = '';
  formPrecioLote.value = 0;
  formExistenciaLote.value = 0;
  formEstado.value = true;
  isModalOpen.value = true;
};

const openEditModal = (l) => {
  isEditing.value = true;
  editingId.value = l.id_lote;
  formMedicamentoId.value = l.id_medicamento;
  formFechaVencimiento.value = l.fecha_vencimiento;
  formFechaProduccion.value = l.fecha_produccion || '';
  formPrecioLote.value = Number(l.precio_lote);
  formExistenciaLote.value = l.existencia_lote;
  formEstado.value = l.estado_lote;
  isModalOpen.value = true;
};

const saveLote = async () => {
  if (!formMedicamentoId.value || !formFechaVencimiento.value) {
    alert('Por favor complete los campos obligatorios.');
    return;
  }

  // Validación de fechas
  if (formFechaProduccion.value && formFechaProduccion.value > formFechaVencimiento.value) {
    alert('La fecha de producción no puede ser posterior a la fecha de vencimiento.');
    return;
  }

  const payload = {
    id_medicamento: Number(formMedicamentoId.value),
    fecha_vencimiento: formFechaVencimiento.value,
    fecha_produccion: formFechaProduccion.value || undefined,
    precio_lote: Number(formPrecioLote.value),
    existencia_lote: Number(formExistenciaLote.value),
    estado_lote: formEstado.value
  };

  try {
    let res;
    if (isEditing.value) {
      res = await fetch(`${API_URL}/lotes/${editingId.value}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });
    } else {
      res = await fetch(`${API_URL}/lotes`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });
    }

    if (res.ok) {
      addAlert(`Lote guardado correctamente.`, 'success');
      isModalOpen.value = false;
      loadLotes();
    } else {
      const err = await res.json();
      alert(err.message || 'Error al guardar lote');
    }
  } catch (error) {
    console.error('Error al guardar lote:', error);
  }
};

const deleteLote = async (id) => {
  if (confirm(`¿Está seguro de eliminar el lote #${id}?`)) {
    try {
      const res = await fetch(`${API_URL}/lotes/${id}`, {
        method: 'DELETE'
      });
      if (res.ok || res.status === 204) {
        addAlert(`Lote #${id} eliminado correctamente.`, 'success');
        loadLotes();
      } else {
        const err = await res.json();
        alert(err.message || 'Error al eliminar');
      }
    } catch (error) {
      console.error('Error al eliminar lote:', error);
    }
  }
};

const changePage = (page) => {
  if (page >= 1 && page <= totalPages.value) {
    currentPage.value = page;
    loadLotes();
  }
};

const formatCurrency = (val) => {
  return new Intl.NumberFormat('es-GT', { style: 'currency', currency: 'GTQ' }).format(val);
};

// Validar si el lote está vencido o por vencer (menos de 3 meses)
const getLoteVencimientoClass = (dateStr) => {
  if (!dateStr) return '';
  const expDate = new Date(dateStr);
  const now = new Date();
  const diffTime = expDate - now;
  const diffDays = Math.ceil(diffTime / (1000 * 60 * 60 * 24));
  
  if (diffDays < 0) return 'status-badge inactive'; // Vencido
  if (diffDays < 90) return 'status-badge'; // Por vencer (Naranja)
  return 'status-badge active'; // Correcto
};

const getLoteVencimientoLabel = (dateStr) => {
  if (!dateStr) return 'N/A';
  const expDate = new Date(dateStr);
  const now = new Date();
  const diffTime = expDate - now;
  const diffDays = Math.ceil(diffTime / (1000 * 60 * 60 * 24));
  
  if (diffDays < 0) return 'VENCIDO';
  if (diffDays < 90) return 'Por Vencer';
  return 'Vigente';
};
</script>

<template>
  <div>
    <!-- Buscador y filtro -->
    <div class="filter-card">
      <div class="filter-left">
        <svg class="search-icon" style="position: static; width:20px; height:20px; margin-right: 8px;" viewBox="0 0 24 24">
          <path d="M15.5 14h-.79l-.28-.27C15.41 12.59 16 11.11 16 9.5 16 5.91 13.09 3 9.5 3S3 5.91 3 9.5 5.91 16 9.5 16c1.61 0 3.09-.59 4.23-1.57l.27.28v.79l5 4.99L20.49 19l-4.99-5zm-6 0C7.01 14 5 11.99 5 9.5S7.01 5 9.5 5 14 7.01 14 9.5 11.99 14 9.5 14z"/>
        </svg>
        <input type="text" class="form-control" v-model="searchPattern" @input="handleSearch" placeholder="Buscar por medicamento o fecha...">
      </div>
      <button class="btn btn-primary" @click="openCreateModal">
        <svg style="width:16px;height:16px;fill:currentColor" viewBox="0 0 24 24">
          <path d="M19 13h-6v6h-2v-6H5v-2h6V5h2v6h6v2z"/>
        </svg>
        Nuevo Lote
      </button>
    </div>

    <!-- Tabla de Lotes -->
    <div class="table-responsive">
      <table class="table">
        <thead>
          <tr>
            <th>ID Lote</th>
            <th>Medicamento</th>
            <th>Fecha Producción</th>
            <th>Fecha Vencimiento</th>
            <th>Costo Lote</th>
            <th>Existencia Lote</th>
            <th>Estado Lote</th>
            <th>Vencimiento</th>
            <th style="text-align: right;">Acciones</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="lote in lotesList" :key="lote.id_lote">
            <td>#L-{{ lote.id_lote }}</td>
            <td style="font-weight: 600;">{{ getMedicamentoNombre(lote.id_medicamento) }}</td>
            <td>{{ lote.fecha_produccion || 'N/A' }}</td>
            <td><code>{{ lote.fecha_vencimiento }}</code></td>
            <td style="font-weight: 700;">{{ formatCurrency(lote.precio_lote) }}</td>
            <td>{{ lote.existencia_lote }}</td>
            <td>
              <span class="status-badge" :class="lote.estado_lote ? 'active' : 'inactive'">
                {{ lote.estado_lote ? 'Activo' : 'Inactivo' }}
              </span>
            </td>
            <td>
              <span :class="getLoteVencimientoClass(lote.fecha_vencimiento)" :style="getLoteVencimientoLabel(lote.fecha_vencimiento) === 'Por Vencer' ? { backgroundColor: 'rgba(245, 158, 11, 0.15)', color: 'var(--warning-color)' } : {}">
                {{ getLoteVencimientoLabel(lote.fecha_vencimiento) }}
              </span>
            </td>
            <td style="text-align: right;">
              <div style="display: inline-flex; gap: 8px;">
                <button class="btn btn-secondary btn-sm" @click="openEditModal(lote)" title="Editar">
                  <svg style="width:14px;height:14px;fill:currentColor" viewBox="0 0 24 24">
                    <path d="M3 17.25V21h3.75L17.81 9.94l-3.75-3.75L3 17.25zM20.71 7.04c.39-.39.39-1.02 0-1.41l-2.34-2.34c-.39-.39-1.02-.39-1.41 0l-1.83 1.83 3.75 3.75 1.83-1.83z"/>
                  </svg>
                </button>
                <button class="btn btn-danger btn-sm" @click="deleteLote(lote.id_lote)" title="Eliminar">
                  <svg style="width:14px;height:14px;fill:currentColor" viewBox="0 0 24 24">
                    <path d="M6 19c0 1.1.9 2 2 2h8c1.1 0 2-.9 2-2V7H6v12zM19 4h-3.5l-1-1h-5l-1 1H5v2h14V4z"/>
                  </svg>
                </button>
              </div>
            </td>
          </tr>
          <tr v-if="lotesList.length === 0">
            <td colspan="9" style="text-align: center; padding: 40px; color: var(--text-muted);">
              No se encontraron lotes registrados.
            </td>
          </tr>
        </tbody>
      </table>
    </div>

    <!-- Paginación -->
    <div class="pagination">
      <div class="pagination-info">
        Mostrando página {{ currentPage }} de {{ totalPages }} (Total: {{ totalCount }} lotes)
      </div>
      <div class="pagination-controls">
        <button class="btn btn-secondary btn-sm" :disabled="currentPage === 1" @click="changePage(currentPage - 1)">Anterior</button>
        <button class="btn btn-secondary btn-sm" :disabled="currentPage === totalPages" @click="changePage(currentPage + 1)">Siguiente</button>
      </div>
    </div>

    <!-- Modal Formulario -->
    <div class="modal-overlay" v-if="isModalOpen">
      <div class="modal-content">
        <div class="modal-header">
          <h3 class="modal-title">{{ isEditing ? 'Editar Lote' : 'Nuevo Lote' }}</h3>
          <button class="close-btn" @click="isModalOpen = false">&times;</button>
        </div>
        <div class="modal-body">
          <div class="form-group">
            <label class="form-label">Medicamento Relacionado *</label>
            <select class="form-control" v-model="formMedicamentoId" required>
              <option v-for="med in medicamentosList" :key="med.id_medicamento" :value="med.id_medicamento">
                {{ med.nombre_medicamento }} (Stock actual: {{ med.existencia_total_medicamento }})
              </option>
              <option v-if="medicamentosList.length === 0" value="">Cargue medicamentos en el inventario</option>
            </select>
          </div>

          <div class="form-row">
            <div class="form-group">
              <label class="form-label">Fecha de Producción</label>
              <input type="date" class="form-control" v-model="formFechaProduccion">
            </div>
            <div class="form-group">
              <label class="form-label">Fecha de Vencimiento *</label>
              <input type="date" class="form-control" v-model="formFechaVencimiento" required>
            </div>
          </div>

          <div class="form-row">
            <div class="form-group">
              <label class="form-label">Costo Total del Lote *</label>
              <input type="number" step="0.01" class="form-control" v-model="formPrecioLote" min="0" required>
            </div>
            <div class="form-group">
              <label class="form-label">Existencia de este Lote *</label>
              <input type="number" class="form-control" v-model="formExistenciaLote" min="0" required>
            </div>
          </div>

          <div class="form-group" style="margin-top: 15px;">
            <label class="form-check">
              <input type="checkbox" class="form-check-input" v-model="formEstado">
              <span class="form-check-label">Lote Habilitado / Activo</span>
            </label>
          </div>
        </div>
        <div class="modal-footer">
          <button class="btn btn-secondary" @click="isModalOpen = false">Cancelar</button>
          <button class="btn btn-primary" @click="saveLote">Guardar</button>
        </div>
      </div>
    </div>
  </div>
</template>
