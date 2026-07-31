<script setup>
import { ref, onMounted, inject } from 'vue';

const emit = defineEmits(['login-success']);
const API_URL = inject('API_URL');
const addAlert = inject('addAlert');

const isLoginMode = ref(true); // Alterna entre login y registro

// Campos de Login
const loginUsername = ref('');
const loginPassword = ref('');
const showLoginPassword = ref(false);
const rememberMe = ref(false);

// Campos de Registro
const regUsername = ref('');
const regPassword = ref('');
const showRegPassword = ref(false);
const regFullName = ref('');
const regPhone = ref('');
const regEmail = ref('');
const regDpi = ref('');
const regRoleId = ref(null);

const rolesList = ref([]);
const loading = ref(false);
const errorMsg = ref('');

// Cargar Roles para el registro
const loadRoles = async () => {
  try {
    const res = await fetch(`${API_URL}/roles?limite=100`);
    if (res.ok) {
      const data = await res.json();
      rolesList.value = data.datos || [];
      if (rolesList.value.length > 0) {
        regRoleId.value = rolesList.value[0].id_rol;
      }
    }
  } catch (err) {
    console.error('Error al cargar roles:', err);
  }
};

onMounted(() => {
  loadRoles();
  // Auto-llenar campos de login si se recordó previamente
  const savedUser = localStorage.getItem('remembered_username');
  if (savedUser) {
    loginUsername.value = savedUser;
    rememberMe.value = true;
  }
});

// Manejar Inicio de Sesión
const handleLogin = async () => {
  if (!loginUsername.value || !loginPassword.value) {
    errorMsg.value = 'Por favor complete todos los campos';
    return;
  }
  
  loading.value = true;
  errorMsg.value = '';
  
  try {
    // Buscamos en la lista de usuarios (nuestro backend no tiene endpoint de auth específico, por lo que consultamos la lista de usuarios)
    const res = await fetch(`${API_URL}/usuarios?limite=100`);
    if (!res.ok) throw new Error('Error al conectar con el servidor');
    
    const data = await res.json();
    const users = data.datos || [];
    
    // Buscar coincidencia de usuario
    const foundUser = users.find(u => u.usuario.toLowerCase() === loginUsername.value.toLowerCase());
    
    if (foundUser) {
      if (!foundUser.estado_usuario) {
        errorMsg.value = 'El usuario está inactivo en el sistema';
        return;
      }
      if (foundUser.password !== loginPassword.value) {
        errorMsg.value = 'Contraseña incorrecta';
        return;
      }
      
      if (rememberMe.value) {
        localStorage.setItem('remembered_username', loginUsername.value);
      } else {
        localStorage.removeItem('remembered_username');
      }
      
      emit('login-success', foundUser);
    } else {
      errorMsg.value = 'Usuario no encontrado';
    }
  } catch (err) {
    errorMsg.value = 'Error al comunicarse con el backend. Asegúrese de que esté encendido.';
    console.error(err);
  } finally {
    loading.value = false;
  }
};

// Crear un Rol Inicial si la BD está vacía
const createInitialRoleIfNeeded = async () => {
  if (rolesList.value.length === 0) {
    try {
      const res = await fetch(`${API_URL}/roles`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ nombre_rol: 'Administrador', estado_rol: true })
      });
      if (res.ok) {
        await loadRoles();
      }
    } catch (err) {
      console.error('Error al crear rol inicial:', err);
    }
  }
};

// Manejar Registro
const handleRegister = async () => {
  if (!regUsername.value || !regPassword.value || !regFullName.value || !regEmail.value) {
    errorMsg.value = 'Por favor complete los campos obligatorios';
    return;
  }

  loading.value = true;
  errorMsg.value = '';

  try {
    await createInitialRoleIfNeeded();
    
    if (!regRoleId.value && rolesList.value.length > 0) {
      regRoleId.value = rolesList.value[0].id_rol;
    }

    if (!regRoleId.value) {
      throw new Error('Debe existir al menos un Rol en el sistema para registrar usuarios.');
    }

    const payload = {
      usuario: regUsername.value,
      password: regPassword.value,
      nombre_usuario: regFullName.value,
      telefono_usuario: regPhone.value || undefined,
      correo_usuario: regEmail.value,
      dpi_usuario: regDpi.value || undefined,
      id_rol: Number(regRoleId.value),
      estado_usuario: true
    };

    const res = await fetch(`${API_URL}/usuarios`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload)
    });

    if (res.ok) {
      const newUser = await res.json();
      isLoginMode.value = true;
      loginUsername.value = newUser.usuario;
      loginPassword.value = '';
      addAlert('Registro exitoso. Inicie sesión con sus nuevas credenciales.', 'success');
    } else {
      const errData = await res.json();
      errorMsg.value = errData.message || 'Error al registrar el usuario';
    }
  } catch (err) {
    errorMsg.value = err.message || 'Error en el servidor al registrar';
    console.error(err);
  } finally {
    loading.value = false;
  }
};
</script>

