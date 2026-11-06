<script setup lang="ts">
import { ref, computed, watch, onMounted, onUnmounted } from "vue";
import { Menu, ChevronLeft, ChevronRight } from "lucide-vue-next";

const props = defineProps<{
  selectedDate?: Date;
  startOfWeekPreference?: "MONDAY" | "SUNDAY";
}>();

const emit = defineEmits<{
  (e: "openSettings"): void;
  (e: "update:selectedDate", date: Date): void;
}>();

const showPicker = ref(false);
const pickerContainer = ref<HTMLElement | null>(null);

// Sincronización de fecha con App.vue
const selectedDate = ref(props.selectedDate || new Date());
const viewDate = ref(new Date(selectedDate.value));

const todayNumber = computed(() => new Date().getDate());

watch(
  () => props.selectedDate,
  (newVal) => {
    if (newVal) {
      selectedDate.value = new Date(newVal);
      viewDate.value = new Date(newVal);
    }
  },
);

// Emisión de cambios al componente padre
const updateDate = (newDate: Date) => {
  selectedDate.value = newDate;
  emit("update:selectedDate", newDate);
};

const selectDate = (date: Date) => {
  updateDate(date);
  showPicker.value = false;
};

const selectToday = () => {
  const now = new Date();
  updateDate(now);
  viewDate.value = now;
  showPicker.value = false;
};

const togglePicker = () => {
  showPicker.value = !showPicker.value;
  if (showPicker.value) {
    viewDate.value = new Date(selectedDate.value);
  }
};

const prevWeek = () => {
  const newDate = new Date(selectedDate.value);
  newDate.setDate(newDate.getDate() - 7);
  updateDate(newDate);
};

const nextWeek = () => {
  const newDate = new Date(selectedDate.value);
  newDate.setDate(newDate.getDate() + 7);
  updateDate(newDate);
};

// Lógica de semanas y calendario
const getStartOfWeek = (d: Date, startOnSunday: boolean) => {
  const date = new Date(d);
  const day = date.getDay();
  const diff = date.getDate() - day + (startOnSunday ? 0 : day === 0 ? -6 : 1);
  return new Date(date.setDate(diff));
};

const weekInfo = computed(() => {
  const startOnSunday = props.startOfWeekPreference === "SUNDAY";
  const start = getStartOfWeek(selectedDate.value, startOnSunday);
  const end = new Date(start);
  end.setDate(start.getDate() + 6);
  const monthStr = start
    .toLocaleString("en-US", { month: "short" })
    .toUpperCase();
  return { label: `${monthStr} / ${start.getDate()} - ${end.getDate()}` };
});

const currentMonthName = computed(() =>
  viewDate.value.toLocaleString("en-US", { month: "short" }).toUpperCase(),
);
const currentYear = computed(() => viewDate.value.getFullYear());

const changeMonth = (delta: number) => {
  const d = new Date(viewDate.value);
  d.setMonth(d.getMonth() + delta);
  viewDate.value = d;
};

const changeYear = (delta: number) => {
  const d = new Date(viewDate.value);
  d.setFullYear(d.getFullYear() + delta);
  viewDate.value = d;
};

const calendarDays = computed(() => {
  const year = viewDate.value.getFullYear();
  const month = viewDate.value.getMonth();
  const firstDay = new Date(year, month, 1);
  const lastDay = new Date(year, month + 1, 0);
  const days: (Date | null)[] = [];

  const startDayOfWeek =
    props.startOfWeekPreference === "SUNDAY"
      ? firstDay.getDay()
      : firstDay.getDay() === 0
        ? 6
        : firstDay.getDay() - 1;

  for (let i = 0; i < startDayOfWeek; i++) days.push(null);
  for (let i = 1; i <= lastDay.getDate(); i++)
    days.push(new Date(year, month, i));
  return days;
});

const handleClickOutside = (e: MouseEvent) => {
  if (
    pickerContainer.value &&
    !pickerContainer.value.contains(e.target as Node)
  ) {
    showPicker.value = false;
  }
};

onMounted(() => document.addEventListener("click", handleClickOutside));
onUnmounted(() => document.removeEventListener("click", handleClickOutside));
</script>

