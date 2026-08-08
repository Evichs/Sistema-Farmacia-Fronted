<script setup>
import { ref, onMounted, inject, computed } from 'vue';

const props = defineProps({
  activeUser: { type: Object, required: true }
});

const API_URL = inject('API_URL');
const addAlert = inject('addAlert');

const clientesList = ref([]);
const medicamentosList = ref([]);
const metodosPagoList = ref([]);
const lotesList = ref([]);

// POS Carrito y Selección
const selectedClienteId = ref('');
const searchMedTerm = ref('');
const medSearchResults = ref([]);
const cartItems = ref([]);
const selectedMetodoPagoId = ref('');

// Control de Modales
const isNewClientModalOpen = ref(false);
const quickClientNombre = ref('');
const quickClientNit = ref('CF');

const isInvoiceModalOpen = ref(false);
const savedInvoice = ref(null);
const invoiceItems = ref([]);
const invoicePayment = ref(null);

const loading = ref(false);

// Cargar Datos Iniciales
const loadData = async () => {
  try {
    const cliRes = await fetch(`${API_URL}/clientes?limite=100`);
    if (cliRes.ok) {
      const d = await cliRes.json();
      let list = d.datos || [];
      if (list.length === 0) {
        try {
          await fetch(`${API_URL}/clientes`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ nombre_cliente: 'Consumidor Final', nit_cliente: 'CF', estado_cliente: true })
          });
          const retryRes = await fetch(`${API_URL}/clientes?limite=100`);
          if (retryRes.ok) {
            const retryData = await retryRes.json();
            list = retryData.datos || [];
          }
        } catch (err) {
          console.error('Error al crear cliente inicial:', err);
        }
      }
      clientesList.value = list;
      if (clientesList.value.length > 0) {
        selectedClienteId.value = clientesList.value[0].id_cliente;
      }
    }

    const payRes = await fetch(`${API_URL}/metodos-pago?limite=100`);
    if (payRes.ok) {
      const d = await payRes.json();
      let list = d.datos || [];
      if (list.length === 0) {
        try {
          await fetch(`${API_URL}/metodos-pago`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ nombre_metodo_pago: 'Efectivo', estado_metodo_pago: true })
          });
          await fetch(`${API_URL}/metodos-pago`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ nombre_metodo_pago: 'Tarjeta', estado_metodo_pago: true })
          });
          const retryRes = await fetch(`${API_URL}/metodos-pago?limite=100`);
          if (retryRes.ok) {
            const retryData = await retryRes.json();
            list = retryData.datos || [];
          }
        } catch (err) {
          console.error('Error al crear métodos de pago iniciales:', err);
        }
      }
      metodosPagoList.value = list;
      if (metodosPagoList.value.length > 0) {
        selectedMetodoPagoId.value = metodosPagoList.value[0].id_metodo_pago;
      }
    }

    const lotRes = await fetch(`${API_URL}/lotes?limite=100`);
    if (lotRes.ok) {
      const d = await lotRes.json();
      lotesList.value = d.datos || [];
    }

    // Cargar medicamentos activos
    const medRes = await fetch(`${API_URL}/medicamentos?limite=100`);
    if (medRes.ok) {
      const d = await medRes.json();
      medicamentosList.value = d.datos || [];
    }
  } catch (error) {
    console.error('Error al cargar datos en POS:', error);
  }
};

onMounted(() => {
  loadData();
});

// Filtrar medicamentos en tiempo real (Catálogo visual)
const filteredMedicamentos = computed(() => {
  const list = medicamentosList.value.filter(m => m.estado_medicamento);
  if (!searchMedTerm.value) return list;
  const term = searchMedTerm.value.toLowerCase();
  return list.filter(m => 
    m.nombre_medicamento.toLowerCase().includes(term) ||
    (m.codigo_barras_medicamento && m.codigo_barras_medicamento.toLowerCase().includes(term)) ||
    (m.componente_activo && m.componente_activo.toLowerCase().includes(term))
  );
});

