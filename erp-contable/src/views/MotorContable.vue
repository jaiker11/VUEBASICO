<script setup>
import { ref, computed, onMounted } from 'vue';
import { movimientoService } from '../services/erpApi';

// ============ ESTADOS REACTIVOS ============
const movimientos = ref([]);
const cargando = ref(false);
const error = ref(null);
const mensajeExito = ref(null);

const nuevoMovimiento = ref({
  concepto: '',
  tipo: 'Ingreso',
  monto: '',
  fecha: new Date().toISOString().split('T')[0]
});

// Tipos de movimiento disponibles
const tiposMovimiento = ['Ingreso', 'Egreso'];

// ============ PROPIEDADES COMPUTADAS (MOTOR CONTABLE) ============
const totalIngresos = computed(() => {
  if (!movimientos.value || movimientos.value.length === 0) return 0;
  return movimientos.value
    .filter(m => m.tipo === 'Ingreso')
    .reduce((sum, m) => sum + Number(m.monto || 0), 0);
});

const totalEgresos = computed(() => {
  if (!movimientos.value || movimientos.value.length === 0) return 0;
  return movimientos.value
    .filter(m => m.tipo === 'Egreso')
    .reduce((sum, m) => sum + Number(m.monto || 0), 0);
});

const saldo = computed(() => {
  return (totalIngresos.value || 0) - (totalEgresos.value || 0);
});

// ============ MÉTODOS ASÍNCRONOS ============
async function cargarMovimientos() {
  cargando.value = true;
  error.value = null;

  try {
    const respuesta = await movimientoService.getAll();
    movimientos.value = respuesta.data.datos || [];
  } catch (err) {
    console.error('Error al cargar movimientos:', err);
    error.value = err.response?.data?.mensaje || 'No se pudo conectar con el servidor Express (http://localhost:3000). Verifica que esté encendido.';
  } finally {
    cargando.value = false;
  }
}

async function guardarMovimiento() {
  if (!nuevoMovimiento.value.concepto || !nuevoMovimiento.value.monto) {
    error.value = 'Por favor completa todos los campos requeridos.';
    return;
  }

  cargando.value = true;
  error.value = null;
  mensajeExito.value = null;

  try {
    const respuesta = await movimientoService.create({
      concepto: nuevoMovimiento.value.concepto,
      tipo: nuevoMovimiento.value.tipo,
      monto: parseFloat(nuevoMovimiento.value.monto),
      fecha: nuevoMovimiento.value.fecha
    });

    if (respuesta.data?.datos) {
      movimientos.value.push(respuesta.data.datos);
    }
    
    // Limpiar formulario tras éxito
    nuevoMovimiento.value = {
      concepto: '',
      tipo: 'Ingreso',
      monto: '',
      fecha: new Date().toISOString().split('T')[0]
    };

    mensajeExito.value = '¡Movimiento registrado con éxito en el servidor!';
    setTimeout(() => {
      mensajeExito.value = null;
    }, 4000);
  } catch (err) {
    console.error('Error al guardar movimiento:', err);
    error.value = err.response?.data?.mensaje || 'Error al guardar el movimiento. Verifica los datos o la conexión.';
  } finally {
    cargando.value = false;
  }
}

// Hook de ciclo de vida para carga inicial
onMounted(() => {
  cargarMovimientos();
});
</script>

