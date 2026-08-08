<script setup>
import { ref, onMounted, inject } from 'vue';

const API_URL = inject('API_URL');
const addAlert = inject('addAlert');

const proveedoresList = ref([]);
const casasMedicasList = ref([]);
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
const formCasaMedicaId = ref('');
const formTelefono = ref('');
const formDireccion = ref('');
const formCorreo = ref('');
const formTotalAdquirido = ref(0);
const formCantidadAdquirido = ref(0);
const formEstado = ref(true);

const loadCasasMedicas = async () => {
  try {
    const res = await fetch(`${API_URL}/casas-medicas?limite=100`);
    if (res.ok) {
      const data = await res.json();
      casasMedicasList.value = data.datos || [];
    }
  } catch (error) {
    console.error('Error al cargar casas médicas:', error);
  }
};

const loadProveedores = async () => {
  try {
    const res = await fetch(`${API_URL}/proveedores?pagina=${currentPage.value}&limite=${limitPerPage.value}`);
    if (res.ok) {
      const data = await res.json();
      proveedoresList.value = data.datos || [];
      totalCount.value = data.total || 0;
      totalPages.value = data.paginas || 1;
    }
  } catch (error) {
    console.error('Error al cargar proveedores:', error);
  }
};

const getCasaMedicaNombre = (id) => {
  const c = casasMedicasList.value.find(item => item.id_casa_medica === id);
  return c ? c.nombre_casa_medica : 'Desconocida';
};

onMounted(() => {
  loadCasasMedicas();
  loadProveedores();
});

const handleSearch = async () => {
  currentPage.value = 1;
  if (!searchPattern.value) {
    loadProveedores();
    return;
  }
  // Búsqueda local avanzada
  try {
    const res = await fetch(`${API_URL}/proveedores?limite=200`);
    if (res.ok) {
      const data = await res.json();
      const allProvs = data.datos || [];
      const filtered = allProvs.filter(p => 
        p.nombre_proveedor.toLowerCase().includes(searchPattern.value.toLowerCase()) ||
        (p.correo_proveedor && p.correo_proveedor.toLowerCase().includes(searchPattern.value.toLowerCase())) ||
        (p.telefono_proveedor && p.telefono_proveedor.includes(searchPattern.value))
      );
      proveedoresList.value = filtered.slice(0, limitPerPage.value);
      totalCount.value = filtered.length;
      totalPages.value = Math.ceil(filtered.length / limitPerPage.value) || 1;
    }
  } catch (error) {
    console.error('Error en búsqueda de proveedores:', error);
  }
};

const openCreateModal = () => {
  isEditing.value = false;
  editingId.value = null;
  formNombre.value = '';
  formCasaMedicaId.value = casasMedicasList.value.length > 0 ? casasMedicasList.value[0].id_casa_medica : '';
  formTelefono.value = '';
  formDireccion.value = '';
  formCorreo.value = '';
  formTotalAdquirido.value = 0;
  formCantidadAdquirido.value = 0;
  formEstado.value = true;
  isModalOpen.value = true;
};

const openEditModal = (p) => {
  isEditing.value = true;
  editingId.value = p.id_proveedor;
  formNombre.value = p.nombre_proveedor;
  formCasaMedicaId.value = p.id_casa_medica;
  formTelefono.value = p.telefono_proveedor || '';
  formDireccion.value = p.direccion_proveedor || '';
  formCorreo.value = p.correo_proveedor || '';
  formTotalAdquirido.value = Number(p.total_adquirido_proveedor || 0);
  formCantidadAdquirido.value = p.cantidad_adquirido_proveedor || 0;
  formEstado.value = p.estado_proveedor;
  isModalOpen.value = true;
};

