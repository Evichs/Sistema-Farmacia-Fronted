<script setup>
import { ref, onMounted, inject } from 'vue';

const API_URL = inject('API_URL');
const addAlert = inject('addAlert');

const clientesList = ref([]);
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
const formNit = ref('CF');
const formEstado = ref(true);

const loadClientes = async () => {
  try {
    const res = await fetch(`${API_URL}/clientes?pagina=${currentPage.value}&limite=${limitPerPage.value}`);
    if (res.ok) {
      const data = await res.json();
      clientesList.value = data.datos || [];
      totalCount.value = data.total || 0;
      totalPages.value = data.paginas || 1;
    }
  } catch (error) {
    console.error('Error al cargar clientes:', error);
  }
};

onMounted(() => {
  loadClientes();
});

const handleSearch = async () => {
  currentPage.value = 1;
  if (!searchPattern.value) {
    loadClientes();
    return;
  }
  // Búsqueda local avanzada
  try {
    const res = await fetch(`${API_URL}/clientes?limite=200`);
    if (res.ok) {
      const data = await res.json();
      const allClients = data.datos || [];
      const filtered = allClients.filter(c => 
        c.nombre_cliente.toLowerCase().includes(searchPattern.value.toLowerCase()) ||
        c.nit_cliente.toLowerCase().includes(searchPattern.value.toLowerCase())
      );
      clientesList.value = filtered.slice(0, limitPerPage.value);
      totalCount.value = filtered.length;
      totalPages.value = Math.ceil(filtered.length / limitPerPage.value) || 1;
    }
  } catch (error) {
    console.error('Error en búsqueda de clientes:', error);
  }
};

const openCreateModal = () => {
  isEditing.value = false;
  editingId.value = null;
  formNombre.value = '';
  formNit.value = 'CF';
  formEstado.value = true;
  isModalOpen.value = true;
};

const openEditModal = (client) => {
  isEditing.value = true;
  editingId.value = client.id_cliente;
  formNombre.value = client.nombre_cliente;
  formNit.value = client.nit_cliente;
  formEstado.value = client.estado_cliente;
  isModalOpen.value = true;
};

