<script setup>
import { ref, onMounted, inject } from 'vue';

const API_URL = inject('API_URL');
const addAlert = inject('addAlert');

const medicamentosList = ref([]);
const presentacionesList = ref([]);
const totalCount = ref(0);
const currentPage = ref(1);
const totalPages = ref(1);
const limitPerPage = ref(10);
const searchPattern = ref('');

const isModalOpen = ref(false);
const isEditing = ref(false);
const editingId = ref(null);

// Campos del formulario
const formNombre = ref('');
const formCodigo = ref('');
const formPresentacionId = ref('');
const formExistencia = ref(0);
const formPrecioVenta = ref(0);
const formPrecioMayorista = ref(0);
const formPrecioMinimo = ref(0);
const formComponenteActivo = ref('');
const formVentaLibre = ref(true);
const formEstado = ref(true);

const loadPresentations = async () => {
  try {
    const res = await fetch(`${API_URL}/presentaciones?limite=100`);
    if (res.ok) {
      const data = await res.json();
      presentacionesList.value = data.datos || [];
    }
  } catch (error) {
    console.error('Error al cargar presentaciones:', error);
  }
};

const loadMedicamentos = async () => {
  try {
    // Si hay un término de búsqueda, en este backend podemos filtrarlo a nivel local o por query params si lo soporta.
    // Como las APIs CRUD por TypeORM suelen soportar paginación, haremos fetch de la página actual.
    const res = await fetch(`${API_URL}/medicamentos?pagina=${currentPage.value}&limite=${limitPerPage.value}`);
    if (res.ok) {
      const data = await res.json();
      medicamentosList.value = data.datos || [];
      totalCount.value = data.total || 0;
      totalPages.value = data.paginas || 1;
    }
  } catch (error) {
    console.error('Error al cargar medicamentos:', error);
  }
};

const getPresentacionNombre = (id) => {
  const p = presentacionesList.value.find(item => item.id_presentacion === id);
  return p ? p.nombre_presentacion : 'Desconocida';
};

onMounted(() => {
  loadPresentations();
  loadMedicamentos();
});

const handleSearch = async () => {
  currentPage.value = 1;
  if (!searchPattern.value) {
    loadMedicamentos();
    return;
  }
  // Filtrado local para búsqueda instantánea avanzada en el frontend
  try {
    const res = await fetch(`${API_URL}/medicamentos?limite=100`);
    if (res.ok) {
      const data = await res.json();
      const allMeds = data.datos || [];
      const filtered = allMeds.filter(m => 
        m.nombre_medicamento.toLowerCase().includes(searchPattern.value.toLowerCase()) ||
        (m.codigo_barras_medicamento && m.codigo_barras_medicamento.toLowerCase().includes(searchPattern.value.toLowerCase())) ||
        (m.componente_activo && m.componente_activo.toLowerCase().includes(searchPattern.value.toLowerCase()))
      );
      medicamentosList.value = filtered.slice(0, limitPerPage.value);
      totalCount.value = filtered.length;
      totalPages.value = Math.ceil(filtered.length / limitPerPage.value) || 1;
    }
  } catch (error) {
    console.error('Error en búsqueda local:', error);
  }
};

const openCreateModal = () => {
  isEditing.value = false;
  editingId.value = null;
  formNombre.value = '';
  formCodigo.value = '';
  formPresentacionId.value = presentacionesList.value.length > 0 ? presentacionesList.value[0].id_presentacion : '';
  formExistencia.value = 0;
  formPrecioVenta.value = 0;
  formPrecioMayorista.value = 0;
  formPrecioMinimo.value = 0;
  formComponenteActivo.value = '';
  formVentaLibre.value = true;
  formEstado.value = true;
  isModalOpen.value = true;
};

const openEditModal = (med) => {
  isEditing.value = true;
  editingId.value = med.id_medicamento;
  formNombre.value = med.nombre_medicamento;
  formCodigo.value = med.codigo_barras_medicamento || '';
  formPresentacionId.value = med.id_presentacion;
  formExistencia.value = med.existencia_total_medicamento;
  formPrecioVenta.value = Number(med.precio_venta);
  formPrecioMayorista.value = Number(med.precio_mayorista || 0);
  formPrecioMinimo.value = Number(med.precio_minimo || 0);
  formComponenteActivo.value = med.componente_activo || '';
  formVentaLibre.value = med.venta_libre;
  formEstado.value = med.estado_medicamento;
  isModalOpen.value = true;
};

