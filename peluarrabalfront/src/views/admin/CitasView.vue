<script setup lang="ts">
import { useCitasStore } from "@/stores/citas";
import { useUiStore } from "@/stores/ui";
import { onMounted, ref, computed } from "vue";

const store = useCitasStore();
const ui = useUiStore();

const dialog = ref(false);
const editando = ref(false);
const search = ref("");
const filtroEstado = ref("todos");

const nuevoCita = ref({
  id_Cliente: "",
  id_Servicio: "",
  id_Empleado: "",
  fecha: "",
  hora_Inicio: "",
  observaciones: "",
});

const citaEditar = ref<any>(null);

onMounted(() => {
  store.fetchCitas();
});

const listaFiltrada = computed(() => {
  let lista = store.lista;
  if (filtroEstado.value !== "todos") {
    lista = lista.filter((c: any) => c.estado === filtroEstado.value);
  }
  if (search.value) {
    const q = search.value.toLowerCase();
    lista = lista.filter(
      (c: any) =>
        c.cliente?.nombre?.toLowerCase().includes(q) ||
        c.servicio?.nombre?.toLowerCase().includes(q) ||
        c.empleado?.especialidad?.toLowerCase().includes(q) ||
        c.fecha?.includes(q)
    );
  }
  return lista;
});

const totalPorEstado = computed(() => ({
  total: store.lista.length,
  confirmadas: store.lista.filter((c: any) => c.estado === "confirmada").length,
  pendientes: store.lista.filter((c: any) => c.estado === "pendiente").length,
  canceladas: store.lista.filter((c: any) => c.estado === "cancelada").length,
}));

async function guardarCita() {
  try {
    await store.createCita(nuevoCita.value);
    ui.notify("Cita creada correctamente", "success");
    dialog.value = false;
    nuevoCita.value = { id_Cliente: "", id_Servicio: "", id_Empleado: "", fecha: "", hora_Inicio: "", observaciones: "" };
  } catch {
    ui.notify("Error al crear la cita", "error");
  }
}

function abrirEditar(cita: any) {
  citaEditar.value = {
    id_Reserva: cita.id_Reserva,
    id_Cliente: cita.cliente?.id_Usuario,
    id_Servicio: cita.servicio?.id_Servicio,
    id_Empleado: cita.empleado?.id_Empleado,
    fecha: cita.fecha,
    estado: cita.estado,
    hora_Inicio: cita.hora_Inicio,
    observaciones: cita.observaciones,
  };
  editando.value = true;
}

async function guardarEdicion() {
  const id = citaEditar.value.id_Reserva;
  const payload = {
    id_Cliente: citaEditar.value.id_Cliente,
    id_Servicio: citaEditar.value.id_Servicio,
    id_Empleado: citaEditar.value.id_Empleado,
    fecha: citaEditar.value.fecha,
    estado: citaEditar.value.estado,
    hora_Inicio: citaEditar.value.hora_Inicio,
    observaciones: citaEditar.value.observaciones,
  };
  await store.updateCita(id, payload);
  ui.notify("Cita actualizada", "success");
  editando.value = false;
}

async function cancelar(id: number) {
  await store.cancelarCita(id);
  ui.notify("Cita cancelada", "success");
}

function estadoColor(estado: string) {
  if (estado === "pendiente") return "warning";
  if (estado === "cancelada") return "error";
  if (estado === "confirmada") return "primary";
  return "success";
}

function estadoIcon(estado: string) {
  if (estado === "pendiente") return "mdi-clock-outline";
  if (estado === "cancelada") return "mdi-close-circle-outline";
  if (estado === "confirmada") return "mdi-check-circle-outline";
  return "mdi-check-all";
}

function formatFecha(fecha: string) {
  if (!fecha) return "-";
  const [y, m, d] = fecha.split("-");
  return `${d}/${m}/${y}`;
}
</script>