const saveCliente = async () => {
  if (!formNombre.value) {
    alert('Por favor complete los campos obligatorios.');
    return;
  }

  const payload = {
    nombre_cliente: formNombre.value,
    nit_cliente: formNit.value || 'CF',
    estado_cliente: formEstado.value
  };

  try {
    let res;
    if (isEditing.value) {
      res = await fetch(`${API_URL}/clientes/${editingId.value}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });
    } else {
      res = await fetch(`${API_URL}/clientes`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });
    }

    if (res.ok) {
      addAlert(`Cliente "${formNombre.value}" ${isEditing.value ? 'actualizado' : 'creado'} correctamente.`, 'success');
      isModalOpen.value = false;
      loadClientes();
    } else {
      const err = await res.json();
      alert(err.message || 'Error al guardar cliente');
    }
  } catch (error) {
    console.error('Error al guardar cliente:', error);
  }
};

const deleteCliente = async (id, name) => {
  if (confirm(`¿Está seguro de eliminar al cliente "${name}"?`)) {
    try {
      const res = await fetch(`${API_URL}/clientes/${id}`, {
        method: 'DELETE'
      });
      if (res.ok || res.status === 204) {
        addAlert(`Cliente "${name}" eliminado de la base de datos.`, 'success');
        loadClientes();
      } else {
        // Capturar errores de clave foránea (status 500) por ventas existentes
        if (res.status === 500) {
          alert(`No se puede eliminar al cliente "${name}" porque tiene ventas o facturas registradas a su nombre en el sistema. \n\nPara mantener el historial e integridad fiscal, le sugerimos desactivarlo cambiando su estado a "Inactivo" en el botón Editar.`);
        } else {
          const err = await res.json();
          alert(err.message || 'Error al eliminar');
        }
      }
    } catch (error) {
      console.error('Error al eliminar cliente:', error);
      alert('Ocurrió un error al intentar conectarse con el servidor para eliminar el cliente.');
    }
  }
};

const changePage = (page) => {
  if (page >= 1 && page <= totalPages.value) {
    currentPage.value = page;
    loadClientes();
  }
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
        <input type="text" class="form-control" v-model="searchPattern" @input="handleSearch" placeholder="Buscar por nombre o NIT...">
      </div>
      <button class="btn btn-primary" @click="openCreateModal">
        <svg style="width:16px;height:16px;fill:currentColor" viewBox="0 0 24 24">
          <path d="M19 13h-6v6h-2v-6H5v-2h6V5h2v6h6v2z"/>
        </svg>
        Nuevo Cliente
      </button>
    </div>

    <!-- Tabla de Clientes -->
    <div class="table-responsive">
      <table class="table">
        <thead>
          <tr>
            <th>ID Cliente</th>
            <th>Nombre Completo</th>
            <th>NIT</th>
            <th>Estado</th>
            <th style="text-align: right;">Acciones</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="client in clientesList" :key="client.id_cliente">
            <td>#000{{ client.id_cliente }}</td>
            <td style="font-weight: 600;">{{ client.nombre_cliente }}</td>
            <td><code>{{ client.nit_cliente }}</code></td>
            <td>
              <span class="status-badge" :class="client.estado_cliente ? 'active' : 'inactive'">
                {{ client.estado_cliente ? 'Activo' : 'Inactivo' }}
              </span>
            </td>
            <td style="text-align: right;">
              <div style="display: inline-flex; gap: 8px;">
                <button class="btn btn-secondary btn-sm" @click="openEditModal(client)" title="Editar">
                  <svg style="width:14px;height:14px;fill:currentColor" viewBox="0 0 24 24">
                    <path d="M3 17.25V21h3.75L17.81 9.94l-3.75-3.75L3 17.25zM20.71 7.04c.39-.39.39-1.02 0-1.41l-2.34-2.34c-.39-.39-1.02-.39-1.41 0l-1.83 1.83 3.75 3.75 1.83-1.83z"/>
                  </svg>
                </button>
                <button class="btn btn-danger btn-sm" @click="deleteCliente(client.id_cliente, client.nombre_cliente)" title="Eliminar">
                  <svg style="width:14px;height:14px;fill:currentColor" viewBox="0 0 24 24">
                    <path d="M6 19c0 1.1.9 2 2 2h8c1.1 0 2-.9 2-2V7H6v12zM19 4h-3.5l-1-1h-5l-1 1H5v2h14V4z"/>
                  </svg>
                </button>
              </div>
            </td>
          </tr>
          <tr v-if="clientesList.length === 0">
            <td colspan="5" style="text-align: center; padding: 40px; color: var(--text-muted);">
              No se encontraron clientes.
            </td>
          </tr>
        </tbody>
      </table>
    </div>

    <!-- Paginación -->
    <div class="pagination">
      <div class="pagination-info">
        Mostrando página {{ currentPage }} de {{ totalPages }} (Total: {{ totalCount }} clientes)
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
          <h3 class="modal-title">{{ isEditing ? 'Editar Cliente' : 'Nuevo Cliente' }}</h3>
          <button class="close-btn" @click="isModalOpen = false">&times;</button>
        </div>
        <div class="modal-body">
          <div class="form-group">
            <label class="form-label">Nombre Completo del Cliente *</label>
            <input type="text" class="form-control" v-model="formNombre" placeholder="Ej. Juan Pérez" required>
          </div>

          <div class="form-group">
            <label class="form-label">NIT / Cédula Fiscal</label>
            <input type="text" class="form-control" v-model="formNit" placeholder="CF o Número de NIT">
          </div>

          <div class="form-group" style="margin-top: 15px;">
            <label class="form-check">
              <input type="checkbox" class="form-check-input" v-model="formEstado">
              <span class="form-check-label">Cliente Activo</span>
            </label>
          </div>
        </div>
        <div class="modal-footer">
          <button class="btn btn-secondary" @click="isModalOpen = false">Cancelar</button>
          <button class="btn btn-primary" @click="saveCliente">Guardar</button>
        </div>
      </div>
    </div>
  </div>
</template>