const saveMedicamento = async () => {
  if (!formNombre.value || !formPresentacionId.value) {
    alert('Por favor complete los campos requeridos.');
    return;
  }

  const payload = {
    id_presentacion: Number(formPresentacionId.value),
    codigo_barras_medicamento: formCodigo.value || undefined,
    nombre_medicamento: formNombre.value,
    existencia_total_medicamento: Number(formExistencia.value),
    precio_venta: Number(formPrecioVenta.value),
    precio_mayorista: Number(formPrecioMayorista.value),
    precio_minimo: Number(formPrecioMinimo.value),
    componente_activo: formComponenteActivo.value || undefined,
    venta_libre: formVentaLibre.value,
    estado_medicamento: formEstado.value
  };

  try {
    let res;
    if (isEditing.value) {
      res = await fetch(`${API_URL}/medicamentos/${editingId.value}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });
    } else {
      res = await fetch(`${API_URL}/medicamentos`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });
    }

    if (res.ok) {
      addAlert(`Medicamento "${formNombre.value}" ${isEditing.value ? 'actualizado' : 'creado'} correctamente.`, 'success');
      isModalOpen.value = false;
      loadMedicamentos();
    } else {
      const err = await res.json();
      alert(err.message || 'Error al guardar el medicamento');
    }
  } catch (error) {
    console.error('Error al guardar medicamento:', error);
  }
};

const deleteMedicamento = async (id, name) => {
  if (confirm(`¿Está seguro de eliminar el medicamento "${name}"?`)) {
    try {
      const res = await fetch(`${API_URL}/medicamentos/${id}`, {
        method: 'DELETE'
      });
      if (res.ok || res.status === 204) {
        addAlert(`Medicamento "${name}" eliminado de la base de datos.`, 'success');
        loadMedicamentos();
      } else {
        const err = await res.json();
        alert(err.message || 'Error al eliminar');
      }
    } catch (error) {
      console.error('Error al eliminar medicamento:', error);
    }
  }
};

const changePage = (page) => {
  if (page >= 1 && page <= totalPages.value) {
    currentPage.value = page;
    loadMedicamentos();
  }
};

const formatCurrency = (val) => {
  return new Intl.NumberFormat('es-GT', { style: 'currency', currency: 'GTQ' }).format(val);
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
        <input type="text" class="form-control" v-model="searchPattern" @input="handleSearch" placeholder="Buscar por nombre, código o componente activo...">
      </div>
      <button class="btn btn-primary" @click="openCreateModal">
        <svg style="width:16px;height:16px;fill:currentColor" viewBox="0 0 24 24">
          <path d="M19 13h-6v6h-2v-6H5v-2h6V5h2v6h6v2z"/>
        </svg>
        Nuevo Medicamento
      </button>
    </div>

    <!-- Tabla de Medicamentos -->
    <div class="table-responsive">
      <table class="table">
        <thead>
          <tr>
            <th>Código / Barras</th>
            <th>Nombre Medicamento</th>
            <th>Presentación</th>
            <th>Existencia</th>
            <th>Precio Venta</th>
            <th>Componente Activo</th>
            <th>Venta Libre</th>
            <th>Estado</th>
            <th style="text-align: right;">Acciones</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="med in medicamentosList" :key="med.id_medicamento">
            <td>
              <code style="font-size:12px;">{{ med.codigo_barras_medicamento || 'N/A' }}</code>
            </td>
            <td style="font-weight: 600;">{{ med.nombre_medicamento }}</td>
            <td>{{ getPresentacionNombre(med.id_presentacion) }}</td>
            <td>
              <span :style="{ color: med.existencia_total_medicamento < 10 ? 'var(--danger-color)' : 'inherit', fontWeight: med.existencia_total_medicamento < 10 ? 'bold' : 'normal' }">
                {{ med.existencia_total_medicamento }}
              </span>
            </td>
            <td style="font-weight: 700;">{{ formatCurrency(med.precio_venta) }}</td>
            <td>{{ med.componente_activo || 'Genérico' }}</td>
            <td>
              <span class="status-badge" :class="med.venta_libre ? 'active' : 'inactive'" style="font-size:10px;">
                {{ med.venta_libre ? 'Sí' : 'Bajo Receta' }}
              </span>
            </td>
            <td>
              <span class="status-badge" :class="med.estado_medicamento ? 'active' : 'inactive'">
                {{ med.estado_medicamento ? 'Activo' : 'Inactivo' }}
              </span>
            </td>
            <td style="text-align: right;">
              <div style="display: inline-flex; gap: 8px;">
                <button class="btn btn-secondary btn-sm" @click="openEditModal(med)" title="Editar">
                  <svg style="width:14px;height:14px;fill:currentColor" viewBox="0 0 24 24">
                    <path d="M3 17.25V21h3.75L17.81 9.94l-3.75-3.75L3 17.25zM20.71 7.04c.39-.39.39-1.02 0-1.41l-2.34-2.34c-.39-.39-1.02-.39-1.41 0l-1.83 1.83 3.75 3.75 1.83-1.83z"/>
                  </svg>
                </button>
                <button class="btn btn-danger btn-sm" @click="deleteMedicamento(med.id_medicamento, med.nombre_medicamento)" title="Eliminar">
                  <svg style="width:14px;height:14px;fill:currentColor" viewBox="0 0 24 24">
                    <path d="M6 19c0 1.1.9 2 2 2h8c1.1 0 2-.9 2-2V7H6v12zM19 4h-3.5l-1-1h-5l-1 1H5v2h14V4z"/>
                  </svg>
                </button>
              </div>
            </td>
          </tr>
          <tr v-if="medicamentosList.length === 0">
            <td colspan="9" style="text-align: center; padding: 40px; color: var(--text-muted);">
              No se encontraron medicamentos en esta página.
            </td>
          </tr>
        </tbody>
      </table>
    </div>

    <!-- Paginación -->
    <div class="pagination">
      <div class="pagination-info">
        Mostrando página {{ currentPage }} de {{ totalPages }} (Total: {{ totalCount }} medicamentos)
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
          <h3 class="modal-title">{{ isEditing ? 'Editar Medicamento' : 'Nuevo Medicamento' }}</h3>
          <button class="close-btn" @click="isModalOpen = false">&times;</button>
        </div>
        <div class="modal-body">
          <div class="form-group">
            <label class="form-label">Nombre del Medicamento *</label>
            <input type="text" class="form-control" v-model="formNombre" placeholder="Ej. Ibuprofeno 400mg" required>
          </div>

          <div class="form-row">
            <div class="form-group">
              <label class="form-label">Código de Barras</label>
              <input type="text" class="form-control" v-model="formCodigo" placeholder="Ej. 740123456789">
            </div>
            <div class="form-group">
              <label class="form-label">Presentación *</label>
              <select class="form-control" v-model="formPresentacionId" required>
                <option v-for="pres in presentacionesList" :key="pres.id_presentacion" :value="pres.id_presentacion">
                  {{ pres.nombre_presentacion }}
                </option>
                <option v-if="presentacionesList.length === 0" value="">Cargue una presentación en Catálogos</option>
              </select>
            </div>
          </div>

          <div class="form-row">
            <div class="form-group">
              <label class="form-label">Existencia Inicial *</label>
              <input type="number" class="form-control" v-model="formExistencia" min="0" required>
            </div>
            <div class="form-group">
              <label class="form-label">Precio Venta (Público) *</label>
              <input type="number" step="0.01" class="form-control" v-model="formPrecioVenta" min="0" required>
            </div>
          </div>

          <div class="form-row">
            <div class="form-group">
              <label class="form-label">Precio Mayorista</label>
              <input type="number" step="0.01" class="form-control" v-model="formPrecioMayorista" min="0">
            </div>
            <div class="form-group">
              <label class="form-label">Precio Mínimo</label>
              <input type="number" step="0.01" class="form-control" v-model="formPrecioMinimo" min="0">
            </div>
          </div>

          <div class="form-group">
            <label class="form-label">Componente Activo</label>
            <input type="text" class="form-control" v-model="formComponenteActivo" placeholder="Ej. Ibuprofeno">
          </div>

          <div class="form-row" style="margin-top: 10px;">
            <div class="form-group">
              <label class="form-check">
                <input type="checkbox" class="form-check-input" v-model="formVentaLibre">
                <span class="form-check-label">Medicamento de Venta Libre</span>
              </label>
            </div>
            <div class="form-group">
              <label class="form-check">
                <input type="checkbox" class="form-check-input" v-model="formEstado">
                <span class="form-check-label">Estado Activo</span>
              </label>
            </div>
          </div>
        </div>
        <div class="modal-footer">
          <button class="btn btn-secondary" @click="isModalOpen = false">Cancelar</button>
          <button class="btn btn-primary" @click="saveMedicamento">Guardar</button>
        </div>
      </div>
    </div>
  </div>
</template>
