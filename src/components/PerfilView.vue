<script setup>
import { ref, onMounted, inject } from 'vue';

const API_URL = inject('API_URL');
const addAlert = inject('addAlert');

const props = defineProps({
  currentUser: {
    type: Object,
    required: true
  }
});

const emit = defineEmits(['update-user']);

const savingProfile = ref(false);
const savingPassword = ref(false);
const rolesList = ref([]);
const roleName = ref('Cargando...');

// Form profile states
const formFullName = ref(props.currentUser.nombre_usuario || '');
const formEmail = ref(props.currentUser.correo_usuario || '');
const formPhone = ref(props.currentUser.telefono_usuario || '');
const formDpi = ref(props.currentUser.dpi_usuario || '');

// Form password states
const formNewPassword = ref('');
const formConfirmPassword = ref('');
const showNewPassword = ref(false);
const showConfirmPassword = ref(false);

// Load Roles to display Role name
const loadRoles = async () => {
  try {
    const res = await fetch(`${API_URL}/roles?limite=100`);
    if (res.ok) {
      const data = await res.json();
      rolesList.value = data.datos || [];
      const userRole = rolesList.value.find(r => r.id_rol === props.currentUser.id_rol);
      roleName.value = userRole ? userRole.nombre_rol : 'Usuario';
    }
  } catch (err) {
    console.error('Error al cargar roles en perfil:', err);
    roleName.value = 'Usuario';
  }
};

onMounted(() => {
  loadRoles();
});

const handleSaveProfile = async () => {
  if (!formFullName.value) {
    addAlert('El nombre completo es requerido.', 'danger');
    return;
  }

  savingProfile.value = true;
  const payload = {
    usuario: props.currentUser.usuario,
    nombre_usuario: formFullName.value,
    correo_usuario: formEmail.value || undefined,
    telefono_usuario: formPhone.value || undefined,
    dpi_usuario: formDpi.value || undefined,
    id_rol: props.currentUser.id_rol,
    estado_usuario: props.currentUser.estado_usuario
  };

  try {
    const res = await fetch(`${API_URL}/usuarios/${props.currentUser.id_usuario}`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload)
    });

    if (res.ok) {
      const updatedUser = {
        ...props.currentUser,
        nombre_usuario: formFullName.value,
        correo_usuario: formEmail.value,
        telefono_usuario: formPhone.value,
        dpi_usuario: formDpi.value
      };
      emit('update-user', updatedUser);
      addAlert('Perfil actualizado correctamente.', 'success');
    } else {
      const err = await res.json();
      addAlert(err.message || 'Error al actualizar el perfil.', 'danger');
    }
  } catch (err) {
    console.error(err);
    addAlert('Error de red al actualizar el perfil.', 'danger');
  } finally {
    savingProfile.value = false;
  }
};

const handleUpdatePassword = async () => {
  if (!formNewPassword.value) {
    addAlert('Por favor ingrese la nueva contraseña.', 'danger');
    return;
  }

  if (formNewPassword.value !== formConfirmPassword.value) {
    addAlert('Las contraseñas no coinciden.', 'danger');
    return;
  }

  savingPassword.value = true;
  const payload = {
    usuario: props.currentUser.usuario,
    nombre_usuario: formFullName.value,
    correo_usuario: formEmail.value || undefined,
    telefono_usuario: formPhone.value || undefined,
    dpi_usuario: formDpi.value || undefined,
    id_rol: props.currentUser.id_rol,
    estado_usuario: props.currentUser.estado_usuario,
    password: formNewPassword.value
  };

  try {
    const res = await fetch(`${API_URL}/usuarios/${props.currentUser.id_usuario}`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload)
    });

    if (res.ok) {
      addAlert('Contraseña actualizada correctamente.', 'success');
      formNewPassword.value = '';
      formConfirmPassword.value = '';
    } else {
      const err = await res.json();
      addAlert(err.message || 'Error al actualizar la contraseña.', 'danger');
    }
  } catch (err) {
    console.error(err);
    addAlert('Error de red al cambiar la contraseña.', 'danger');
  } finally {
    savingPassword.value = false;
  }
};
</script>