// Agregar al carrito
const addToCart = (med) => {
  if (med.existencia_total_medicamento <= 0) {
    alert('Este medicamento no tiene existencias en stock.');
    return;
  }

  // Buscar si ya está en el carrito
  const existing = cartItems.value.find(item => item.id_medicamento === med.id_medicamento);
  if (existing) {
    if (existing.cantidad >= med.existencia_total_medicamento) {
      alert(`No puede vender más de la existencia disponible (${med.existencia_total_medicamento} unidades).`);
      return;
    }
    existing.cantidad++;
  } else {
    // Buscar lote activo relacionado con el medicamento
    const relatedLote = lotesList.value.find(l => l.id_medicamento === med.id_medicamento && l.estado_lote && l.existencia_lote > 0);
    
    cartItems.value.push({
      id_medicamento: med.id_medicamento,
      nombre_medicamento: med.nombre_medicamento,
      precio_venta: Number(med.precio_venta),
      cantidad: 1,
      id_lote: relatedLote ? relatedLote.id_lote : null,
      maxStock: med.existencia_total_medicamento
    });
  }

  searchMedTerm.value = '';
  medSearchResults.value = [];
};

const updateQty = (item, amt) => {
  const newQty = item.cantidad + amt;
  if (newQty <= 0) {
    cartItems.value = cartItems.value.filter(i => i.id_medicamento !== item.id_medicamento);
  } else if (newQty > item.maxStock) {
    alert(`No hay stock suficiente. Máximo disponible: ${item.maxStock}`);
  } else {
    item.cantidad = newQty;
  }
};

const removeItem = (medId) => {
  cartItems.value = cartItems.value.filter(i => i.id_medicamento !== medId);
};

// Cálculos del Carrito
const cartSubtotal = computed(() => {
  return cartItems.value.reduce((acc, curr) => acc + (curr.precio_venta * curr.cantidad), 0);
});

const cartTotal = computed(() => {
  return cartSubtotal.value; // Puedes sumar impuestos aquí si fuera necesario
});

// Crear Cliente Rápido
const createQuickClient = async () => {
  if (!quickClientNombre.value) return;
  try {
    const res = await fetch(`${API_URL}/clientes`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ nombre_cliente: quickClientNombre.value, nit_cliente: quickClientNit.value, estado_cliente: true })
    });
    if (res.ok) {
      const newClient = await res.json();
      clientesList.value.push(newClient);
      selectedClienteId.value = newClient.id_cliente;
      isNewClientModalOpen.value = false;
      addAlert(`Cliente "${quickClientNombre.value}" agregado con éxito.`, 'success');
      quickClientNombre.value = '';
      quickClientNit.value = 'CF';
    }
  } catch (e) {
    console.error('Error al agregar cliente rápido:', e);
  }
};

