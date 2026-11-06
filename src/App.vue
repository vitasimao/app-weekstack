<script setup lang="ts">
import { ref, computed, onMounted, watch } from 'vue';
import SplashScreen from './components/SplashScreen.vue';
import HeaderNav from './components/HeaderNav.vue';
import DayCard, { type Task } from './components/DayCard.vue';
import SettingsModal from './components/SettingsModal.vue';

// Tipo extendido para soportar fechas excluidas
export interface ExtendedTask extends Task {
  excludedDates?: string[];
}

const isLoading = ref(true);
const selectedDate = ref(new Date());
const isSettingsOpen = ref(false);

// Preferencias configurables
const startOfWeek = ref<'MONDAY' | 'SUNDAY'>('MONDAY');
const autoCleanCompleted = ref(false);
const darkMode = ref(false);

const allTasksByDate = ref<Record<string, ExtendedTask[]>>({});

// Modal de eliminación
const showDeleteModal = ref(false);
const taskToDelete = ref<ExtendedTask | null>(null);
const taskToDeleteDateKey = ref<string | null>(null);

onMounted(() => {
  const savedTasks = localStorage.getItem('app_tasks_by_date');
  if (savedTasks) {
    try {
      allTasksByDate.value = JSON.parse(savedTasks);
    } catch (e) {
      console.error('Error al cargar tareas', e);
    }
  }

  const savedSettings = localStorage.getItem('app_settings');
  if (savedSettings) {
    try {
      const parsed = JSON.parse(savedSettings);
      startOfWeek.value = parsed.startOfWeek || 'MONDAY';
      autoCleanCompleted.value = parsed.autoCleanCompleted ?? false;
      darkMode.value = parsed.darkMode ?? false;
    } catch (e) {
      console.error('Error cargando preferencias:', e);
    }
  }

  setTimeout(() => {
    isLoading.value = false;
  }, 1800);
});

watch(
  allTasksByDate,
  (newTasks) => {
    localStorage.setItem('app_tasks_by_date', JSON.stringify(newTasks));
  },
  { deep: true }
);

watch([startOfWeek, autoCleanCompleted, darkMode], () => {
  localStorage.setItem('app_settings', JSON.stringify({
    startOfWeek: startOfWeek.value,
    autoCleanCompleted: autoCleanCompleted.value,
    darkMode: darkMode.value
  }));
}, { deep: true });

const handleOpenSettings = () => {
  isSettingsOpen.value = true;
};

// Lógica de Auto-Clean
const handleUpdateTasks = (dateKey: string, newTasks: ExtendedTask[]) => {
  const tasksToSave = autoCleanCompleted.value 
    ? newTasks.filter(t => !t.completed) 
    : newTasks;

  allTasksByDate.value = {
    ...allTasksByDate.value,
    [dateKey]: tasksToSave,
  };
};

watch(autoCleanCompleted, (isEnabled) => {
  if (isEnabled) {
    const cleanedTasks: Record<string, ExtendedTask[]> = {};
    Object.entries(allTasksByDate.value).forEach(([dKey, tasks]) => {
      cleanedTasks[dKey] = tasks.filter(t => !t.completed);
    });
    allTasksByDate.value = cleanedTasks;
  }
});

// Gestión de eliminación recurrente
const handleRequestDelete = ({ task, dateKey }: { task: ExtendedTask; dateKey: string }) => {
  if (task.isRecurring || (task.recurringRule && task.recurringRule !== 'none')) {
    taskToDelete.value = task;
    taskToDeleteDateKey.value = dateKey;
    showDeleteModal.value = true;
  } else {
    deleteSingleTask(task.id, dateKey);
  }
};

const deleteSingleTask = (taskId: string, dateKey: string) => {
  const currentTasks = allTasksByDate.value[dateKey] || [];
  const updatedTasks = currentTasks.filter(t => t.id !== taskId);
  handleUpdateTasks(dateKey, updatedTasks);
};

const closeDeleteModal = () => {
  showDeleteModal.value = false;
  taskToDelete.value = null;
  taskToDeleteDateKey.value = null;
};