<template>
  <div class="profile-container">
    <div class="profile-grid">
      <!-- Left Column: Avatar Card -->
      <div class="profile-card avatar-section">
        <div class="avatar-wrapper">
          <img class="profile-avatar" src="https://images.unsplash.com/photo-1535713875002-d1d0cf377fde?w=200&auto=format&fit=crop&q=80" alt="Avatar" />
          <div class="avatar-badge" :title="props.currentUser.estado_usuario ? 'Activo' : 'Inactivo'">
            <span class="status-dot" :class="{ active: props.currentUser.estado_usuario }"></span>
            {{ props.currentUser.estado_usuario ? 'Activo' : 'Inactivo' }}
          </div>
        </div>
        <h2 class="profile-display-name">{{ formFullName || props.currentUser.nombre_usuario }}</h2>
        <p class="profile-display-role">{{ roleName }}</p>
        <div class="profile-quick-stats">
          <div class="stat-item">
            <span class="stat-label">Usuario</span>
            <span class="stat-value">@{{ props.currentUser.usuario }}</span>
          </div>
        </div>
      </div>

      <!-- Right Column: Profile Info Form & Password -->
      <div class="profile-forms-column">
        <!-- Edit Profile Form -->
        <div class="profile-card">
          <h3 class="profile-card-title">Información de la Cuenta</h3>
          <form @submit.prevent="handleSaveProfile" class="profile-form">
            <div class="form-grid">
              <div class="form-group">
                <label for="username">Nombre de Usuario</label>
                <input type="text" id="username" :value="props.currentUser.usuario" disabled class="profile-input disabled" />
              </div>
              <div class="form-group">
                <label for="fullName">Nombre Completo *</label>
                <input type="text" id="fullName" v-model="formFullName" required placeholder="Escriba su nombre completo" class="profile-input" />
              </div>
              <div class="form-group">
                <label for="email">Correo Electrónico</label>
                <input type="email" id="email" v-model="formEmail" placeholder="correo@ejemplo.com" class="profile-input" />
              </div>
              <div class="form-group">
                <label for="phone">Teléfono</label>
                <input type="text" id="phone" v-model="formPhone" placeholder="Teléfono" class="profile-input" />
              </div>
              <div class="form-group full-width">
                <label for="dpi">DPI / Documento de Identificación</label>
                <input type="text" id="dpi" v-model="formDpi" placeholder="DPI" class="profile-input" />
              </div>
            </div>
            <div class="form-actions">
              <button type="submit" class="profile-btn primary" :disabled="savingProfile">
                <span v-if="savingProfile">Guardando...</span>
                <span v-else>Guardar Cambios</span>
              </button>
            </div>
          </form>
        </div>

        <!-- Change Password Form -->
        <div class="profile-card">
          <h3 class="profile-card-title">Seguridad y Contraseña</h3>
          <form @submit.prevent="handleUpdatePassword" class="profile-form">
            <div class="form-grid">
              <div class="form-group">
                <label for="newPassword">Nueva Contraseña</label>
                <div class="password-input-wrapper">
                  <input :type="showNewPassword ? 'text' : 'password'" id="newPassword" v-model="formNewPassword" placeholder="••••••••" class="profile-input" />
                  <button type="button" class="password-toggle-btn" @click="showNewPassword = !showNewPassword" title="Mostrar/Ocultar contraseña">
                    <svg v-if="showNewPassword" viewBox="0 0 24 24"><path d="M12 7c2.76 0 5 2.24 5 5 0 .65-.13 1.26-.36 1.82l2.92 2.92c1.51-1.39 2.7-3.18 3.44-5.24-1.73-4.39-6-7.5-11-7.5-1.4 0-2.74.25-3.98.7l2.16 2.16C10.74 7.13 11.35 7 12 7zM2 4.27l2.28 2.28.46.46C3.08 8.3 1.78 10.02 1 12c1.73 4.39 6 7.5 11 7.5 1.55 0 3.03-.3 4.38-.84l.42.42L19.73 22 21 20.73 3.27 3 2 4.27zM7.53 9.8l1.55 1.55c-.05.21-.08.43-.08.65 0 1.66 1.34 3 3 3 .22 0 .44-.03.65-.08l1.55 1.55c-.67.33-1.41.53-2.2.53-2.76 0-5-2.24-5-5 0-.79.2-1.53.53-2.2zm4.31-.78l3.15 3.15.02-.16c0-1.66-1.34-3-3-3l-.17.01z"/></svg>
                    <svg v-else viewBox="0 0 24 24"><path d="M12 4.5C7 4.5 2.73 7.61 1 12c1.73 4.39 6 7.5 11 7.5s9.27-3.11 11-7.5c-1.73-4.39-6-7.5-11-7.5zM12 17c-2.76 0-5-2.24-5-5s2.24-5 5-5 5 2.24 5 5-2.24 5-5 5zm0-8c-1.66 0-3 1.34-3 3s1.34 3 3 3 3-1.34 3-3-1.34-3-3-3z"/></svg>
                  </button>
                </div>
              </div>
              <div class="form-group">
                <label for="confirmPassword">Confirmar Nueva Contraseña</label>
                <div class="password-input-wrapper">
                  <input :type="showConfirmPassword ? 'text' : 'password'" id="confirmPassword" v-model="formConfirmPassword" placeholder="••••••••" class="profile-input" />
                  <button type="button" class="password-toggle-btn" @click="showConfirmPassword = !showConfirmPassword" title="Mostrar/Ocultar contraseña">
                    <svg v-if="showConfirmPassword" viewBox="0 0 24 24"><path d="M12 7c2.76 0 5 2.24 5 5 0 .65-.13 1.26-.36 1.82l2.92 2.92c1.51-1.39 2.7-3.18 3.44-5.24-1.73-4.39-6-7.5-11-7.5-1.4 0-2.74.25-3.98.7l2.16 2.16C10.74 7.13 11.35 7 12 7zM2 4.27l2.28 2.28.46.46C3.08 8.3 1.78 10.02 1 12c1.73 4.39 6 7.5 11 7.5 1.55 0 3.03-.3 4.38-.84l.42.42L19.73 22 21 20.73 3.27 3 2 4.27zM7.53 9.8l1.55 1.55c-.05.21-.08.43-.08.65 0 1.66 1.34 3 3 3 .22 0 .44-.03.65-.08l1.55 1.55c-.67.33-1.41.53-2.2.53-2.76 0-5-2.24-5-5 0-.79.2-1.53.53-2.2zm4.31-.78l3.15 3.15.02-.16c0-1.66-1.34-3-3-3l-.17.01z"/></svg>
                    <svg v-else viewBox="0 0 24 24"><path d="M12 4.5C7 4.5 2.73 7.61 1 12c1.73 4.39 6 7.5 11 7.5s9.27-3.11 11-7.5c-1.73-4.39-6-7.5-11-7.5zM12 17c-2.76 0-5-2.24-5-5s2.24-5 5-5 5 2.24 5 5-2.24 5-5 5zm0-8c-1.66 0-3 1.34-3 3s1.34 3 3 3 3-1.34 3-3-1.34-3-3-3z"/></svg>
                  </button>
                </div>
              </div>
            </div>
            <div class="form-actions">
              <button type="submit" class="profile-btn secondary" :disabled="savingPassword">
                <span v-if="savingPassword">Actualizando...</span>
                <span v-else>Actualizar Contraseña</span>
              </button>
            </div>
          </form>
        </div>
      </div>
    </div>
  </div>
</template>
