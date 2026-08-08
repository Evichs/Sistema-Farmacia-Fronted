<script setup>
import { ref, onMounted, provide, computed } from 'vue';
import LoginView from './components/LoginView.vue';
import DashboardView from './components/DashboardView.vue';
import MedicamentosView from './components/MedicamentosView.vue';
import ClientesView from './components/ClientesView.vue';
import ProveedoresView from './components/ProveedoresView.vue';
import LotesView from './components/LotesView.vue';
import VentasView from './components/VentasView.vue';
import ComprasView from './components/ComprasView.vue';
import CatalogosAuxView from './components/CatalogosAuxView.vue';
import UsuariosView from './components/UsuariosView.vue';
import PerfilView from './components/PerfilView.vue';


// Configuración de la URL de API
const API_URL = import.meta.env.VITE_API_URL;
provide('API_URL', API_URL);

// Estados globales
const currentUser = ref(null);
const currentView = ref('dashboard');
const isLightTheme = ref(false);
const notifications = ref([]);
const isSidebarCollapsed = ref(false);

// Proveer alertas y notificaciones a componentes hijos
const alerts = ref([]);
const addAlert = (message, type = 'warning') => {
  const id = Date.now() + Math.random();
  alerts.value.push({ id, message, type });
  setTimeout(() => {
    removeAlert(id);
  }, 5000);
};
const removeAlert = (id) => {
  alerts.value = alerts.value.filter(a => a.id !== id);
};
provide('addAlert', addAlert);
provide('alerts', alerts);

const handleUpdateUser = (updatedUser) => {
  currentUser.value = updatedUser;
  localStorage.setItem('user', JSON.stringify(updatedUser));
};

// Temas claro y oscuro
const toggleTheme = () => {
  isLightTheme.value = !isLightTheme.value;
  localStorage.setItem('theme', isLightTheme.value ? 'light' : 'dark');
  applyTheme();
};

const applyTheme = () => {
  if (isLightTheme.value) {
    document.body.classList.add('light-theme');
  } else {
    document.body.classList.remove('light-theme');
  }
};

// Autenticación
const handleLoginSuccess = (user) => {
  currentUser.value = user;
  localStorage.setItem('user', JSON.stringify(user));
  addAlert(`¡Bienvenido de nuevo, ${user.nombre_usuario}!`, 'success');
  checkInventoryAlerts();
};

const handleLogout = () => {
  currentUser.value = null;
  localStorage.removeItem('user');
  currentView.value = 'dashboard';
};

// Cargar estado inicial
onMounted(() => {
  const savedTheme = localStorage.getItem('theme');
  if (savedTheme === 'light') {
    isLightTheme.value = true;
    applyTheme();
  }
  
  const savedUser = localStorage.getItem('user');
  if (savedUser) {
    currentUser.value = JSON.parse(savedUser);
    checkInventoryAlerts();
  }
});

// Alertas de inventario al cargar
const checkInventoryAlerts = async () => {
  try {
    const res = await fetch(`${API_URL}/medicamentos?limite=100`);
    if (res.ok) {
      const data = await res.json();
      const lowStock = data.datos.filter(m => m.existencia_total_medicamento < 10);
      lowStock.forEach(m => {
        addAlert(`Bajo Stock: "${m.nombre_medicamento}" tiene solo ${m.existencia_total_medicamento} unidades.`, 'danger');
      });
    }
  } catch (error) {
    console.error('Error cargando alertas de medicamentos:', error);
  }
};