<template>
  <div class="admin-page">
    <!-- ─── HEADER ─── -->
    <div class="page-header">
      <div class="header-left">
        <h1 class="page-title">Gestión de Citas</h1>
        <p class="page-sub">Visualiza, edita y gestiona todas las reservas</p>
      </div>
      <v-btn
        color="primary"
        rounded="lg"
        elevation="0"
        prepend-icon="mdi-plus"
        class="new-btn"
        @click="dialog = true"
      >
        Nueva Cita
      </v-btn>
    </div>

    <!-- ─── STATS ─── -->
    <div class="stats-row">
      <div class="stat-chip" @click="filtroEstado = 'todos'" :class="{ 'stat-chip--active': filtroEstado === 'todos' }">
        <v-icon size="17" color="#6B9292">mdi-calendar-month</v-icon>
        <span class="stat-num">{{ totalPorEstado.total }}</span>
        <span class="stat-lbl">Total</span>
      </div>
      <div class="stat-chip" @click="filtroEstado = 'confirmada'" :class="{ 'stat-chip--active': filtroEstado === 'confirmada' }">
        <v-icon size="17" color="#6B9292">mdi-check-circle-outline</v-icon>
        <span class="stat-num stat-num--teal">{{ totalPorEstado.confirmadas }}</span>
        <span class="stat-lbl">Confirmadas</span>
      </div>
      <div class="stat-chip" @click="filtroEstado = 'pendiente'" :class="{ 'stat-chip--active': filtroEstado === 'pendiente' }">
        <v-icon size="17" color="#C8A96E">mdi-clock-outline</v-icon>
        <span class="stat-num stat-num--gold">{{ totalPorEstado.pendientes }}</span>
        <span class="stat-lbl">Pendientes</span>
      </div>
      <div class="stat-chip" @click="filtroEstado = 'cancelada'" :class="{ 'stat-chip--active': filtroEstado === 'cancelada' }">
        <v-icon size="17" color="#C97878">mdi-close-circle-outline</v-icon>
        <span class="stat-num stat-num--red">{{ totalPorEstado.canceladas }}</span>
        <span class="stat-lbl">Canceladas</span>
      </div>
    </div>

    <!-- ─── TABLE CARD ─── -->
    <div class="table-card">
      <!-- Toolbar -->
      <div class="table-toolbar">
        <v-text-field
          v-model="search"
          placeholder="Buscar por cliente, servicio, empleado o fecha..."
          prepend-inner-icon="mdi-magnify"
          variant="outlined"
          rounded="lg"
          density="compact"
          hide-details
          color="primary"
          class="search-field"
          clearable
        />
        <v-chip
          v-if="filtroEstado !== 'todos'"
          closable
          size="small"
          :color="estadoColor(filtroEstado)"
          variant="tonal"
          @click:close="filtroEstado = 'todos'"
        >
          {{ filtroEstado }}
        </v-chip>
      </div>

      <v-divider style="border-color: rgba(200,169,110,0.07)" />

      <!-- Table -->
      <v-data-table
        :items="listaFiltrada"
        :loading="store.loading"
        item-key="id_Reserva"
        class="admin-data-table"
        :items-per-page="10"
      >
        <template #headers>
          <tr>
            <th class="th">Fecha</th>
            <th class="th">Hora</th>
            <th class="th">Cliente</th>
            <th class="th">Servicio</th>
            <th class="th">Empleado</th>
            <th class="th">Estado</th>
            <th class="th" style="text-align:right">Acciones</th>
          </tr>
        </template>

        <template #item="{ item }">
          <tr class="data-row">
            <td class="td">
              <div class="date-cell">
                <v-icon size="14" color="rgba(107,146,146,0.5)">mdi-calendar</v-icon>
                <span class="date-text">{{ formatFecha(item.fecha) }}</span>
              </div>
            </td>
            <td class="td">
              <div class="date-cell">
                <v-icon size="14" color="rgba(107,146,146,0.5)">mdi-clock-outline</v-icon>
                <span>{{ item.hora_Inicio?.slice(0, 5) }}</span>
              </div>
            </td>
            <td class="td">
              <div class="client-cell">
                <div class="mini-avatar">{{ item.cliente?.nombre?.[0] ?? '?' }}</div>
                <span class="client-name">{{ item.cliente?.nombre }}</span>
              </div>
            </td>
            <td class="td">
              <span class="service-badge">{{ item.servicio?.nombre }}</span>
            </td>
            <td class="td td-soft">{{ item.empleado?.especialidad }}</td>
            <td class="td">
              <v-chip
                :color="estadoColor(item.estado)"
                size="small"
                variant="tonal"
                :prepend-icon="estadoIcon(item.estado)"
              >
                {{ item.estado }}
              </v-chip>
            </td>
            <td class="td" style="text-align:right">
              <div class="action-btns">
                <v-btn
                  size="small"
                  variant="tonal"
                  color="warning"
                  rounded="lg"
                  prepend-icon="mdi-pencil"
                  class="action-btn"
                  @click="abrirEditar(item)"
                >
                  Editar
                </v-btn>
                <v-btn
                  size="small"
                  variant="tonal"
                  color="error"
                  rounded="lg"
                  icon="mdi-cancel"
                  class="action-btn-icon"
                  @click="cancelar(item.id_Reserva)"
                />
              </div>
            </td>
          </tr>
        </template>

        <template #no-data>
          <div class="empty-state">
            <v-icon size="48" color="rgba(242,227,188,0.15)">mdi-calendar-search</v-icon>
            <p>No se encontraron citas</p>
          </div>
        </template>
      </v-data-table>
    </div>

    <!-- ─── DIALOG: Nueva cita ─── -->
    <v-dialog v-model="dialog" width="500">
      <v-card class="admin-dialog" rounded="xl" elevation="0">
        <div class="dialog-header">
          <div class="dialog-header-left">
            <div class="dialog-icon-wrap">
              <v-icon size="20" color="primary">mdi-calendar-plus</v-icon>
            </div>
            <h3 class="dialog-title">Nueva cita</h3>
          </div>
          <v-btn icon variant="text" size="small" @click="dialog = false">
            <v-icon>mdi-close</v-icon>
          </v-btn>
        </div>
        <v-divider style="border-color: rgba(200,169,110,0.08)" />
        <div class="dialog-body">
          <div class="form-row">
            <div class="form-col">
              <label class="field-label">ID Cliente</label>
              <v-text-field v-model="nuevoCita.id_Cliente" variant="outlined" rounded="lg" density="comfortable" hide-details="auto" color="primary" />
            </div>
            <div class="form-col">
              <label class="field-label">ID Servicio</label>
              <v-text-field v-model="nuevoCita.id_Servicio" variant="outlined" rounded="lg" density="comfortable" hide-details="auto" color="primary" />
            </div>
          </div>
          <div class="form-row mt-3">
            <div class="form-col">
              <label class="field-label">ID Empleado</label>
              <v-text-field v-model="nuevoCita.id_Empleado" variant="outlined" rounded="lg" density="comfortable" hide-details="auto" color="primary" />
            </div>
            <div class="form-col">
              <label class="field-label">Hora</label>
              <v-text-field v-model="nuevoCita.hora_Inicio" type="time" variant="outlined" rounded="lg" density="comfortable" hide-details="auto" color="primary" />
            </div>
          </div>
          <label class="field-label mt-3">Fecha</label>
          <v-text-field v-model="nuevoCita.fecha" type="date" variant="outlined" rounded="lg" density="comfortable" hide-details="auto" color="primary" class="mb-3" />
          <label class="field-label">Observaciones</label>
          <v-text-field v-model="nuevoCita.observaciones" variant="outlined" rounded="lg" density="comfortable" hide-details="auto" color="primary" />
        </div>
        <div class="dialog-actions">
          <v-btn variant="text" rounded="lg" @click="dialog = false">Cancelar</v-btn>
          <v-btn color="primary" rounded="lg" elevation="0" @click="guardarCita">Guardar cita</v-btn>
        </div>
      </v-card>
    </v-dialog>

    <!-- ─── DIALOG: Editar cita ─── -->
    <v-dialog v-model="editando" width="500">
      <v-card v-if="citaEditar" class="admin-dialog" rounded="xl" elevation="0">
        <div class="dialog-header">
          <div class="dialog-header-left">
            <div class="dialog-icon-wrap" style="background: rgba(200,169,110,0.12); border-color: rgba(200,169,110,0.2)">
              <v-icon size="20" color="warning">mdi-calendar-edit</v-icon>
            </div>
            <h3 class="dialog-title">Editar cita</h3>
          </div>
          <v-btn icon variant="text" size="small" @click="editando = false">
            <v-icon>mdi-close</v-icon>
          </v-btn>
        </div>
        <v-divider style="border-color: rgba(200,169,110,0.08)" />
        <div class="dialog-body">
          <div class="form-row">
            <div class="form-col">
              <label class="field-label">ID Cliente</label>
              <v-text-field v-model="citaEditar.id_Cliente" variant="outlined" rounded="lg" density="comfortable" hide-details="auto" color="primary" />
            </div>
            <div class="form-col">
              <label class="field-label">ID Servicio</label>
              <v-text-field v-model="citaEditar.id_Servicio" variant="outlined" rounded="lg" density="comfortable" hide-details="auto" color="primary" />
            </div>
          </div>
          <div class="form-row mt-3">
            <div class="form-col">
              <label class="field-label">ID Empleado</label>
              <v-text-field v-model="citaEditar.id_Empleado" variant="outlined" rounded="lg" density="comfortable" hide-details="auto" color="primary" />
            </div>
            <div class="form-col">
              <label class="field-label">Hora</label>
              <v-text-field v-model="citaEditar.hora_Inicio" type="time" variant="outlined" rounded="lg" density="comfortable" hide-details="auto" color="primary" />
            </div>
          </div>
          <div class="form-row mt-3">
            <div class="form-col">
              <label class="field-label">Fecha</label>
              <v-text-field v-model="citaEditar.fecha" type="date" variant="outlined" rounded="lg" density="comfortable" hide-details="auto" color="primary" />
            </div>
            <div class="form-col">
              <label class="field-label">Estado</label>
              <v-select
                v-model="citaEditar.estado"
                :items="['pendiente', 'confirmada', 'cancelada', 'completada']"
                variant="outlined"
                rounded="lg"
                density="comfortable"
                hide-details="auto"
                color="primary"
              />
            </div>
          </div>
          <label class="field-label mt-3">Observaciones</label>
          <v-text-field v-model="citaEditar.observaciones" variant="outlined" rounded="lg" density="comfortable" hide-details="auto" color="primary" />
        </div>
        <div class="dialog-actions">
          <v-btn variant="text" rounded="lg" @click="editando = false">Cancelar</v-btn>
          <v-btn color="warning" rounded="lg" elevation="0" @click="guardarEdicion">Guardar cambios</v-btn>
        </div>
      </v-card>
    </v-dialog>
  </div>
