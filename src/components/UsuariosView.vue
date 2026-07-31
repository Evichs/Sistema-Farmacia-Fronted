<script setup>
import { ref, onMounted, inject } from 'vue';

const API_URL = inject('API_URL');
const addAlert = inject('addAlert');

const props = defineProps({
  activeTabProp: { type: String, default: 'usuarios' }
});

const activeTab = ref(props.activeTabProp); // 'usuarios' | 'roles'

// --- ESTADO USUARIOS ---
const usuariosList = ref([]);
const rolesList = ref([]);

const isUserModalOpen = ref(false);
const isUserEditing = ref(false);
const userEditingId = ref(null);

const formUsername = ref('');
const formPassword = ref('');
const showUserPassword = ref(false);
const formFullName = ref('');
const formPhone = ref('');
const formEmail = ref('');
const formDpi = ref('');
const formRoleId = ref('');
const formEstado = ref(true);

// --- ESTADO ROLES ---
const isRoleModalOpen = ref(false);
const isRoleEditing = ref(false);
const roleEditingId = ref(null);

const formRoleName = ref('');
const formRoleEstado = ref(true);

// --- CARGAR DATOS ---
const loadRoles = async () => {
  try {
    const res = await fetch(`${API_URL}/roles?limite=100`);
    if (res.ok) {
      const data = await res.json();
      rolesList.value = data.datos || [];
    }
  } catch (e) { console.error(e); }
};

const loadUsuarios = async () => {
  try {
    const res = await fetch(`${API_URL}/usuarios?limite=100`);
    if (res.ok) {
      const data = await res.json();
      usuariosList.value = data.datos || [];
    }
  } catch (e) { console.error(e); }
};

const getRoleName = (id) => {
  const r = rolesList.value.find(item => item.id_rol === id);
  return r ? r.nombre_rol : 'Desconocido';
};

onMounted(() => {
  loadRoles();
  loadUsuarios();
});

// --- ACCIONES USUARIOS ---
const openUserCreateModal = () => {
  isUserEditing.value = false;
  userEditingId.value = null;
  formUsername.value = '';
  formPassword.value = '';
  formFullName.value = '';
  formPhone.value = '';
  formEmail.value = '';
  formDpi.value = '';
  formRoleId.value = rolesList.value.length > 0 ? rolesList.value[0].id_rol : '';
  formEstado.value = true;
  isUserModalOpen.value = true;
};

const openUserEditModal = (u) => {
  isUserEditing.value = true;
  userEditingId.value = u.id_usuario;
  formUsername.value = u.usuario;
  formPassword.value = ''; // No cargar contraseña existente
  formFullName.value = u.nombre_usuario;
  formPhone.value = u.telefono_usuario || '';
  formEmail.value = u.correo_usuario || '';
  formDpi.value = u.dpi_usuario || '';
  formRoleId.value = u.id_rol;
  formEstado.value = u.estado_usuario;
  isUserModalOpen.value = true;
};

