<script setup>
import { ref, onMounted, inject } from 'vue';

const API_URL = inject('API_URL');
const addAlert = inject('addAlert');

const totalSales = ref(0);
const totalPurchases = ref(0);
const totalMedicines = ref(0);
const totalClients = ref(0);

const recentSalesList = ref([]);
const loading = ref(true);

const loadDashboardData = async () => {
  loading.value = true;
  try {
    // 1. Ventas
    const salesRes = await fetch(`${API_URL}/ventas?limite=100`);
    if (salesRes.ok) {
      const salesData = await salesRes.json();
      const sales = salesData.datos || [];
      totalSales.value = sales.reduce((acc, curr) => acc + Number(curr.total_venta || 0), 0);
      recentSalesList.value = sales.slice(0, 5); // Tomar las últimas 5 ventas
    }

    // 2. Compras
    const purchasesRes = await fetch(`${API_URL}/compras?limite=100`);
    if (purchasesRes.ok) {
      const purchasesData = await purchasesRes.json();
      const purchases = purchasesData.datos || [];
      totalPurchases.value = purchases.reduce((acc, curr) => acc + Number(curr.total_compra || 0), 0);
    }

    // 3. Medicamentos
    const medsRes = await fetch(`${API_URL}/medicamentos?limite=100`);
    if (medsRes.ok) {
      const medsData = await medsRes.json();
      totalMedicines.value = medsData.total || 0;
    }

    // 4. Clientes
    const clientsRes = await fetch(`${API_URL}/clientes?limite=100`);
    if (clientsRes.ok) {
      const clientsData = await clientsRes.json();
      totalClients.value = clientsData.total || 0;
    }

  } catch (error) {
    console.error('Error al cargar datos del dashboard:', error);
  } finally {
    loading.value = false;
  }
};

onMounted(() => {
  loadDashboardData();
});

// Formatear monedas
const formatCurrency = (val) => {
  return new Intl.NumberFormat('es-GT', { style: 'currency', currency: 'GTQ' }).format(val);
};

// Formatear fechas
const formatDate = (dateStr) => {
  if (!dateStr) return '';
  const date = new Date(dateStr);
  return date.toLocaleDateString() + ' ' + date.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
};
</script>