const saveProveedor = async () => {
  if (!formNombre.value || !formCasaMedicaId.value) {
    alert('Por favor complete los campos obligatorios.');
    return;
  }

  const payload = {
    id_casa_medica: Number(formCasaMedicaId.value),
    nombre_proveedor: formNombre.value,
    estado_proveedor: formEstado.value,
    telefono_proveedor: formTelefono.value || undefined,
    direccion_proveedor: formDireccion.value || undefined,
    correo_proveedor: formCorreo.value || undefined,
    total_adquirido_proveedor: Number(formTotalAdquirido.value),
    cantidad_adquirido_proveedor: Number(formCantidadAdquirido.value)
  };

  try {
    let res;
    if (isEditing.value) {
      res = await fetch(`${API_URL}/proveedores/${editingId.value}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });
    } else {
      res = await fetch(`${API_URL}/proveedores`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });
    }

    if (res.ok) {
      addAlert(`Proveedor "${formNombre.value}" ${isEditing.value ? 'actualizado' : 'creado'} correctamente.`, 'success');
      isModalOpen.value = false;
      loadProveedores();
    } else {
      const err = await res.json();
      alert(err.message || 'Error al guardar proveedor');
    }
  } catch (error) {
    console.error('Error al guardar proveedor:', error);
  }
};

const deleteProveedor = async (id, name) => {
  if (confirm(`¿Está seguro de eliminar al proveedor "${name}"?`)) {
    try {
      const res = await fetch(`${API_URL}/proveedores/${id}`, {
        method: 'DELETE'
      });
      if (res.ok || res.status === 204) {
        addAlert(`Proveedor "${name}" eliminado de la base de datos.`, 'success');
        loadProveedores();
      } else {
        const err = await res.json();
        alert(err.message || 'Error al eliminar');
      }
    } catch (error) {
      console.error('Error al eliminar proveedor:', error);
    }
  }
};

const changePage = (page) => {
  if (page >= 1 && page <= totalPages.value) {
    currentPage.value = page;
    loadProveedores();
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
        <input type="text" class="form-control" v-model="searchPattern" @input="handleSearch" placeholder="Buscar por nombre, teléfono o correo...">
      </div>
      <button class="btn btn-primary" @click="openCreateModal">
        <svg style="width:16px;height:16px;fill:currentColor" viewBox="0 0 24 24">
          <path d="M19 13h-6v6h-2v-6H5v-2h6V5h2v6h6v2z"/>
        </svg>
        Nuevo Proveedor
      </button>
    </div>

    <!-- Tabla de Proveedores -->
    <div class="table-responsive">
      <table class="table">
        <thead>
          <tr>
            <th>Proveedor</th>
            <th>Casa Médica</th>
            <th>Teléfono</th>
            <th>Correo Electrónico</th>
            <th>Dirección</th>
            <th>Total Suministrado</th>
            <th>Cant. Pedidos</th>
            <th>Estado</th>
            <th style="text-align: right;">Acciones</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="prov in proveedoresList" :key="prov.id_proveedor">
            <td style="font-weight: 600;">{{ prov.nombre_proveedor }}</td>
            <td>{{ getCasaMedicaNombre(prov.id_casa_medica) }}</td>
            <td><code>{{ prov.telefono_proveedor || 'N/A' }}</code></td>
            <td>{{ prov.correo_proveedor || 'N/A' }}</td>
            <td style="max-width: 200px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap;">
              {{ prov.direccion_proveedor || 'N/A' }}
            </td>
            <td style="font-weight: 700;">{{ formatCurrency(prov.total_adquirido_proveedor) }}</td>
            <td>{{ prov.cantidad_adquirido_proveedor }}</td>
            <td>
              <span class="status-badge" :class="prov.estado_proveedor ? 'active' : 'inactive'">
                {{ prov.estado_proveedor ? 'Activo' : 'Inactivo' }}
              </span>
            </td>
            <td style="text-align: right;">
              <div style="display: inline-flex; gap: 8px;">
                <button class="btn btn-secondary btn-sm" @click="openEditModal(prov)" title="Editar">
                  <svg style="width:14px;height:14px;fill:currentColor" viewBox="0 0 24 24">
                    <path d="M3 17.25V21h3.75L17.81 9.94l-3.75-3.75L3 17.25zM20.71 7.04c.39-.39.39-1.02 0-1.41l-2.34-2.34c-.39-.39-1.02-.39-1.41 0l-1.83 1.83 3.75 3.75 1.83-1.83z"/>
                  </svg>
                </button>
                <button class="btn btn-danger btn-sm" @click="deleteProveedor(prov.id_proveedor, prov.nombre_proveedor)" title="Eliminar">
                  <svg style="width:14px;height:14px;fill:currentColor" viewBox="0 0 24 24">
                    <path d="M6 19c0 1.1.9 2 2 2h8c1.1 0 2-.9 2-2V7H6v12zM19 4h-3.5l-1-1h-5l-1 1H5v2h14V4z"/>
                  </svg>
                </button>
              </div>
            </td>
          </tr>
          <tr v-if="proveedoresList.length === 0">
            <td colspan="9" style="text-align: center; padding: 40px; color: var(--text-muted);">
              No se encontraron proveedores.
            </td>
          </tr>
        </tbody>
      </table>
    </div>

    <!-- Paginación -->
    <div class="pagination">
      <div class="pagination-info">
        Mostrando página {{ currentPage }} de {{ totalPages }} (Total: {{ totalCount }} proveedores)
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
          <h3 class="modal-title">{{ isEditing ? 'Editar Proveedor' : 'Nuevo Proveedor' }}</h3>
          <button class="close-btn" @click="isModalOpen = false">&times;</button>
        </div>
        <div class="modal-body">
          <div class="form-group">
            <label class="form-label">Nombre del Proveedor *</label>
            <input type="text" class="form-control" v-model="formNombre" placeholder="Ej. Droguería Nacional" required>
          </div>

          <div class="form-row">
            <div class="form-group">
              <label class="form-label">Casa Médica Fabricante *</label>
              <select class="form-control" v-model="formCasaMedicaId" required>
                <option v-for="casa in casasMedicasList" :key="casa.id_casa_medica" :value="casa.id_casa_medica">
                  {{ casa.nombre_casa_medica }}
                </option>
                <option v-if="casasMedicasList.length === 0" value="">Cargue una casa médica en Catálogos</option>
              </select>
            </div>
            <div class="form-group">
              <label class="form-label">Teléfono de Contacto</label>
              <input type="text" class="form-control" v-model="formTelefono" placeholder="Ej. 2200-5500">
            </div>
          </div>

          <div class="form-row">
            <div class="form-group">
              <label class="form-label">Correo Electrónico</label>
              <input type="email" class="form-control" v-model="formCorreo" placeholder="proveedor@correo.com">
            </div>
            <div class="form-group">
              <label class="form-label">Dirección Fiscal</label>
              <input type="text" class="form-control" v-model="formDireccion" placeholder="Ciudad de Guatemala">
            </div>
          </div>

          <div class="form-row">
            <div class="form-group">
              <label class="form-label">Total Adquirido (Histórico)</label>
              <input type="number" step="0.01" class="form-control" v-model="formTotalAdquirido" min="0">
            </div>
            <div class="form-group">
              <label class="form-label">Cantidad Adquirida (Pedidos)</label>
              <input type="number" class="form-control" v-model="formCantidadAdquirido" min="0">
            </div>
          </div>

          <div class="form-group" style="margin-top: 15px;">
            <label class="form-check">
              <input type="checkbox" class="form-check-input" v-model="formEstado">
              <span class="form-check-label">Proveedor Activo</span>
            </label>
          </div>
        </div>
        <div class="modal-footer">
          <button class="btn btn-secondary" @click="isModalOpen = false">Cancelar</button>
          <button class="btn btn-primary" @click="saveProveedor">Guardar</button>
        </div>
      </div>
    </div>
  </div>
</template>