const saveUsuario = async () => {
  if (!formUsername.value || !formFullName.value || !formRoleId.value || (!isUserEditing.value && !formPassword.value)) {
    alert('Por favor complete los campos requeridos.');
    return;
  }

  const payload = {
    usuario: formUsername.value,
    nombre_usuario: formFullName.value,
    telefono_usuario: formPhone.value || undefined,
    correo_usuario: formEmail.value || undefined,
    dpi_usuario: formDpi.value || undefined,
    id_rol: Number(formRoleId.value),
    estado_usuario: formEstado.value
  };

  // Solo enviar password si se escribe algo (o al crear)
  if (formPassword.value) {
    payload.password = formPassword.value;
  }

  try {
    let res;
    if (isUserEditing.value) {
      res = await fetch(`${API_URL}/usuarios/${userEditingId.value}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });
    } else {
      res = await fetch(`${API_URL}/usuarios`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });
    }

    if (res.ok) {
      addAlert(`Usuario "${formUsername.value}" guardado correctamente.`, 'success');
      isUserModalOpen.value = false;
      loadUsuarios();
    } else {
      const err = await res.json();
      alert(err.message || 'Error al guardar usuario');
    }
  } catch (error) {
    console.error(error);
  }
};

const deleteUsuario = async (id, name) => {
  if (confirm(`¿Eliminar al usuario "${name}"?`)) {
    try {
      const res = await fetch(`${API_URL}/usuarios/${id}`, { method: 'DELETE' });
      if (res.ok || res.status === 204) {
        addAlert(`Usuario "${name}" eliminado.`, 'success');
        loadUsuarios();
      }
    } catch (e) { console.error(e); }
  }
};

// --- ACCIONES ROLES ---
const openRoleCreateModal = () => {
  isRoleEditing.value = false;
  roleEditingId.value = null;
  formRoleName.value = '';
  formRoleEstado.value = true;
  isRoleModalOpen.value = true;
};

const openRoleEditModal = (r) => {
  isRoleEditing.value = true;
  roleEditingId.value = r.id_rol;
  formRoleName.value = r.nombre_rol;
  formRoleEstado.value = r.estado_rol;
  isRoleModalOpen.value = true;
};

const saveRol = async () => {
  if (!formRoleName.value) return;
  const payload = { nombre_rol: formRoleName.value, estado_rol: formRoleEstado.value };
  try {
    let res;
    if (isRoleEditing.value) {
      res = await fetch(`${API_URL}/roles/${roleEditingId.value}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });
    } else {
      res = await fetch(`${API_URL}/roles`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });
    }
    if (res.ok) {
      addAlert(`Rol guardado correctamente.`, 'success');
      isRoleModalOpen.value = false;
      loadRoles();
    } else {
      const err = await res.json();
      alert(err.message || 'Error al guardar rol');
    }
  } catch (e) { console.error(e); }
};

const deleteRol = async (id, name) => {
  if (confirm(`¿Eliminar el rol "${name}"?`)) {
    try {
      const res = await fetch(`${API_URL}/roles/${id}`, { method: 'DELETE' });
      if (res.ok || res.status === 204) {
        addAlert(`Rol "${name}" eliminado.`, 'success');
        loadRoles();
      }
    } catch (e) { console.error(e); }
  }
};
</script>