<template>
  <div>
    <!-- Spinner de carga -->
    <div v-if="loading" style="text-align: center; padding: 40px;">
      <div style="font-size: 16px; color: var(--text-secondary);">Cargando estadísticas del sistema...</div>
    </div>

    <div v-else>
      <!-- Rejilla de tarjetas de estadísticas -->
      <div class="card-grid">
        <!-- Ingresos Totales (Ventas) -->
        <div class="stats-card">
          <div class="stats-label">Ingresos de Ventas</div>
          <div class="stats-value">{{ formatCurrency(totalSales) }}</div>
          <div class="stats-trend up">
            <svg style="width:14px;height:14px;fill:currentColor" viewBox="0 0 24 24">
              <path d="M5 13h3v8h2v-8h3l-4-4-4 4z"/>
            </svg>
            +12.4%
          </div>
          <!-- Mini gráfico SVG -->
          <div class="stats-chart-wrapper">
            <svg viewBox="0 0 100 30" style="width: 100%; height: 100%;">
              <path d="M0,25 Q15,5 30,20 T60,10 T90,22 T100,5" fill="none" stroke="var(--success-color)" stroke-width="2" />
              <path d="M0,25 Q15,5 30,20 T60,10 T90,22 T100,5 L100,30 L0,30 Z" fill="rgba(16, 185, 129, 0.08)" />
            </svg>
          </div>
        </div>

        <!-- Gastos Totales (Compras) -->
        <div class="stats-card">
          <div class="stats-label">Gastos por Compras</div>
          <div class="stats-value">{{ formatCurrency(totalPurchases) }}</div>
          <div class="stats-trend down">
            <svg style="width:14px;height:14px;fill:currentColor" viewBox="0 0 24 24">
              <path d="M19 11h-3V3h-2v8h-3l4 4 4-4z"/>
            </svg>
            -4.2%
          </div>
          <!-- Mini gráfico SVG -->
          <div class="stats-chart-wrapper">
            <svg viewBox="0 0 100 30" style="width: 100%; height: 100%;">
              <path d="M0,10 Q20,25 40,15 T70,22 T100,8" fill="none" stroke="var(--danger-color)" stroke-width="2" />
              <path d="M0,10 Q20,25 40,15 T70,22 T100,8 L100,30 L0,30 Z" fill="rgba(239, 68, 68, 0.08)" />
            </svg>
          </div>
        </div>

        <!-- Medicamentos Activos -->
        <div class="stats-card">
          <div class="stats-label">Medicamentos en Catálogo</div>
          <div class="stats-value">{{ totalMedicines }}</div>
          <div class="stats-trend up">
            <svg style="width:14px;height:14px;fill:currentColor" viewBox="0 0 24 24">
              <path d="M5 13h3v8h2v-8h3l-4-4-4 4z"/>
            </svg>
            Activos
          </div>
          <!-- Mini gráfico SVG -->
          <div class="stats-chart-wrapper">
            <svg viewBox="0 0 100 30" style="width: 100%; height: 100%;">
              <path d="M0,28 L15,22 L30,24 L45,18 L60,14 L75,10 L90,5 L100,2" fill="none" stroke="var(--primary-color)" stroke-width="2" />
            </svg>
          </div>
        </div>

        <!-- Clientes -->
        <div class="stats-card">
          <div class="stats-label">Clientes Registrados</div>
          <div class="stats-value">{{ totalClients }}</div>
          <div class="stats-trend up">
            <svg style="width:14px;height:14px;fill:currentColor" viewBox="0 0 24 24">
              <path d="M5 13h3v8h2v-8h3l-4-4-4 4z"/>
            </svg>
            +18.9%
          </div>
          <!-- Mini gráfico SVG -->
          <div class="stats-chart-wrapper">
            <svg viewBox="0 0 100 30" style="width: 100%; height: 100%;">
              <path d="M0,25 C20,10 40,30 60,15 C80,0 90,20 100,10" fill="none" stroke="var(--accent-color)" stroke-width="2" />
            </svg>
          </div>
        </div>
      </div>

      <!-- Secciones de Gráficos e Historial -->
      <div style="display: grid; grid-template-columns: 1fr 1.5fr; gap: 24px; margin-bottom: 30px;">
        
        <!-- Tarjeta Izquierda: Los mejores productos -->
        <div class="stats-card" style="height: fit-content;">
          <h3 class="pos-section-title" style="margin-bottom: 24px;">Productos y Cobertura</h3>
          <div style="display: flex; flex-direction: column; align-items: center; justify-content: center; padding: 20px 0;">
            <!-- Doughnut Chart en SVG -->
            <svg width="180" height="180" viewBox="0 0 36 36" style="transform: rotate(-90deg);">
              <!-- Círculo fondo -->
              <circle cx="18" cy="18" r="15.915" fill="none" stroke="var(--border-color)" stroke-width="3" />
              <!-- Círculo progreso Ventas -->
              <circle cx="18" cy="18" r="15.915" fill="none" stroke="var(--primary-color)" stroke-width="3.5" 
                      stroke-dasharray="70 30" stroke-dashoffset="0" stroke-linecap="round"/>
              <!-- Círculo progreso Compras -->
              <circle cx="18" cy="18" r="15.915" fill="none" stroke="var(--success-color)" stroke-width="3.5" 
                      stroke-dasharray="20 80" stroke-dashoffset="-70" stroke-linecap="round"/>
            </svg>
            <div style="display: flex; gap: 20px; margin-top: 24px;">
              <div style="display: flex; align-items: center; gap: 8px;">
                <div style="width: 12px; height: 12px; border-radius: 50%; background-color: var(--primary-color);"></div>
                <span style="font-size: 13px; color: var(--text-secondary);">Venta de Medicamentos</span>
              </div>
              <div style="display: flex; align-items: center; gap: 8px;">
                <div style="width: 12px; height: 12px; border-radius: 50%; background-color: var(--success-color);"></div>
                <span style="font-size: 13px; color: var(--text-secondary);">Existencias Libres</span>
              </div>
            </div>
          </div>
        </div>

        <!-- Tarjeta Derecha: Análisis de Ventas -->
        <div class="stats-card">
          <h3 class="pos-section-title" style="margin-bottom: 24px;">Análisis de Ventas (Volumen Diario)</h3>
          <!-- Bar chart simulado en SVG -->
          <div style="height: 180px; display: flex; align-items: flex-end; justify-content: space-between; padding-bottom: 10px; border-bottom: 1px solid var(--border-color);">
            <!-- Barra 1 -->
            <div style="flex: 1; display: flex; flex-direction: column; align-items: center; gap: 8px;">
              <div style="width: 32px; height: 60px; background: linear-gradient(180deg, var(--primary-color) 0%, rgba(59,130,246,0.3) 100%); border-radius: 4px 4px 0 0;"></div>
              <span style="font-size: 11px; color: var(--text-muted);">Lun</span>
            </div>
            <!-- Barra 2 -->
            <div style="flex: 1; display: flex; flex-direction: column; align-items: center; gap: 8px;">
              <div style="width: 32px; height: 110px; background: linear-gradient(180deg, var(--primary-color) 0%, rgba(59,130,246,0.3) 100%); border-radius: 4px 4px 0 0;"></div>
              <span style="font-size: 11px; color: var(--text-muted);">Mar</span>
            </div>
            <!-- Barra 3 -->
            <div style="flex: 1; display: flex; flex-direction: column; align-items: center; gap: 8px;">
              <div style="width: 32px; height: 85px; background: linear-gradient(180deg, var(--primary-color) 0%, rgba(59,130,246,0.3) 100%); border-radius: 4px 4px 0 0;"></div>
              <span style="font-size: 11px; color: var(--text-muted);">Mié</span>
            </div>
            <!-- Barra 4 -->
            <div style="flex: 1; display: flex; flex-direction: column; align-items: center; gap: 8px;">
              <div style="width: 32px; height: 140px; background: linear-gradient(180deg, var(--accent-color) 0%, rgba(99,102,241,0.3) 100%); border-radius: 4px 4px 0 0;"></div>
              <span style="font-size: 11px; color: var(--text-muted);">Jue</span>
            </div>
            <!-- Barra 5 -->
            <div style="flex: 1; display: flex; flex-direction: column; align-items: center; gap: 8px;">
              <div style="width: 32px; height: 100px; background: linear-gradient(180deg, var(--primary-color) 0%, rgba(59,130,246,0.3) 100%); border-radius: 4px 4px 0 0;"></div>
              <span style="font-size: 11px; color: var(--text-muted);">Vie</span>
            </div>
            <!-- Barra 6 -->
            <div style="flex: 1; display: flex; flex-direction: column; align-items: center; gap: 8px;">
              <div style="width: 32px; height: 165px; background: linear-gradient(180deg, var(--success-color) 0%, rgba(16,185,129,0.3) 100%); border-radius: 4px 4px 0 0;"></div>
              <span style="font-size: 11px; color: var(--text-muted);">Sáb</span>
            </div>
            <!-- Barra 7 -->
            <div style="flex: 1; display: flex; flex-direction: column; align-items: center; gap: 8px;">
              <div style="width: 32px; height: 50px; background: linear-gradient(180deg, var(--primary-color) 0%, rgba(59,130,246,0.3) 100%); border-radius: 4px 4px 0 0;"></div>
              <span style="font-size: 11px; color: var(--text-muted);">Dom</span>
            </div>
          </div>
        </div>

      </div>

      <!-- Ventas recientes -->
      <div class="table-responsive">
        <div style="padding: 20px 24px; font-weight: 700; border-bottom: 1px solid var(--border-color); font-size: 16px;">
          Últimas Transacciones Registradas
        </div>
        <table class="table">
          <thead>
            <tr>
              <th>ID Venta</th>
              <th>ID Cliente</th>
              <th>Fecha y Hora</th>
              <th>Estado</th>
              <th>Monto Total</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="sale in recentSalesList" :key="sale.id_venta">
              <td>#000{{ sale.id_venta }}</td>
              <td>{{ sale.id_cliente }}</td>
              <td>{{ formatDate(sale.fecha_venta) }}</td>
              <td>
                <span class="status-badge" :class="sale.estado_venta ? 'active' : 'inactive'">
                  {{ sale.estado_venta ? 'Completado' : 'Cancelado' }}
                </span>
              </td>
              <td style="font-weight: 700;">{{ formatCurrency(sale.total_venta) }}</td>
            </tr>
            <tr v-if="recentSalesList.length === 0">
              <td colspan="5" style="text-align: center; color: var(--text-muted); padding: 30px;">
                No hay transacciones registradas todavía. ¡Realiza una venta en la sección POS!
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>