</template>

<style scoped>
.admin-page {
  padding: 32px 36px;
  width: 100%;
}

/* ─── HEADER ─── */
.page-header {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  margin-bottom: 24px;
  flex-wrap: wrap;
  gap: 16px;
}

.header-left {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.page-title {
  font-size: 1.55rem;
  font-weight: 700;
  color: #F2E3BC;
  line-height: 1.2;
}

.page-sub {
  font-size: 0.84rem;
  color: rgba(242, 227, 188, 0.4);
}

.new-btn {
  font-weight: 600 !important;
  letter-spacing: 0.03em !important;
}

/* ─── STATS ─── */
.stats-row {
  display: flex;
  gap: 12px;
  margin-bottom: 20px;
  flex-wrap: wrap;
}

.stat-chip {
  display: flex;
  align-items: center;
  gap: 8px;
  background: rgba(200, 169, 110, 0.05);
  border: 1px solid rgba(200, 169, 110, 0.10);
  border-radius: 10px;
  padding: 10px 18px;
  cursor: pointer;
  transition: all 0.18s;
  user-select: none;
}

.stat-chip:hover {
  background: rgba(200, 169, 110, 0.09);
  border-color: rgba(200, 169, 110, 0.22);
}

.stat-chip--active {
  background: rgba(107, 146, 146, 0.12) !important;
  border-color: rgba(107, 146, 146, 0.30) !important;
}

.stat-num {
  font-size: 1.15rem;
  font-weight: 700;
  color: #F2E3BC;
}

.stat-num--teal  { color: #6B9292; }
.stat-num--gold  { color: #C8A96E; }
.stat-num--red   { color: #C97878; }

.stat-lbl {
  font-size: 0.78rem;
  color: rgba(242, 227, 188, 0.4);
}

/* ─── TABLE CARD ─── */
.table-card {
  background: #1e2217;
  border: 1px solid rgba(200, 169, 110, 0.09);
  border-radius: 18px;
  overflow: hidden;
}

.table-toolbar {
  padding: 16px 20px;
  display: flex;
  align-items: center;
  gap: 12px;
}

.search-field {
  max-width: 460px;
}

.admin-data-table {
  background: transparent !important;
}

.th {
  font-size: 0.71rem !important;
  font-weight: 700 !important;
  text-transform: uppercase !important;
  letter-spacing: 0.09em !important;
  color: rgba(242, 227, 188, 0.30) !important;
  padding: 14px 20px !important;
  border-bottom: 1px solid rgba(200, 169, 110, 0.07) !important;
  background: rgba(0, 0, 0, 0.12) !important;
}

.data-row {
  transition: background 0.15s;
}

.data-row:hover {
  background: rgba(107, 146, 146, 0.06) !important;
}

.td {
  font-size: 0.875rem !important;
  color: rgba(242, 227, 188, 0.65) !important;
  padding: 15px 20px !important;
  border-bottom: 1px solid rgba(200, 169, 110, 0.04) !important;
  vertical-align: middle !important;
}

.td-soft {
  color: rgba(242, 227, 188, 0.45) !important;
  font-size: 0.82rem !important;
}

/* ─── CELLS ─── */
.date-cell {
  display: flex;
  align-items: center;
  gap: 6px;
}

.date-text {
  font-weight: 500;
  color: #F2E3BC;
}

.client-cell {
  display: flex;
  align-items: center;
  gap: 10px;
}

.mini-avatar {
  width: 30px;
  height: 30px;
  border-radius: 50%;
  background: rgba(107, 146, 146, 0.12);
  border: 1px solid rgba(107, 146, 146, 0.20);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.72rem;
  font-weight: 700;
  color: #6B9292;
  flex-shrink: 0;
  text-transform: uppercase;
}

.client-name {
  font-weight: 500;
  color: #F2E3BC;
}

.service-badge {
  background: rgba(200, 169, 110, 0.07);
  border: 1px solid rgba(200, 169, 110, 0.12);
  border-radius: 6px;
  padding: 3px 10px;
  font-size: 0.78rem;
  color: rgba(200, 169, 110, 0.85);
  white-space: nowrap;
}

/* ─── ACTIONS ─── */
.action-btns {
  display: flex;
  gap: 8px;
  justify-content: flex-end;
  align-items: center;
}

.action-btn {
  font-size: 0.78rem !important;
  font-weight: 500 !important;
}

/* ─── EMPTY STATE ─── */
.empty-state {
  padding: 60px 20px;
  text-align: center;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 14px;
  color: rgba(242, 227, 188, 0.28);
  font-size: 0.9rem;
}

/* ─── DIALOG ─── */
.admin-dialog {
  background: #1e2217 !important;
  border: 1px solid rgba(200, 169, 110, 0.10) !important;
}

.dialog-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 20px 24px;
}

.dialog-header-left {
  display: flex;
  align-items: center;
  gap: 12px;
}

.dialog-icon-wrap {
  width: 38px;
  height: 38px;
  border-radius: 10px;
  background: rgba(107, 146, 146, 0.12);
  border: 1px solid rgba(107, 146, 146, 0.20);
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.dialog-title {
  font-size: 1.05rem;
  font-weight: 700;
  color: #F2E3BC;
}

.dialog-body {
  padding: 20px 24px;
}

.form-row {
  display: flex;
  gap: 12px;
}

.form-col {
  flex: 1;
  min-width: 0;
}

.field-label {
  display: block;
  font-size: 0.78rem;
  font-weight: 500;
  color: rgba(242, 227, 188, 0.45);
  margin-bottom: 6px;
  letter-spacing: 0.02em;
}

.mt-3 {
  margin-top: 12px;
}

.mb-3 {
  margin-bottom: 12px;
}

.dialog-actions {
  padding: 16px 24px 22px;
  display: flex;
  justify-content: flex-end;
  gap: 8px;
}
</style>