<template>
  <div>
    <!-- ENCABEZADO -->
    <div class="d-flex align-center justify-space-between mb-6">
      <div>
        <h1 class="text-h4 font-weight-bold">Motor Contable (Semana 8)</h1>
        <p class="text-subtitle-1 text-grey">Integración asíncrona Vue 3 + Axios + Express</p>
      </div>
      <v-btn
        color="primary"
        prepend-icon="mdi-refresh"
        :loading="cargando"
        @click="cargarMovimientos"
      >
        Recargar Datos
      </v-btn>
    </div>

    <!-- MENSAJES DE ALERTA (ERROR Y ÉXITO) -->
    <v-alert
      v-if="error"
      type="error"
      variant="tonal"
      closable
      class="mb-4"
      @click:close="error = null"
    >
      <v-icon start>mdi-alert-circle</v-icon>
      <strong>Error:</strong> {{ error }}
    </v-alert>

    <v-alert
      v-if="mensajeExito"
      type="success"
      variant="tonal"
      closable
      class="mb-4"
      @click:close="mensajeExito = null"
    >
      <v-icon start>mdi-check-circle</v-icon>
      {{ mensajeExito }}
    </v-alert>

    <!-- TARJETAS DE RESUMEN (KPIS REACTIVOS CON COMPUTED) -->
    <v-row class="mb-6">
      <v-col cols="12" sm="4">
        <v-card elevation="2" class="rounded-lg">
          <v-card-text class="d-flex align-center">
            <v-avatar color="green-lighten-4" size="56" class="mr-4">
              <v-icon color="green-darken-2" size="32">mdi-arrow-up-bold-circle</v-icon>
            </v-avatar>
            <div>
              <div class="text-caption text-grey font-weight-bold">TOTAL INGRESOS</div>
              <div class="text-h5 font-weight-bold text-green-darken-2">
                ${{ totalIngresos.toLocaleString('es-MX', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}
              </div>
            </div>
          </v-card-text>
        </v-card>
      </v-col>

      <v-col cols="12" sm="4">
        <v-card elevation="2" class="rounded-lg">
          <v-card-text class="d-flex align-center">
            <v-avatar color="red-lighten-4" size="56" class="mr-4">
              <v-icon color="red-darken-2" size="32">mdi-arrow-down-bold-circle</v-icon>
            </v-avatar>
            <div>
              <div class="text-caption text-grey font-weight-bold">TOTAL EGRESOS</div>
              <div class="text-h5 font-weight-bold text-red-darken-2">
                ${{ totalEgresos.toLocaleString('es-MX', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}
              </div>
            </div>
          </v-card-text>
        </v-card>
      </v-col>

      <v-col cols="12" sm="4">
        <v-card elevation="2" class="rounded-lg">
          <v-card-text class="d-flex align-center">
            <v-avatar :color="saldo >= 0 ? 'blue-lighten-4' : 'orange-lighten-4'" size="56" class="mr-4">
              <v-icon :color="saldo >= 0 ? 'blue-darken-2' : 'orange-darken-2'" size="32">mdi-scale-balance</v-icon>
            </v-avatar>
            <div>
              <div class="text-caption text-grey font-weight-bold">SALDO NETO</div>
              <div class="text-h5 font-weight-bold" :class="saldo >= 0 ? 'text-blue-darken-2' : 'text-orange-darken-2'">
                ${{ saldo.toLocaleString('es-MX', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}
              </div>
            </div>
          </v-card-text>
        </v-card>
      </v-col>
    </v-row>

    <!-- SECCIÓN PRINCIPAL: FORMULARIO Y TABLA -->
    <v-row>
      <!-- FORMULARIO DE REGISTRO -->
      <v-col cols="12" md="4">
        <v-card elevation="2" class="rounded-lg pa-4">
          <v-card-title class="px-0 pt-0 text-h6 font-weight-bold">
            <v-icon start color="primary">mdi-plus-circle</v-icon>
            Nuevo Movimiento
          </v-card-title>
          <v-divider class="mb-4" />

          <form @submit.prevent="guardarMovimiento">
            <v-text-field
              v-model="nuevoMovimiento.concepto"
              label="Concepto"
              placeholder="Ej: Venta de software"
              prepend-inner-icon="mdi-format-text"
              variant="outlined"
              density="comfortable"
              class="mb-2"
              required
            />

            <v-select
              v-model="nuevoMovimiento.tipo"
              :items="tiposMovimiento"
              label="Tipo de Movimiento"
              prepend-inner-icon="mdi-swap-horizontal"
              variant="outlined"
              density="comfortable"
              class="mb-2"
            />

            <v-text-field
              v-model="nuevoMovimiento.monto"
              label="Monto ($)"
              type="number"
              step="0.01"
              placeholder="0.00"
              prepend-inner-icon="mdi-currency-usd"
              variant="outlined"
              density="comfortable"
              class="mb-2"
              required
            />

            <v-text-field
              v-model="nuevoMovimiento.fecha"
              label="Fecha"
              type="date"
              prepend-inner-icon="mdi-calendar"
              variant="outlined"
              density="comfortable"
              class="mb-4"
            />

            <v-btn
              type="submit"
              color="primary"
              block
              size="large"
              :loading="cargando"
              prepend-icon="mdi-content-save"
            >
              Registrar Movimiento
            </v-btn>
          </form>
        </v-card>
      </v-col>

      <!-- TABLA DE MOVIMIENTOS -->
      <v-col cols="12" md="8">
        <v-card elevation="2" class="rounded-lg pa-4">
          <div class="d-flex align-center justify-space-between mb-2">
            <v-card-title class="px-0 pt-0 text-h6 font-weight-bold">
              <v-icon start color="primary">mdi-clipboard-list</v-icon>
              Historial de Movimientos
            </v-card-title>
            <v-chip color="primary" variant="outlined" size="small">
              {{ movimientos.length }} registros
            </v-chip>
          </div>
          <v-divider class="mb-4" />

          <!-- SPINNER DE CARGA -->
          <div v-if="cargando && movimientos.length === 0" class="text-center pa-8">
            <v-progress-circular indeterminate color="primary" size="64" />
            <div class="mt-4 text-grey">Cargando movimientos desde Express API...</div>
          </div>

          <!-- TABLA -->
          <v-table v-else hover density="comfortable">
            <thead>
              <tr>
                <th class="text-left font-weight-bold">ID</th>
                <th class="text-left font-weight-bold">Fecha</th>
                <th class="text-left font-weight-bold">Concepto</th>
                <th class="text-left font-weight-bold">Tipo</th>
                <th class="text-right font-weight-bold">Monto</th>
              </tr>
            </thead>
            <tbody>
              <tr v-if="movimientos.length === 0">
                <td colspan="5" class="text-center text-grey py-6">
                  No hay movimientos registrados. ¡Crea uno usando el formulario!
                </td>
              </tr>
              <tr v-for="m in movimientos" :key="m.id">
                <td class="text-caption text-grey">{{ m.id }}</td>
                <td>{{ m.fecha || 'N/A' }}</td>
                <td class="font-weight-medium">{{ m.concepto }}</td>
                <td>
                  <v-chip
                    :color="m.tipo === 'Ingreso' ? 'success' : 'error'"
                    size="small"
                    variant="tonal"
                  >
                    <v-icon start size="small">
                      {{ m.tipo === 'Ingreso' ? 'mdi-arrow-up' : 'mdi-arrow-down' }}
                    </v-icon>
                    {{ m.tipo }}
                  </v-chip>
                </td>
                <td
                  class="text-right font-weight-bold"
                  :class="m.tipo === 'Ingreso' ? 'text-success' : 'text-error'"
                >
                  {{ m.tipo === 'Ingreso' ? '+' : '-' }}${{ Number(m.monto).toLocaleString('es-MX', { minimumFractionDigits: 2 }) }}
                </td>
              </tr>
            </tbody>
          </v-table>
        </v-card>
      </v-col>
    </v-row>
  </div>
</template>