// Guardar e Imprimir Transacción de Venta
const processSale = async () => {
  if (cartItems.value.length === 0) {
    alert('El carrito está vacío.');
    return;
  }
  if (!selectedClienteId.value) {
    alert('Por favor seleccione un cliente.');
    return;
  }
  if (!selectedMetodoPagoId.value) {
    alert('Por favor seleccione un método de pago.');
    return;
  }

  loading.value = true;
  try {
    // 1. Guardar la cabecera de la Venta
    const salePayload = {
      id_usuario: props.activeUser.id_usuario,
      id_cliente: Number(selectedClienteId.value),
      total_venta: Number(cartTotal.value),
      estado_venta: true
    };

    const saleRes = await fetch(`${API_URL}/ventas`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(salePayload)
    });

    if (!saleRes.ok) {
      const err = await saleRes.json();
      throw new Error(err.message || 'Error al guardar cabecera de venta');
    }

    const savedSale = await saleRes.json();

    // 2. Guardar Detalles de la Venta uno a uno
    const detailsPromises = cartItems.value.map(async (item) => {
      const detailPayload = {
        id_venta: savedSale.id_venta,
        id_medicamento: item.id_medicamento,
        id_lote: item.id_lote || undefined,
        cantidad_detalle_venta: item.cantidad,
        subtotal_detalle_venta: Number(item.precio_venta * item.cantidad),
        estado_detalle_venta: true
      };

      const detRes = await fetch(`${API_URL}/detalles-venta`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(detailPayload)
      });
      return detRes.ok ? detRes.json() : null;
    });

    const savedDetails = await Promise.all(detailsPromises);

    // 3. Registrar el método de pago detalle
    const paymentPayload = {
      id_venta: savedSale.id_venta,
      id_metodo_pago: Number(selectedMetodoPagoId.value),
      cantidad_detalle_metodos_pago: Number(cartTotal.value),
      estado_detalle_metodos_pago: true
    };

    const paymentRes = await fetch(`${API_URL}/detalles-metodos-pago`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(paymentPayload)
    });

    const savedPayment = paymentRes.ok ? await paymentRes.json() : null;

    // Guardar para la vista de Factura
    savedInvoice.value = savedSale;
    invoiceItems.value = cartItems.value;
    invoicePayment.value = metodosPagoList.value.find(m => m.id_metodo_pago === Number(selectedMetodoPagoId.value));
    
    isInvoiceModalOpen.value = true;
    addAlert(`Venta #${savedSale.id_venta} registrada correctamente.`, 'success');
    
    // Limpiar Carrito y recargar catálogos
    cartItems.value = [];
    loadData();

  } catch (error) {
    alert(error.message || 'Error al procesar la venta');
    console.error(error);
  } finally {
    loading.value = false;
  }
};

const getClienteNombre = (id) => {
  const cli = clientesList.value.find(c => c.id_cliente === Number(id));
  return cli ? cli.nombre_cliente : 'Consumidor Final';
};

const getClienteNit = (id) => {
  const cli = clientesList.value.find(c => c.id_cliente === Number(id));
  return cli ? cli.nit_cliente : 'CF';
};

const formatCurrency = (val) => {
  return new Intl.NumberFormat('es-GT', { style: 'currency', currency: 'GTQ' }).format(val);
};

const printInvoice = () => {
  window.print();
};
</script>