// CORRECCIÓN AQUI: Marca únicamente la fecha seleccionada como excluida
const confirmDeleteSingle = () => {
  if (taskToDelete.value && taskToDeleteDateKey.value) {
    const targetDateKey = taskToDeleteDateKey.value;
    const baseTaskId = taskToDelete.value.id.split('_')[0];
    const targetTitle = taskToDelete.value.title;

    // Si la tarea existía de forma explícita en el día objetivo, la quitamos
    deleteSingleTask(taskToDelete.value.id, targetDateKey);

    // Buscamos la tarea original creada y añadimos la fecha excluida
    const updatedAll = { ...allTasksByDate.value };
    let modified = false;

    Object.entries(updatedAll).forEach(([dKey, tasks]) => {
      updatedAll[dKey] = tasks.map(t => {
        const currentBaseId = t.id.split('_')[0];
        if (currentBaseId === baseTaskId || t.title === targetTitle) {
          modified = true;
          const excludedDates = t.excludedDates || [];
          if (!excludedDates.includes(targetDateKey)) {
            return {
              ...t,
              excludedDates: [...excludedDates, targetDateKey]
            };
          }
        }
        return t;
      });
    });

    if (modified) {
      allTasksByDate.value = updatedAll;
    }
  }
  closeDeleteModal();
};

const confirmDeleteSeries = () => {
  if (!taskToDelete.value) return;

  const targetTitle = taskToDelete.value.title;
  const baseId = taskToDelete.value.id.split('_')[0];
  const updatedAll: Record<string, ExtendedTask[]> = {};

  Object.entries(allTasksByDate.value).forEach(([dKey, tasks]) => {
    updatedAll[dKey] = tasks.filter(t => {
      const currentBaseId = t.id.split('_')[0];
      return currentBaseId !== baseId && t.title !== targetTitle;
    });
  });

  allTasksByDate.value = updatedAll;
  closeDeleteModal();
};

const getTasksForDate = (targetDateKey: string, dayOfWeekIndex: number): ExtendedTask[] => {
  const explicitTasks = allTasksByDate.value[targetDateKey] || [];
  const projectedTasks: ExtendedTask[] = [...explicitTasks];

  const targetDate = new Date(targetDateKey + 'T00:00:00');

  Object.entries(allTasksByDate.value).forEach(([sourceDateKey, tasks]) => {
    if (sourceDateKey >= targetDateKey) return;

    const sourceDate = new Date(sourceDateKey + 'T00:00:00');

    tasks.forEach(task => {
      if (!task.isRecurring || !task.recurringRule || task.recurringRule === 'none') return;

      // CORRECCIÓN AQUÍ: Ignorar la proyección si esta fecha fue excluida
      if (task.excludedDates && task.excludedDates.includes(targetDateKey)) return;

      const baseTaskId = task.id.split('_')[0];

      const alreadyExists = projectedTasks.some(t => {
        const currentBaseId = t.id.split('_')[0];
        return currentBaseId === baseTaskId || t.title === task.title;
      });

      if (alreadyExists) return;

      let shouldProject = false;

      if (task.recurringRule === 'daily') {
        shouldProject = true;
      } else if (task.recurringRule === 'weekly') {
        shouldProject = sourceDate.getDay() === targetDate.getDay();
      } else if (task.recurringRule === 'workdays') {
        const isWorkday = dayOfWeekIndex >= 1 && dayOfWeekIndex <= 5;
        shouldProject = isWorkday;
      }

      if (shouldProject) {
        projectedTasks.push({
          ...task,
          id: `${baseTaskId}_${targetDateKey}`,
          completed: false,
        });
      }
    });
  });

  return projectedTasks;
};

const currentWeekDays = computed(() => {
  const date = new Date(selectedDate.value);
  const dayOfWeek = date.getDay();
  const startOnSunday = startOfWeek.value === 'SUNDAY';
  
  const diff = date.getDate() - dayOfWeek + (startOnSunday ? 0 : (dayOfWeek === 0 ? -6 : 1));
  const monday = new Date(date.setDate(diff));

  const days = [];
  const dayNames = ['SUNDAY', 'MONDAY', 'TUESDAY', 'WEDNESDAY', 'THURSDAY', 'FRIDAY', 'SATURDAY'];

  for (let i = 0; i < 7; i++) {
    const d = new Date(monday);
    d.setDate(monday.getDate() + i);

    const year = d.getFullYear();
    const month = String(d.getMonth() + 1).padStart(2, '0');
    const dayNum = String(d.getDate()).padStart(2, '0');
    const dateKey = `${year}-${month}-${dayNum}`;

    const monthName = d.toLocaleString('en-US', { month: 'long' });
    const formattedDateStr = `${monthName} ${d.getDate()}, ${year}`;

    days.push({
      id: dateKey,
      dateKey,
      name: dayNames[d.getDay()],
      dayIndex: d.getDay(),
      dateStr: formattedDateStr,
      isOpen: false,
    });
  }

  return days;
});