<template>
  <div>
    <!-- Tabs -->
    <div style="display: flex; gap: 8px; margin-bottom: 24px; border-bottom: 1px solid var(--border-color); padding-bottom: 12px;">
      <button class="btn" :class="activeTab === 'usuarios' ? 'btn-primary' : 'btn-secondary'" @click="activeTab = 'usuarios'">
        Usuarios del Sistema
      </button>
      <button class="btn" :class="activeTab === 'roles' ? 'btn-primary' : 'btn-secondary'" @click="activeTab = 'roles'">
        Roles y Accesos
      </button>
    </div>

    <!-- 1. SECCIÓN USUARIOS -->
    <div v-if="activeTab === 'usuarios'">
      <div class="filter-card" style="justify-content: flex-end;">
        <button class="btn btn-primary" @click="openUserCreateModal">
          <svg style="width:16px;height:16px;fill:currentColor" viewBox="0 0 24 24">
            <path d="M19 13h-6v6h-2v-6H5v-2h6V5h2v6h6v2z"/>
          </svg>
          Nuevo Usuario
        </button>
      </div>

      <div class="table-responsive">
        <table class="table">
          <thead>
            <tr>
              <th>Usuario</th>
              <th>Nombre Completo</th>
              <th>DPI</th>
              <th>Correo</th>
              <th>Teléfono</th>
              <th>Rol</th>
              <th>Estado</th>
              <th style="text-align: right;">Acciones</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="user in usuariosList" :key="user.id_usuario">
              <td><code>{{ user.usuario }}</code></td>
              <td style="font-weight: 600;">{{ user.nombre_usuario }}</td>
              <td>{{ user.dpi_usuario || 'N/A' }}</td>
              <td>{{ user.correo_usuario || 'N/A' }}</td>
              <td>{{ user.telefono_usuario || 'N/A' }}</td>
              <td><span style="font-weight:600;color:var(--primary-color);">{{ getRoleName(user.id_rol) }}</span></td>
              <td>
                <span class="status-badge" :class="user.estado_usuario ? 'active' : 'inactive'">
                  {{ user.estado_usuario ? 'Activo' : 'Suspendido' }}
                </span>
              </td>
              <td style="text-align: right;">
                <div style="display: inline-flex; gap: 8px;">
                  <button class="btn btn-secondary btn-sm" @click="openUserEditModal(user)">
                    Editar
                  </button>
                  <button class="btn btn-danger btn-sm" @click="deleteUsuario(user.id_usuario, user.usuario)">
                    Eliminar
                  </button>
                </div>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <!-- 2. SECCIÓN ROLES -->
    <div v-if="activeTab === 'roles'">
      <div class="filter-card" style="justify-content: flex-end;">
        <button class="btn btn-primary" @click="openRoleCreateModal">
          <svg style="width:16px;height:16px;fill:currentColor" viewBox="0 0 24 24">
            <path d="M19 13h-6v6h-2v-6H5v-2h6V5h2v6h6v2z"/>
          </svg>
          Nuevo Rol
        </button>
      </div>

      <div class="table-responsive">
        <table class="table">
          <thead>
            <tr>
              <th>ID Rol</th>
              <th>Nombre Rol</th>
              <th>Estado</th>
              <th style="text-align: right;">Acciones</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="role in rolesList" :key="role.id_rol">
              <td>#00{{ role.id_rol }}</td>
              <td style="font-weight: 600;">{{ role.nombre_rol }}</td>
              <td>
                <span class="status-badge" :class="role.estado_rol ? 'active' : 'inactive'">
                  {{ role.estado_rol ? 'Habilitado' : 'Inactivo' }}
                </span>
              </td>
              <td style="text-align: right;">
                <div style="display: inline-flex; gap: 8px;">
                  <button class="btn btn-secondary btn-sm" @click="openRoleEditModal(role)">
                    Editar
                  </button>
                  <button class="btn btn-danger btn-sm" @click="deleteRol(role.id_rol, role.nombre_rol)">
                    Eliminar
                  </button>
                </div>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <!-- Modal Formulario Usuario -->
    <div class="modal-overlay" v-if="isUserModalOpen">
      <div class="modal-content">
        <div class="modal-header">
          <h3 class="modal-title">{{ isUserEditing ? 'Editar Usuario' : 'Nuevo Usuario' }}</h3>
          <button class="close-btn" @click="isUserModalOpen = false">&times;</button>
        </div>
        <div class="modal-body">
          <div class="form-group">
            <label class="form-label">Nombre Completo *</label>
            <input type="text" class="form-control" v-model="formFullName" placeholder="Ej. Carlos Mendoza" required>
          </div>

          <div class="form-row">
            <div class="form-group">
              <label class="form-label">Nombre de Usuario (Nickname) *</label>
              <input type="text" class="form-control" v-model="formUsername" placeholder="Ej. cmendoza" required>
            </div>
            <div class="form-group">
              <label class="form-label">Contraseña {{ isUserEditing ? '(Dejar vacío para no cambiar)' : '*' }}</label>
              <div class="password-input-wrapper">
                <input :type="showUserPassword ? 'text' : 'password'" class="form-control" v-model="formPassword" placeholder="******" autocomplete="new-password">
                <button type="button" class="password-toggle-btn" @click="showUserPassword = !showUserPassword" title="Mostrar/Ocultar contraseña">
                  <svg v-if="showUserPassword" viewBox="0 0 24 24"><path d="M12 7c2.76 0 5 2.24 5 5 0 .65-.13 1.26-.36 1.82l2.92 2.92c1.51-1.39 2.7-3.18 3.44-5.24-1.73-4.39-6-7.5-11-7.5-1.4 0-2.74.25-3.98.7l2.16 2.16C10.74 7.13 11.35 7 12 7zM2 4.27l2.28 2.28.46.46C3.08 8.3 1.78 10.02 1 12c1.73 4.39 6 7.5 11 7.5 1.55 0 3.03-.3 4.38-.84l.42.42L19.73 22 21 20.73 3.27 3 2 4.27zM7.53 9.8l1.55 1.55c-.05.21-.08.43-.08.65 0 1.66 1.34 3 3 3 .22 0 .44-.03.65-.08l1.55 1.55c-.67.33-1.41.53-2.2.53-2.76 0-5-2.24-5-5 0-.79.2-1.53.53-2.2zm4.31-.78l3.15 3.15.02-.16c0-1.66-1.34-3-3-3l-.17.01z"/></svg>
                  <svg v-else viewBox="0 0 24 24"><path d="M12 4.5C7 4.5 2.73 7.61 1 12c1.73 4.39 6 7.5 11 7.5s9.27-3.11 11-7.5c-1.73-4.39-6-7.5-11-7.5zM12 17c-2.76 0-5-2.24-5-5s2.24-5 5-5 5 2.24 5 5-2.24 5-5 5zm0-8c-1.66 0-3 1.34-3 3s1.34 3 3 3 3-1.34 3-3-1.34-3-3-3z"/></svg>
                </button>
              </div>
            </div>
          </div>

          <div class="form-row">
            <div class="form-group">
              <label class="form-label">Correo Electrónico</label>
              <input type="email" class="form-control" v-model="formEmail" placeholder="carlos@correo.com">
            </div>
            <div class="form-group">
              <label class="form-label">Teléfono</label>
              <input type="text" class="form-control" v-model="formPhone" placeholder="5555-5555">
            </div>
          </div>

          <div class="form-row">
            <div class="form-group">
              <label class="form-label">DPI / Cédula Identificación</label>
              <input type="text" class="form-control" v-model="formDpi" placeholder="1234 56789 0101">
            </div>
            <div class="form-group">
              <label class="form-label">Rol Asignado *</label>
              <select class="form-control" v-model="formRoleId" required>
                <option v-for="role in rolesList" :key="role.id_rol" :value="role.id_rol">
                  {{ role.nombre_rol }}
                </option>
              </select>
            </div>
          </div>

          <div class="form-group" style="margin-top: 15px;">
            <label class="form-check">
              <input type="checkbox" class="form-check-input" v-model="formEstado">
              <span class="form-check-label">Usuario Activo / Permitir Acceso</span>
            </label>
          </div>
        </div>
        <div class="modal-footer">
          <button class="btn btn-secondary" @click="isUserModalOpen = false">Cancelar</button>
          <button class="btn btn-primary" @click="saveUsuario">Guardar</button>
        </div>
      </div>
    </div>

    <!-- Modal Formulario Rol -->
    <div class="modal-overlay" v-if="isRoleModalOpen">
      <div class="modal-content">
        <div class="modal-header">
          <h3 class="modal-title">{{ isRoleEditing ? 'Editar Rol' : 'Nuevo Rol' }}</h3>
          <button class="close-btn" @click="isRoleModalOpen = false">&times;</button>
        </div>
        <div class="modal-body">
          <div class="form-group">
            <label class="form-label">Nombre del Rol *</label>
            <input type="text" class="form-control" v-model="formRoleName" placeholder="Ej. Administrador, Cajero, Bodeguero" required>
          </div>
          <div class="form-group" style="margin-top: 15px;">
            <label class="form-check">
              <input type="checkbox" class="form-check-input" v-model="formRoleEstado">
              <span class="form-check-label">Rol Habilitado</span>
            </label>
          </div>
        </div>
        <div class="modal-footer">
          <button class="btn btn-secondary" @click="isRoleModalOpen = false">Cancelar</button>
          <button class="btn btn-primary" @click="saveRol">Guardar</button>
        </div>
      </div>
    </div>
  </div>
</template>