<template>
  <div class="pos-container">
    <!-- Mitad Izquierda: Configuración del Carrito -->
    <div class="pos-left">
      
      <!-- Panel de Cliente -->
      <div class="stats-card">
        <h4 class="pos-section-title">Cliente de la Transacción</h4>
        <div style="display: flex; gap: 16px; align-items: flex-end;">
          <div class="form-group" style="flex: 1; margin-bottom: 0;">
            <label class="form-label">Seleccionar Cliente</label>
            <select class="form-control" v-model="selectedClienteId">
              <option v-for="cli in clientesList" :key="cli.id_cliente" :value="cli.id_cliente">
                {{ cli.nombre_cliente }} (NIT: {{ cli.nit_cliente }})
              </option>
            </select>
          </div>
          <button class="btn btn-secondary" @click="isNewClientModalOpen = true">
            + Agregar Cliente
          </button>
        </div>
      </div>

      <!-- Panel de Selección de Medicamentos -->
      <div class="stats-card">
        <h4 class="pos-section-title">Catálogo de Medicamentos Disponibles</h4>
        <div class="form-group" style="margin-bottom: 15px;">
          <input type="text" class="form-control" 
                 v-model="searchMedTerm" 
                 placeholder="Filtrar por nombre, código o componente activo...">
        </div>
        
        <!-- Grid de Medicamentos en Stock -->
        <div class="pos-med-grid">
          <div v-for="med in filteredMedicamentos" 
               :key="med.id_medicamento" 
               class="pos-med-card" 
               @click="addToCart(med)"
               :title="med.existencia_total_medicamento === 0 ? 'Sin existencias' : 'Haga clic para agregar al carrito'">
            <div>
              <div class="pos-med-name">{{ med.nombre_medicamento }}</div>
              <div class="pos-med-component">{{ med.componente_activo || 'Fórmula Genérica' }}</div>
            </div>
            <div style="display: flex; justify-content: space-between; align-items: center; margin-top: 12px;">
              <span class="pos-med-price">{{ formatCurrency(med.precio_venta) }}</span>
              <span class="pos-med-stock" 
                    :class="med.existencia_total_medicamento === 0 ? 'out-of-stock' : (med.existencia_total_medicamento < 10 ? 'low-stock' : 'in-stock')">
                Existencia: {{ med.existencia_total_medicamento }}
              </span>
            </div>
          </div>
          <div v-if="filteredMedicamentos.length === 0" style="grid-column: 1 / -1; text-align: center; color: var(--text-muted); padding: 30px;">
            No se encontraron medicamentos activos que coincidan con la búsqueda.
          </div>
        </div>
      </div>

      <!-- Detalle de Productos -->
      <div class="table-responsive">
        <div style="padding: 16px 20px; font-weight: 700; border-bottom: 1px solid var(--border-color); font-size: 15px;">
          Medicamentos Seleccionados
        </div>
        <table class="table">
          <thead>
            <tr>
              <th>Artículo</th>
              <th>Lote Relacionado</th>
              <th style="width: 150px; text-align: center;">Cantidad</th>
              <th>Precio Unitario</th>
              <th>Subtotal</th>
              <th style="text-align: right;">Acciones</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="item in cartItems" :key="item.id_medicamento">
              <td style="font-weight: 600;">{{ item.nombre_medicamento }}</td>
              <td>
                <code>{{ item.id_lote ? '#L-' + item.id_lote : 'Auto-asignar' }}</code>
              </td>
              <td>
                <div class="pos-qty-actions">
                  <button class="pos-qty-btn" @click="updateQty(item, -1)">-</button>
                  <input type="text" class="pos-qty-input" :value="item.cantidad" readonly>
                  <button class="pos-qty-btn" @click="updateQty(item, 1)">+</button>
                </div>
              </td>
              <td style="font-weight: 600;">{{ formatCurrency(item.precio_venta) }}</td>
              <td style="font-weight: 700;">{{ formatCurrency(item.precio_venta * item.cantidad) }}</td>
              <td style="text-align: right;">
                <button class="btn btn-danger btn-sm" @click="removeItem(item.id_medicamento)" title="Eliminar">
                  &times;
                </button>
              </td>
            </tr>
            <tr v-if="cartItems.length === 0">
              <td colspan="6" style="text-align: center; padding: 40px; color: var(--text-muted);">
                El carrito de ventas está vacío. Busque medicamentos arriba para agregarlos.
              </td>
            </tr>
          </tbody>
        </table>
      </div>

    </div>

    <!-- Mitad Derecha: Resumen de Caja -->
    <div class="pos-right">
      <h4 class="pos-section-title">Resumen de Caja</h4>
      
      <div class="form-group">
        <label class="form-label">Método de Pago</label>
        <select class="form-control" v-model="selectedMetodoPagoId">
          <option v-for="met in metodosPagoList" :key="met.id_metodo_pago" :value="met.id_metodo_pago">
            {{ met.nombre_metodo_pago }} {{ met.cuenta_metodo_pago ? '(' + met.cuenta_metodo_pago + ')' : '' }}
          </option>
        </select>
      </div>

      <div class="pos-summary">
        <div class="pos-summary-row">
          <span>Subtotal</span>
          <span>{{ formatCurrency(cartSubtotal) }}</span>
        </div>
        <div class="pos-summary-row">
          <span>Impuestos (IVA Incluido)</span>
          <span>Q 0.00</span>
        </div>
        <div class="pos-summary-row total">
          <span>Total a Pagar</span>
          <span>{{ formatCurrency(cartTotal) }}</span>
        </div>
      </div>

      <button class="btn btn-success" style="width:100%; margin-top: 24px; font-size:16px; padding: 12px;" 
              :disabled="cartItems.length === 0 || loading" 
              @click="processSale">
        {{ loading ? 'Registrando...' : 'Cobrar y Crear Factura' }}
      </button>
    </div>

    <!-- Modal Nuevo Cliente -->
    <div class="modal-overlay" v-if="isNewClientModalOpen">
      <div class="modal-content">
        <div class="modal-header">
          <h3 class="modal-title">Agregar Cliente Rápido</h3>
          <button class="close-btn" @click="isNewClientModalOpen = false">&times;</button>
        </div>
        <div class="modal-body">
          <div class="form-group">
            <label class="form-label">Nombre del Cliente *</label>
            <input type="text" class="form-control" v-model="quickClientNombre" placeholder="Ej. Ana Gómez" required>
          </div>
          <div class="form-group">
            <label class="form-label">NIT del Cliente</label>
            <input type="text" class="form-control" v-model="quickClientNit" placeholder="Ej. 123456-7 o CF">
          </div>
        </div>
        <div class="modal-footer">
          <button class="btn btn-secondary" @click="isNewClientModalOpen = false">Cancelar</button>
          <button class="btn btn-primary" @click="createQuickClient">Guardar</button>
        </div>
      </div>
    </div>

    <!-- Modal Factura Preview Estilo DREZOC -->
    <div class="modal-overlay" v-if="isInvoiceModalOpen">
      <div class="modal-content large">
        <div class="modal-header">
          <h3 class="modal-title">Factura Generada</h3>
          <button class="close-btn" @click="isInvoiceModalOpen = false">&times;</button>
        </div>
        <div class="modal-body" style="background-color: var(--bg-primary);">
          
          <!-- Contenido Imprimible de la Factura -->
          <div class="invoice-wrapper" id="printable-invoice">
            <div class="invoice-header">
              <div>
                <div class="invoice-brand">
                  <svg style="width:24px;height:24px;fill:var(--primary-color);" viewBox="0 0 24 24">
                    <path d="M12 17.27L18.18 21l-1.64-7.03L22 9.24l-7.19-.61L12 2 9.19 8.63 2 9.24l5.46 4.73L5.82 21z"/>
                  </svg>
                  Farmacia Evichs
                </div>
                <div style="font-size: 13px; color: var(--text-muted); margin-top: 5px;">Farmacia Profesional Integrada</div>
              </div>
              <div>
                <div class="invoice-title">Factura</div>
                <div style="text-align:right; font-size:12px; color:var(--text-muted); margin-top: 4px;">Original del Cliente</div>
              </div>
            </div>

            <!-- Datos de Facturación -->
            <div class="invoice-details-grid">
              <div class="invoice-detail-block">
                <h4>Detalles del Cliente</h4>
                <p style="font-weight:700; color:var(--text-primary);">{{ getClienteNombre(savedInvoice?.id_cliente) }}</p>
                <p>NIT: {{ getClienteNit(savedInvoice?.id_cliente) }}</p>
                <p>Dirección: Ciudad de Guatemala</p>
              </div>
              <div class="invoice-detail-block">
                <ul class="invoice-meta-list">
                  <li>
                    <span>Fecha Emisión:</span>
                    <span>{{ savedInvoice ? new Date(savedInvoice.fecha_venta).toLocaleDateString() : '' }}</span>
                  </li>
                  <li>
                    <span>Estado del Pedido:</span>
                    <span><span class="invoice-badge paid">Pagado</span></span>
                  </li>
                  <li>
                    <span>Número de Pedido:</span>
                    <span>0000{{ savedInvoice?.id_venta }}</span>
                  </li>
                  <li>
                    <span>Método de Pago:</span>
                    <span>{{ invoicePayment?.nombre_metodo_pago || 'Efectivo' }}</span>
                  </li>
                </ul>
              </div>
            </div>

            <!-- Tabla de Ítems -->
            <table class="table" style="margin-top: 20px;">
              <thead>
                <tr>
                  <th style="width: 50px;">#</th>
                  <th>Artículo</th>
                  <th style="text-align: center;">Cantidad</th>
                  <th style="text-align: right;">Tarifa / Precio</th>
                  <th style="text-align: right;">Total</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="(item, index) in invoiceItems" :key="item.id_medicamento">
                  <td>{{ index + 1 }}</td>
                  <td>
                    <p style="font-weight: 600; margin: 0;">{{ item.nombre_medicamento }}</p>
                    <span style="font-size: 11px; color: var(--text-muted)">Lote: {{ item.id_lote ? '#L-' + item.id_lote : 'Genérico' }}</span>
                  </td>
                  <td style="text-align: center;">{{ item.cantidad }}</td>
                  <td style="text-align: right;">{{ formatCurrency(item.precio_venta) }}</td>
                  <td style="text-align: right; font-weight:700;">{{ formatCurrency(item.precio_venta * item.cantidad) }}</td>
                </tr>
              </tbody>
            </table>

            <!-- Resumen Inferior -->
            <div class="invoice-summary">
              <div class="invoice-summary-block">
                <div class="invoice-summary-row">
                  <span>Subtotal parcial:</span>
                  <span>{{ formatCurrency(savedInvoice?.total_venta) }}</span>
                </div>
                <div class="invoice-summary-row">
                  <span>Impuesto (IVA 0%):</span>
                  <span>Q 0.00</span>
                </div>
                <div class="invoice-summary-row total">
                  <span>Total Facturado:</span>
                  <span>{{ formatCurrency(savedInvoice?.total_venta) }}</span>
                </div>
              </div>
            </div>
          </div>

        </div>
        <div class="modal-footer">
          <button class="btn btn-secondary" @click="isInvoiceModalOpen = false">Cerrar</button>
          <button class="btn btn-primary" @click="printInvoice">Imprimir Factura</button>
        </div>
      </div>
    </div>

    </div>
  </template>

  <style scoped>
  .pos-med-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
    gap: 16px;
    margin-top: 20px;
    max-height: 380px;
    overflow-y: auto;
    padding-right: 8px;
  }

  .pos-med-card {
    background-color: var(--bg-primary);
    border: 1px solid var(--border-color);
    border-radius: 10px;
    padding: 16px;
    cursor: pointer;
    transition: all 0.25s cubic-bezier(0.4, 0, 0.2, 1);
    display: flex;
    flex-direction: column;
    justify-content: space-between;
    box-shadow: var(--shadow-sm);
    min-height: 120px;
    user-select: none;
  }

  .pos-med-card:hover {
     border-color: var(--primary-color);
     transform: translateY(-2px);
     box-shadow: var(--shadow-md);
     background-color: rgba(59, 130, 246, 0.03);
  }

  .pos-med-name {
    font-size: 14px;
    font-weight: 600;
    color: var(--text-primary);
    line-height: 1.4;
  }

  .pos-med-component {
    font-size: 11px;
    color: var(--text-muted);
    margin-top: 4px;
    font-style: italic;
  }

  .pos-med-price {
    font-size: 15px;
    font-weight: 700;
    color: var(--success-color);
  }

  .pos-med-stock {
    font-size: 11px;
    font-weight: 600;
    padding: 2px 6px;
    border-radius: 4px;
  }

  .pos-med-stock.in-stock {
    background-color: rgba(16, 185, 129, 0.1);
    color: var(--success-color);
  }

  .pos-med-stock.low-stock {
    background-color: rgba(245, 158, 11, 0.1);
    color: var(--warning-color);
  }

  .pos-med-stock.out-of-stock {
    background-color: rgba(239, 68, 68, 0.1);
    color: var(--danger-color);
  }
  </style>