// Menú lateral por categorías
const menuGroups = [
  {
    title: 'General',
    items: [
      { view: 'dashboard', label: 'Paneles de control', icon: 'M3 13h1v7c0 1.1-.9 2-2 2H4c-1.1 0-2-.9-2-2v-7h1M12 2L2 12h3v8c0 .55.45 1 1 1h4v-6h2v6h4c.55 0 1-.45 1-1v-8h3L12 2z' }
    ]
  },
  {
    title: 'Operaciones',
    items: [
      { view: 'ventas', label: 'Ventas (POS)', icon: 'M11.5 2C6.81 2 3 5.81 3 10.5S6.81 19 11.5 19h.5v3c0 .83.67 1.5 1.5 1.5h3c.83 0 1.5-.67 1.5-1.5v-3h.5c4.69 0 8.5-3.81 8.5-8.5S20.69 2 16 2h-4.5zM16 16.5H8.5c-.83 0-1.5-.67-1.5-1.5s.67-1.5 1.5-1.5H16c.83 0 1.5.67 1.5 1.5s-.67 1.5-1.5 1.5z' },
      { view: 'compras', label: 'Compras', icon: 'M7 18c-1.1 0-1.99.9-1.99 2S5.9 22 7 22s2-.9 2-2-.9-2-2-2zM1 2v2h2l3.6 7.59-1.35 2.45c-.16.28-.25.61-.25.96 0 1.1.9 2 2 2h12v-2H7.42c-.14 0-.25-.11-.25-.25l.03-.12.9-1.63h7.45c.75 0 1.41-.41 1.75-1.03l3.58-6.49c.08-.14.12-.31.12-.48 0-.55-.45-1-1-1H5.21l-.94-2H1zm16 16c-1.1 0-1.99.9-1.99 2s.89 2 1.99 2 2-.9 2-2-.9-2-2-2z' }
    ]
  },
  {
    title: 'Inventario',
    items: [
      { view: 'medicamentos', label: 'Medicamentos', icon: 'M4.5 10.5C3.67 10.5 3 11.17 3 12s.67 1.5 1.5 1.5h15c.83 0 1.5-.67 1.5-1.5s-.67-1.5-1.5-1.5h-15z M19 3H5c-1.1 0-2 .9-2 2v3h18V5c0-1.1-.9-2-2-2z M5 21h14c1.1 0 2-.9 2-2v-3H3v3c0 1.1.9 2 2 2z' },
      { view: 'lotes', label: 'Lotes / Inventario', icon: 'M19 3H5c-1.1 0-2 .9-2 2v14c0 1.1.9 2 2 2h14c1.1 0 2-.9 2-2V5c0-1.1-.9-2-2-2zm-2 10H7v-2h10v2zm0-4H7V7h10v2zm0 8H7v-2h10v2z' },
      { view: 'proveedores', label: 'Proveedores', icon: 'M20 4H4c-1.1 0-1.99.9-1.99 2L2 18c0 1.1.9 2 2 2h16c1.1 0 2-.9 2-2V6c0-1.1-.9-2-2-2zm-5 14H4v-4h11v4zm0-5H4V9h11v4zm5 5h-4V9h4v9z' }
    ]
  },
  {
    title: 'Administración',
    items: [
      { view: 'clientes', label: 'Clientes', icon: 'M16 11c1.66 0 2.99-1.34 2.99-3S17.66 5 16 5s-3 1.34-3 3 1.34 3 3 3zm-8 0c1.66 0 2.99-1.34 2.99-3S9.66 5 8 5 5 6.34 5 8s1.34 3 3 3zm0 2c-2.33 0-7 1.17-7 3.5V19h14v-2.5c0-2.33-4.67-3.5-7-3.5zm8 0c-.29 0-.62.02-.97.05 1.16.84 1.97 1.97 1.97 3.45V19h6v-2.5c0-2.33-4.67-3.5-7-3.5z' },
      { view: 'catalogos', label: 'Catálogos Aux', icon: 'M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm1 17h-2v-2h2v2zm2.07-7.75l-.9.92C13.45 12.9 13 13.5 13 15h-2v-.5c0-1.1.45-2.1 1.17-2.83l1.24-1.26c.37-.36.59-.86.59-1.41 0-1.1-.9-2-2-2s-2 .9-2 2H7c0-2.76 2.24-5 5-5s5 2.24 5 5c0 1.04-.42 1.99-1.07 2.75z' }
    ]
  },
  {
    title: 'Seguridad',
    items: [
      { view: 'usuarios', label: 'Usuarios / Roles', icon: 'M12 12c2.21 0 4-1.79 4-4s-1.79-4-4-4-4 1.79-4 4 1.79 4 4 4zm0 2c-2.67 0-8 1.34-8 4v2h16v-2c0-2.66-5.33-4-8-4z' }
    ]
  },
  {
    title: 'Ajustes',
    items: [
      { view: 'perfil', label: 'Mi Perfil', icon: 'M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm0 3c1.66 0 3 1.34 3 3s-1.34 3-3 3-3-1.34-3-3 1.34-3 3-3zm0 14.2c-2.5 0-4.71-1.28-6-3.22.03-1.99 4-3.08 6-3.08 1.99 0 5.97 1.09 6 3.08-1.29 1.94-3.5 3.22-6 3.22z' }
    ]
  }
];