watch(
  darkMode,
  (newVal) => {
    if (newVal) {
      document.documentElement.classList.add('dark');
    } else {
      document.documentElement.classList.remove('dark');
    }
  },
  { immediate: true }
);
</script>

<template>
  <div>
    <Transition name="fade">
      <SplashScreen v-if="isLoading" />
    </Transition>

    <div v-if="!isLoading" class="h-screen w-screen flex items-center justify-center p-0 sm:p-6 overflow-hidden">
      
      <!-- CONTENEDOR PRINCIPAL -->
      <main 
        :class="[
          darkMode ? 'dark bg-[#18181b] text-white' : 'bg-white text-gray-900',
          'w-full sm:w-[390px] h-full sm:h-[844px] max-h-full sm:max-h-[90vh] flex flex-col font-sans sm:rounded-[44px] shadow-2xl overflow-hidden border border-gray-800/20 relative transition-colors duration-300'
        ]"
      >
        <HeaderNav 
          :selectedDate="selectedDate"
          :startOfWeekPreference="startOfWeek"
          @update:selectedDate="(newDate) => selectedDate = newDate"
          @openSettings="handleOpenSettings" 
        />

        <div class="flex-1 flex flex-col justify-between overflow-y-auto custom-scrollbar">
          <DayCard 
            v-for="day in currentWeekDays" 
            :key="day.dateKey"
            :dayName="day.name"
            :dateStr="day.dateStr"
            :dateKey="day.dateKey"
            :tasks="getTasksForDate(day.dateKey, day.dayIndex)"
            :isOpenInitial="day.isOpen"
            @update:tasks="(newTasks) => handleUpdateTasks(day.dateKey, newTasks)"
            @request-delete="handleRequestDelete"
          />
        </div>

        <SettingsModal 
          :isOpen="isSettingsOpen"
          v-model:startOfWeek="startOfWeek"
          v-model:autoCleanCompleted="autoCleanCompleted"
          v-model:darkMode="darkMode"
          @close="isSettingsOpen = false"
        />

        <Teleport to="body">
          <div 
            v-if="showDeleteModal" 
            class="fixed inset-0 z-50 flex items-center justify-center bg-black/50 backdrop-blur-xs p-4 animate-scale-in"
            @click.self="closeDeleteModal"
          >
            <div class="bg-white dark:bg-[#18181b] rounded-2xl p-6 max-w-xs w-full shadow-2xl border border-gray-100 dark:border-zinc-700 text-center space-y-4">
              <h3 class="text-base font-bold text-gray-900 dark:text-zinc-100">Eliminar tarea recurrente</h3>
              <p class="text-xs text-gray-500 dark:text-zinc-400 leading-relaxed">
                Esta tarea está configurada para repetirse. ¿Qué deseas eliminar?
              </p>
              <div class="space-y-2 pt-1">
                <button 
                  @click="confirmDeleteSingle"
                  class="w-full bg-gray-100 dark:bg-zinc-800 text-gray-900 dark:text-zinc-200 hover:bg-gray-200 dark:hover:bg-zinc-700 text-xs font-semibold py-2.5 rounded-xl transition cursor-pointer"
                >
                  Solo esta ocurrencia
                </button>
                <button 
                  @click="confirmDeleteSeries"
                  class="w-full bg-[#FF6B00] text-white hover:bg-[#e05e00] text-xs font-semibold py-2.5 rounded-xl transition cursor-pointer"
                >
                  Toda la serie
                </button>
                <button 
                  @click="closeDeleteModal"
                  class="w-full text-gray-400 hover:text-gray-600 dark:hover:text-zinc-300 text-xs font-medium py-1 transition cursor-pointer"
                >
                  Cancelar
                </button>
              </div>
            </div>
          </div>
        </Teleport>
      </main>
    </div>
  </div>
</template>

<style scoped>
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.4s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>