<template>
  <div class="login-container">
    <!-- Mitad Izquierda con Imagen Ilustrativa -->
    <div class="login-left" style="background-image: url('https://images.unsplash.com/photo-1586015555751-63bb77f4322a?w=1000&auto=format&fit=crop&q=80')">
      <div class="login-left-content">
        <h2 class="login-left-title">Sistema de Farmacia Evichs</h2>
        <p class="login-left-desc">Panel administrativo avanzado para la gestión de compras, ventas y control de inventario de medicamentos.</p>
      </div>
    </div>

    <!-- Mitad Derecha con Formularios -->
    <div class="login-right">
      <div class="login-header">
        <div class="login-brand">
          <svg style="width:28px;height:28px;fill:currentColor" viewBox="0 0 24 24">
            <path d="M12 17.27L18.18 21l-1.64-7.03L22 9.24l-7.19-.61L12 2 9.19 8.63 2 9.24l5.46 4.73L5.82 21z"/>
          </svg>
          Farmacia Evichs
        </div>
        <h3 class="login-title">{{ isLoginMode ? '¡Bienvenido de nuevo!' : 'Crear una cuenta' }}</h3>
        <p class="login-subtitle">
          {{ isLoginMode 
              ? 'Introduce tu usuario y contraseña para acceder al panel de administración.' 
              : 'Completa los siguientes datos para dar de alta un nuevo usuario.' }}
        </p>
      </div>

      <!-- Alerta de Error -->
      <div v-if="errorMsg" class="alert-item danger" style="padding: 10px 14px; margin-bottom: 20px;">
        <span class="alert-message" style="font-size: 13px;">{{ errorMsg }}</span>
      </div>

      <!-- Formulario de LOGIN -->
      <form v-if="isLoginMode" @submit.prevent="handleLogin">
        <div class="form-group">
          <label class="form-label">Nombre de Usuario / Nickname</label>
          <input type="text" class="form-control" v-model="loginUsername" placeholder="Introduce tu usuario" required autocomplete="username">
        </div>

        <div class="form-group">
          <label class="form-label">Contraseña</label>
          <div class="password-input-wrapper">
            <input :type="showLoginPassword ? 'text' : 'password'" class="form-control" v-model="loginPassword" placeholder="Introduce tu contraseña" required autocomplete="current-password">
            <button type="button" class="password-toggle-btn" @click="showLoginPassword = !showLoginPassword" title="Mostrar/Ocultar contraseña">
              <svg v-if="showLoginPassword" viewBox="0 0 24 24"><path d="M12 7c2.76 0 5 2.24 5 5 0 .65-.13 1.26-.36 1.82l2.92 2.92c1.51-1.39 2.7-3.18 3.44-5.24-1.73-4.39-6-7.5-11-7.5-1.4 0-2.74.25-3.98.7l2.16 2.16C10.74 7.13 11.35 7 12 7zM2 4.27l2.28 2.28.46.46C3.08 8.3 1.78 10.02 1 12c1.73 4.39 6 7.5 11 7.5 1.55 0 3.03-.3 4.38-.84l.42.42L19.73 22 21 20.73 3.27 3 2 4.27zM7.53 9.8l1.55 1.55c-.05.21-.08.43-.08.65 0 1.66 1.34 3 3 3 .22 0 .44-.03.65-.08l1.55 1.55c-.67.33-1.41.53-2.2.53-2.76 0-5-2.24-5-5 0-.79.2-1.53.53-2.2zm4.31-.78l3.15 3.15.02-.16c0-1.66-1.34-3-3-3l-.17.01z"/></svg>
              <svg v-else viewBox="0 0 24 24"><path d="M12 4.5C7 4.5 2.73 7.61 1 12c1.73 4.39 6 7.5 11 7.5s9.27-3.11 11-7.5c-1.73-4.39-6-7.5-11-7.5zM12 17c-2.76 0-5-2.24-5-5s2.24-5 5-5 5 2.24 5 5-2.24 5-5 5zm0-8c-1.66 0-3 1.34-3 3s1.34 3 3 3 3-1.34 3-3-1.34-3-3-3z"/></svg>
            </button>
          </div>
        </div>

        <div class="form-group" style="display: flex; justify-content: space-between; align-items: center;">
          <label class="form-check">
            <input type="checkbox" class="form-check-input" v-model="rememberMe">
            <span class="form-check-label">Acuérdate de mí</span>
          </label>
        </div>

        <button type="submit" class="btn btn-primary" style="width: 100%; margin-top: 10px;" :disabled="loading">
          {{ loading ? 'Ingresando...' : 'Acceso' }}
        </button>
      </form>

      <!-- Formulario de REGISTRO -->
      <form v-else @submit.prevent="handleRegister">
        <div class="form-group">
          <label class="form-label">Nombre Completo *</label>
          <input type="text" class="form-control" v-model="regFullName" placeholder="Juan Pérez" required>
        </div>

        <div class="form-row">
          <div class="form-group">
            <label class="form-label">Nombre de Usuario *</label>
            <input type="text" class="form-control" v-model="regUsername" placeholder="jperez" required>
          </div>
          <div class="form-group">
            <label class="form-label">Contraseña *</label>
            <div class="password-input-wrapper">
              <input :type="showRegPassword ? 'text' : 'password'" class="form-control" v-model="regPassword" placeholder="******" required autocomplete="new-password">
              <button type="button" class="password-toggle-btn" @click="showRegPassword = !showRegPassword" title="Mostrar/Ocultar contraseña">
                <svg v-if="showRegPassword" viewBox="0 0 24 24"><path d="M12 7c2.76 0 5 2.24 5 5 0 .65-.13 1.26-.36 1.82l2.92 2.92c1.51-1.39 2.7-3.18 3.44-5.24-1.73-4.39-6-7.5-11-7.5-1.4 0-2.74.25-3.98.7l2.16 2.16C10.74 7.13 11.35 7 12 7zM2 4.27l2.28 2.28.46.46C3.08 8.3 1.78 10.02 1 12c1.73 4.39 6 7.5 11 7.5 1.55 0 3.03-.3 4.38-.84l.42.42L19.73 22 21 20.73 3.27 3 2 4.27zM7.53 9.8l1.55 1.55c-.05.21-.08.43-.08.65 0 1.66 1.34 3 3 3 .22 0 .44-.03.65-.08l1.55 1.55c-.67.33-1.41.53-2.2.53-2.76 0-5-2.24-5-5 0-.79.2-1.53.53-2.2zm4.31-.78l3.15 3.15.02-.16c0-1.66-1.34-3-3-3l-.17.01z"/></svg>
                <svg v-else viewBox="0 0 24 24"><path d="M12 4.5C7 4.5 2.73 7.61 1 12c1.73 4.39 6 7.5 11 7.5s9.27-3.11 11-7.5c-1.73-4.39-6-7.5-11-7.5zM12 17c-2.76 0-5-2.24-5-5s2.24-5 5-5 5 2.24 5 5-2.24 5-5 5zm0-8c-1.66 0-3 1.34-3 3s1.34 3 3 3 3-1.34 3-3-1.34-3-3-3z"/></svg>
              </button>
            </div>
          </div>
        </div>

        <div class="form-row">
          <div class="form-group">
            <label class="form-label">Correo Electrónico *</label>
            <input type="email" class="form-control" v-model="regEmail" placeholder="juan@correo.com" required>
          </div>
          <div class="form-group">
            <label class="form-label">Teléfono</label>
            <input type="text" class="form-control" v-model="regPhone" placeholder="5555-4444">
          </div>
        </div>

        <div class="form-row">
          <div class="form-group">
            <label class="form-label">DPI / Cédula</label>
            <input type="text" class="form-control" v-model="regDpi" placeholder="2999 12345 0101">
          </div>
          <div class="form-group">
            <label class="form-label">Rol del Usuario *</label>
            <select class="form-control" v-model="regRoleId" required>
              <option v-for="rol in rolesList" :key="rol.id_rol" :value="rol.id_rol">
                {{ rol.nombre_rol }}
              </option>
              <option v-if="rolesList.length === 0" :value="1">Administrador (Se creará en BD)</option>
            </select>
          </div>
        </div>

        <button type="submit" class="btn btn-success" style="width: 100%; margin-top: 10px;" :disabled="loading">
          {{ loading ? 'Registrando...' : 'Registrar Cuenta' }}
        </button>
      </form>

      <!-- Alternador de Modos -->
      <div class="divider">o</div>
      <div style="text-align: center; font-size: 14px;">
        <span style="color: var(--text-muted)">
          {{ isLoginMode ? '¿No tienes cuenta?' : '¿Ya tienes una cuenta?' }}
        </span>
        <a href="#" style="color: var(--primary-color); font-weight: 600; text-decoration: none; margin-left: 8px;" @click.prevent="isLoginMode = !isLoginMode; errorMsg = ''">
          {{ isLoginMode ? 'Regístrate aquí' : 'Inicia Sesión' }}
        </a>
      </div>
    </div>
  </div>
</template>