<template>
  <header class="flex items-center justify-between px-6 py-4 border-b border-gray-100 dark:border-zinc-800 bg-white dark:bg-[#18181b] transition-colors duration-300">
    <!-- Botón Menú / Ajustes -->
    <button
      @click="emit('openSettings')"
      class="text-gray-800 dark:text-zinc-200 hover:opacity-75 transition p-1 -ml-1 focus:outline-none"
      title="Configuración"
    >
      <Menu :size="22" stroke-width="1.8" />
    </button>

    <!-- Navegador de Semana -->
    <div
      class="flex items-center space-x-2 text-xs font-semibold tracking-widest text-gray-700 dark:text-zinc-300"
    >
      <button
        @click="prevWeek"
        class="hover:text-black dark:hover:text-white px-1 py-0.5 transition font-bold text-sm"
        title="Semana anterior"
      >
        &lt;
      </button>
      <span class="cursor-default select-none">{{ weekInfo.label }}</span>
      <button
        @click="nextWeek"
        class="hover:text-black dark:hover:text-white px-1 py-0.5 transition font-bold text-sm"
        title="Semana siguiente"
      >
        &gt;
      </button>
    </div>

    <!-- Botón Calendario -->
    <div ref="pickerContainer" class="relative">
      <button
        @click="togglePicker"
        title="Abrir calendario"
        class="p-1 -mr-1 hover:scale-105 active:scale-95 transition focus:outline-none"
      >
        <div
          class="border-[1.5px] border-gray-800 dark:border-zinc-200 rounded-[4px] w-5 h-5 flex items-center justify-center text-[10px] font-extrabold text-gray-800 dark:text-zinc-200 leading-none bg-white dark:bg-[#18181b] shadow-2xs"
        >
          {{ todayNumber }}
        </div>
      </button>

      <!-- Date Picker Popover -->
      <div
        v-if="showPicker"
        class="absolute right-0 mt-3 w-64 bg-white dark:bg-[#18181b] border border-gray-100 dark:border-zinc-700 shadow-xl rounded-2xl p-4 z-50 font-sans text-xs animate-fade-in text-gray-800 dark:text-zinc-100"
      >
        <!-- Control Mes / Año -->
        <div class="flex items-center justify-between mb-3">
          <div class="flex items-center space-x-1">
            <button
              @click="changeMonth(-1)"
              class="p-1 hover:bg-gray-100 dark:hover:bg-zinc-800 rounded-md transition"
            >
              <ChevronLeft :size="14" />
            </button>
            <span class="font-bold tracking-wider w-10 text-center">{{
              currentMonthName
            }}</span>
            <button
              @click="changeMonth(1)"
              class="p-1 hover:bg-gray-100 dark:hover:bg-zinc-800 rounded-md transition"
            >
              <ChevronRight :size="14" />
            </button>
          </div>

          <div class="flex items-center space-x-1">
            <button
              @click="changeYear(-1)"
              class="p-1 hover:bg-gray-100 dark:hover:bg-zinc-800 rounded-md transition"
            >
              <ChevronLeft :size="14" />
            </button>
            <span class="font-bold tracking-wider">{{ currentYear }}</span>
            <button
              @click="changeYear(1)"
              class="p-1 hover:bg-gray-100 dark:hover:bg-zinc-800 rounded-md transition"
            >
              <ChevronRight :size="14" />
            </button>
          </div>
        </div>

        <!-- Encabezado de Días -->
        <div
          class="grid grid-cols-7 text-center font-bold text-[10px] text-gray-400 dark:text-zinc-500 mb-2"
        >
          <span
            v-for="d in startOfWeekPreference === 'SUNDAY'
              ? ['S', 'M', 'T', 'W', 'T', 'F', 'S']
              : ['M', 'T', 'W', 'T', 'F', 'S', 'S']"
            :key="d"
          >
            {{ d }}
          </span>
        </div>

        <!-- Grilla de Días -->
        <div class="grid grid-cols-7 gap-1 text-center mb-3">
          <template v-for="(day, index) in calendarDays" :key="index">
            <div v-if="!day" class="h-7"></div>
            <button
              v-else
              @click="selectDate(day)"
              :class="[
                day.toDateString() === selectedDate.toDateString()
                  ? 'bg-[#FF6B00] text-white font-bold'
                  : 'text-gray-800 dark:text-zinc-200 hover:bg-gray-100 dark:hover:bg-zinc-800',
                day.toDateString() === new Date().toDateString() &&
                day.toDateString() !== selectedDate.toDateString()
                  ? 'border border-[#FF6B00] text-[#FF6B00] font-bold'
                  : '',
              ]"
              class="h-7 w-7 mx-auto flex items-center justify-center rounded-lg text-xs transition"
            >
              {{ day.getDate() }}
            </button>
          </template>
        </div>

        <!-- Footer con Acción Directa: Ir a HOY -->
        <div
          class="pt-2 border-t border-gray-100 dark:border-zinc-800 flex justify-between items-center text-[10px]"
        >
          <span class="text-gray-400 dark:text-zinc-500">Ir a fecha actual:</span>
          <button
            @click="selectToday"
            class="font-bold text-[#FF6B00] hover:underline uppercase tracking-wider px-2 py-0.5 rounded transition"
          >
            HOY
          </button>
        </div>
      </div>
    </div>
  </header>
</template>

<style scoped>
.animate-fade-in {
  animation: fadeIn 0.15s ease-out forwards;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(-4px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
</style>