// Nombre de la vista actual
const viewTitle = computed(() => {
  for (const group of menuGroups) {
    const item = group.items.find(i => i.view === currentView.value);
    if (item) return item.label;
  }
  return 'Farmacia';
});
</script>

<template>
  <!-- Si el usuario no está logueado, mostrar login -->
  <LoginView v-if="!currentUser" @login-success="handleLoginSuccess" />

  <!-- Layout Principal de la App -->
  <div v-else class="app-container">
    <!-- Sidebar -->
    <aside class="sidebar" :style="{ width: isSidebarCollapsed ? '70px' : '260px' }">
      <div class="sidebar-brand">
        <span class="sidebar-logo">
          <svg style="width:24px;height:24px;fill:currentColor" viewBox="0 0 24 24">
            <path d="M19 10.5h-5.5V5c0-.83-.67-1.5-1.5-1.5s-1.5.67-1.5 1.5v5.5H5c-.83 0-1.5.67-1.5 1.5s.67 1.5 1.5 1.5h5.5V19c0 .83.67 1.5 1.5 1.5s1.5-.67 1.5-1.5v-5.5H19c.83 0 1.5-.67 1.5-1.5s-.67-1.5-1.5-1.5z"/>
          </svg>
          <span v-if="!isSidebarCollapsed">DREZOC</span>
        </span>
      </div>

      <ul class="sidebar-menu">
        <template v-for="group in menuGroups" :key="group.title">
          <li v-if="!isSidebarCollapsed" class="menu-section-label">{{ group.title }}</li>
          <li v-for="item in group.items" :key="item.view" class="menu-item">
            <a class="menu-link" 
               :class="{ active: currentView === item.view }" 
               @click="currentView = item.view"
               :title="item.label">
              <svg class="menu-icon" viewBox="0 0 24 24">
                <path :d="item.icon" />
              </svg>
              <span v-if="!isSidebarCollapsed">{{ item.label }}</span>
            </a>
          </li>
        </template>
      </ul>
    </aside>

    <!-- Main Content Wrapper -->
    <div class="main-wrapper">
      <!-- Topbar -->
      <header class="topbar">
        <div class="topbar-left">
          <button class="menu-toggle-btn" @click="isSidebarCollapsed = !isSidebarCollapsed">
            <svg style="width:24px;height:24px;fill:currentColor" viewBox="0 0 24 24">
              <path d="M3 18h18v-2H3v2zm0-5h18v-2H3v2zm0-7v2h18V6H3z"/>
            </svg>
          </button>
        </div>

        <div class="topbar-right">
          <!-- Conmutador de Tema Claro/Oscuro -->
          <button class="topbar-action-btn" @click="toggleTheme" title="Cambiar tema claro/oscuro">
            <svg v-if="isLightTheme" style="width:20px;height:20px;fill:currentColor" viewBox="0 0 24 24">
              <!-- Sol -->
              <path d="M12 7c-2.76 0-5 2.24-5 5s2.24 5 5 5 5-2.24 5-5-2.24-5-5-5zM2 13h2c.55 0 1-.45 1-1s-.45-1-1-1H2c-.55 0-1 .45-1 1s.45 1 1 1zm18 0h2c.55 0 1-.45 1-1s-.45-1-1-1h-2c-.55 0-1 .45-1 1s.45 1 1 1zM11 2v2c0 .55.45 1 1 1s1-.45 1-1V2c0-.55-.45-1-1-1s-1 .45-1 1zm0 18v2c0 .55.45 1 1 1s1-.45 1-1v-2c0-.55-.45-1-1-1s-1 .45-1 1zM5.99 4.58c-.39-.39-1.03-.39-1.41 0s-.39 1.03 0 1.41l1.06 1.06c.39.39 1.03.39 1.41 0s.39-1.03 0-1.41L5.99 4.58zm12.37 12.37c-.39-.39-1.03-.39-1.41 0s-.39 1.03 0 1.41l1.06 1.06c.39.39 1.03.39 1.41 0s.39-1.02 0-1.41l-1.06-1.06zm1.06-12.37c-.39-.39-1.03-.39-1.41 0l-1.06 1.06c-.39.39-.39 1.03 0 1.41s1.03.39 1.41 0l1.06-1.06c.39-.38.39-1.02 0-1.41zm-12.37 12.37c-.39-.39-1.03-.39-1.41 0l-1.06 1.06c-.39.39-.39 1.03 0 1.41s1.03.39 1.41 0l1.06-1.06c.39-.38.39-1.02 0-1.41z"/>
            </svg>
            <svg v-else style="width:20px;height:20px;fill:currentColor" viewBox="0 0 24 24">
              <!-- Luna -->
              <path d="M9.37 5.51c-.18.64-.27 1.31-.27 2 0 4.14 3.36 7.5 7.5 7.5.69 0 1.36-.09 2-.27 1.54 5.25-2.29 9.76-7.5 9.76-5.25 0-9.5-4.25-9.5-9.5 0-5.21 4.51-9.04 7.77-9.49z"/>
            </svg>
          </button>

          <!-- Alertas rápidas (Notificaciones) -->
          <button class="topbar-action-btn" title="Alertas de Inventario">
            <svg style="width:20px;height:20px;fill:currentColor" viewBox="0 0 24 24">
              <path d="M12 22c1.1 0 2-.9 2-2h-4c0 1.1.89 2 2 2zm6-6v-5c0-3.07-1.64-5.64-4.5-6.32V4c0-.83-.67-1.5-1.5-1.5s-1.5.67-1.5 1.5v.68C7.63 5.36 6 7.92 6 11v5l-2 2v1h16v-1l-2-2z"/>
            </svg>
            <span v-if="alerts.length > 0" class="badge">{{ alerts.length }}</span>
          </button>

          <!-- Perfil de usuario -->
          <div class="user-profile-group" style="display: flex; align-items: center; gap: 12px;">
            <div class="user-profile" @click="currentView = 'perfil'" title="Ver mi perfil" style="cursor: pointer; display: flex; align-items: center; gap: 10px;">
              <img class="avatar" src="https://images.unsplash.com/photo-1535713875002-d1d0cf377fde?w=100&auto=format&fit=crop&q=60" alt="Usuario" />
              <div class="user-info">
                <span class="user-name">{{ currentUser.nombre_usuario }}</span>
                <span class="user-role">Ver Perfil</span>
              </div>
            </div>
            <button class="topbar-action-btn" @click="handleLogout" title="Cerrar Sesión" style="margin-left: 8px;">
              <svg style="width:20px;height:20px;fill:currentColor" viewBox="0 0 24 24">
                <path d="M14.08 15.59L16.67 13H7v-2h9.67l-2.59-2.59L15.5 7l5 5-5 5-1.42-1.41M19 12c0 3.87-3.13 7-7 7s-7-3.13-7-7 3.13-7 7-7c1.93 0 3.68.78 4.95 2.05l-1.41 1.41C11.64 5.54 10.88 5 9 5c-3.87 0-7 3.13-7 7s3.13 7 7 7 7-3.13 7-7h2z"/>
              </svg>
            </button>
          </div>
        </div>
      </header>

      <!-- Page Content Area -->
      <main class="page-container">
        <!-- Notificaciones de Alerta en vivo -->
        <TransitionGroup name="toast" tag="div" class="alert-list">
          <div v-for="alert in alerts" :key="alert.id" class="alert-item" :class="alert.type">
            <span class="alert-message">{{ alert.message }}</span>
            <button class="alert-close-btn" @click="removeAlert(alert.id)">&times;</button>
          </div>
        </TransitionGroup>

        <div class="page-header">
          <div>
            <h1 class="page-title">{{ viewTitle }}</h1>
            <div class="page-breadcrumbs">
              <a href="#">Drezoc</a> <span>/</span> {{ viewTitle }}
            </div>
          </div>
        </div>

        <!-- Renderizado de Vistas SPA -->
        <DashboardView v-if="currentView === 'dashboard'" />
        <MedicamentosView v-else-if="currentView === 'medicamentos'" />
        <ClientesView v-else-if="currentView === 'clientes'" />
        <ProveedoresView v-else-if="currentView === 'proveedores'" />
        <LotesView v-else-if="currentView === 'lotes'" />
        <VentasView v-else-if="currentView === 'ventas'" :activeUser="currentUser" />
        <ComprasView v-else-if="currentView === 'compras'" />
        <CatalogosAuxView v-else-if="currentView === 'catalogos'" />
        <UsuariosView v-else-if="currentView === 'usuarios'" />
        <PerfilView v-else-if="currentView === 'perfil'" :currentUser="currentUser" @update-user="handleUpdateUser" />
      </main>
    </div>
  </div>
</template>
