<script setup lang="ts">
import { ref, computed, watch, nextTick } from 'vue';
import { Check, Repeat, GripVertical, Plus, Trash2, Pencil } from 'lucide-vue-next';

export type RecurringRule = 'none' | 'daily' | 'weekly' | 'workdays';

export interface Task {
  id: string;
  title: string;
  completed: boolean;
  isRecurring: boolean;
  recurringRule?: RecurringRule;
  createdAtKey?: string;
  seriesId?: string;
}

const props = defineProps<{
  dayName: string;
  dateStr: string;
  dateKey: string;
  tasks: Task[];
  isOpenInitial?: boolean;
}>();

const emit = defineEmits<{
  (e: 'update:tasks', tasks: Task[]): void;
  (e: 'request-delete', payload: { task: Task; dateKey: string }): void;
}>();

const isOpen = ref(props.isOpenInitial ?? false);
const showAddInput = ref(false);
const newTaskTitle = ref('');

// Control del menú desplegable de repetición
const activeRepeatMenuId = ref<string | null>(null);

// Estado de Edición
const editingTaskId = ref<string | null>(null);
const editingTitle = ref('');
const editInputRef = ref<HTMLInputElement[] | null>(null);

// Drag and Drop
const draggedIndex = ref<number | null>(null);

const completedCount = computed(() => props.tasks.filter(t => t.completed).length);

const toggleCard = () => {
  isOpen.value = !isOpen.value;
};

const toggleTask = (id: string) => {
  const updated = props.tasks.map(t => 
    t.id === id ? { ...t, completed: !t.completed } : t
  );
  emit('update:tasks', updated);
};

const setRecurringRule = (task: Task, rule: RecurringRule) => {
  const isRecurring = rule !== 'none';
  const updated = props.tasks.map(t => 
    t.id === task.id ? { 
      ...t, 
      isRecurring, 
      recurringRule: rule,
      createdAtKey: t.createdAtKey || props.dateKey
    } : t
  );
  emit('update:tasks', updated);
  activeRepeatMenuId.value = null;
};

const toggleRepeatMenu = (id: string) => {
  activeRepeatMenuId.value = activeRepeatMenuId.value === id ? null : id;
};

const handleDeleteRequest = (task: Task) => {
  emit('request-delete', { task, dateKey: props.dateKey });
};

const startEditing = (task: Task) => {
  editingTaskId.value = task.id;
  editingTitle.value = task.title;
  nextTick(() => {
    if (editInputRef.value && editInputRef.value[0]) {
      editInputRef.value[0].focus();
    }
  });
};

const saveEditing = (id: string) => {
  if (editingTaskId.value === null) return;
  
  const trimmed = editingTitle.value.trim();
  if (trimmed) {
    const updated = props.tasks.map(t => 
      t.id === id ? { ...t, title: trimmed } : t
    );
    emit('update:tasks', updated);
  } else {
    const task = props.tasks.find(t => t.id === id);
    if (task) handleDeleteRequest(task);
  }
  editingTaskId.value = null;
};

const cancelEditing = () => {
  editingTaskId.value = null;
};

const handleAddTask = () => {
  const title = newTaskTitle.value.trim();
  if (title) {
    const newTask: Task = {
      id: Date.now().toString(),
      title,
      completed: false,
      isRecurring: false,
      recurringRule: 'none',
      createdAtKey: props.dateKey,
    };
    emit('update:tasks', [...props.tasks, newTask]);
    newTaskTitle.value = '';
    showAddInput.value = false;
  }
};

const onDragStart = (index: number, event: DragEvent) => {
  draggedIndex.value = index;
  if (event.dataTransfer) {
    event.dataTransfer.effectAllowed = 'move';
  }
};

const onDragOver = (event: DragEvent) => {
  event.preventDefault();
};

const onDrop = (targetIndex: number) => {
  if (draggedIndex.value === null || draggedIndex.value === targetIndex) return;

  const currentTasks = [...props.tasks];
  const [movedTask] = currentTasks.splice(draggedIndex.value, 1);
  currentTasks.splice(targetIndex, 0, movedTask);

  emit('update:tasks', currentTasks);
  draggedIndex.value = null;
};

const onDragEnd = () => {
  draggedIndex.value = null;
};

watch(() => props.isOpenInitial, (newVal) => {
  isOpen.value = newVal ?? false;
});
</script>

<template>
  <div class="flex-1 flex flex-col justify-center border-b border-gray-100 dark:border-zinc-800/80 bg-white dark:bg-[#18181b] transition-colors duration-300">
    <!-- Header del Día -->
    <div 
      @click="toggleCard"
      class="flex items-center justify-between px-6 py-4.5 cursor-pointer select-none hover:bg-gray-50/60 dark:hover:bg-zinc-800/40 transition"
    >
      <h2 class="text-sm font-semibold tracking-wide text-gray-800 dark:text-zinc-200 uppercase font-sans">
        {{ dayName }}
      </h2>
      <span class="text-sm font-semibold tracking-wide text-gray-400 dark:text-zinc-400">
        {{ completedCount }}/{{ tasks.length }}
      </span>
    </div>

    <!-- Contenido desplegable -->
    <div v-if="isOpen" class="px-6 pb-5 space-y-3 animate-fade-in relative bg-gray-50/40 dark:bg-[#121214]">
      <div class="flex justify-between items-center text-xs text-gray-400 dark:text-zinc-500 font-medium pt-2">
        <span>{{ dateStr }}</span>
        <button 
          @click.stop="showAddInput = !showAddInput" 
          class="text-gray-800 dark:text-zinc-200 hover:text-[#FF6B00] dark:hover:text-[#FF6B00] font-semibold text-xs tracking-wide transition flex items-center space-x-1"
        >
          <span>{{ showAddInput ? 'Cancelar' : 'Add +' }}</span>
        </button>
      </div>

      <!-- Input para agregar tarea -->
      <div v-if="showAddInput" class="flex items-center space-x-2 pt-1 animate-fade-in">
        <input 
          v-model="newTaskTitle"
          @keyup.enter="handleAddTask"
          type="text" 
          placeholder="Escribe una tarea..." 
          class="flex-1 border border-gray-200 dark:border-zinc-700 bg-white dark:bg-zinc-800 text-gray-900 dark:text-zinc-100 rounded-lg px-3 py-1.5 text-xs focus:outline-none focus:border-[#FF6B00] dark:focus:border-[#FF6B00] transition"
          autoFocus
        />
        <button 
          @click="handleAddTask"
          class="bg-gray-900 dark:bg-zinc-100 text-white dark:text-zinc-900 p-1.5 rounded-lg hover:bg-[#FF6B00] dark:hover:bg-[#FF6B00] dark:hover:text-white transition"
        >
          <Plus :size="14" />
        </button>
      </div>

      <!-- Lista de tareas -->
      <div class="space-y-1.5 pt-1">
        <p v-if="tasks.length === 0" class="text-xs text-gray-400 dark:text-zinc-500 italic py-2">
          Sin tareas para esta fecha.
        </p>

        <div 
          v-for="(task, index) in tasks" 
          :key="task.id"
          :draggable="editingTaskId !== task.id"
          @dragstart="onDragStart(index, $event)"
          @dragover="onDragOver($event)"
          @drop="onDrop(index)"
          @dragend="onDragEnd"
          :class="[
            draggedIndex === index ? 'opacity-40 bg-gray-100 dark:bg-zinc-800' : 'bg-white dark:bg-[#18181b]',
            'group relative flex items-center justify-between text-xs py-2 px-2 rounded-lg border border-transparent hover:border-gray-200 dark:hover:border-zinc-700 transition-all select-none'
          ]"
        >
          <div class="flex items-center space-x-3 flex-1 min-w-0 pr-2">
            <button 
              @click="toggleTask(task.id)"
              :class="task.completed ? 'bg-[#FF6B00] border-[#FF6B00]' : 'border-gray-300 dark:border-zinc-600 bg-white dark:bg-zinc-800 hover:border-gray-400 dark:hover:border-zinc-500'"
              class="w-4 h-4 border rounded-[3px] flex items-center justify-center transition shrink-0"
            >
              <Check v-if="task.completed" :size="11" class="text-white stroke-[3.5]" />
            </button>

            <input 
              v-if="editingTaskId === task.id"
              ref="editInputRef"
              v-model="editingTitle"
              @keyup.enter="saveEditing(task.id)"
              @keyup.esc="cancelEditing"
              @blur="saveEditing(task.id)"
              type="text"
              class="flex-1 bg-transparent border-b border-[#FF6B00] text-gray-900 dark:text-zinc-100 font-medium text-xs focus:outline-none px-0 py-0"
            />

            <span 
              v-else
              @dblclick="startEditing(task)"
              :class="task.completed ? 'line-through text-gray-300 dark:text-zinc-600' : 'text-gray-800 dark:text-zinc-200 font-medium'"
              class="transition-all truncate cursor-pointer flex-1"
              title="Doble clic para editar"
            >
              {{ task.title }}
            </span>
          </div>

          <div class="flex items-center space-x-1 shrink-0 relative">
            <button 
              @click="startEditing(task)"
              title="Editar tarea"
              class="p-1 text-gray-300 dark:text-zinc-600 hover:text-[#FF6B00] dark:hover:text-[#FF6B00] transition opacity-80 group-hover:opacity-100"
            >
              <Pencil :size="13" stroke-width="2" />
            </button>

            <div class="relative">
              <button 
                @click="toggleRepeatMenu(task.id)"
                title="Configurar repetición"
                :class="task.isRecurring ? 'text-[#FF6B00]' : 'text-gray-300 dark:text-zinc-600 hover:text-[#FF6B00] dark:hover:text-[#FF6B00]'"
                class="p-1 transition flex items-center space-x-0.5"
              >
                <Repeat :size="13" stroke-width="2.2" />
              </button>

              <div 
                v-if="activeRepeatMenuId === task.id" 
                @click="activeRepeatMenuId = null" 
                class="fixed inset-0 z-30" 
              />

              <div 
                v-if="activeRepeatMenuId === task.id"
                :class="[
                  index >= 2 && index >= tasks.length - 2 ? 'bottom-full mb-1' : 'top-7',
                  'absolute right-0 z-40 w-40 bg-white dark:bg-zinc-800 border border-gray-200 dark:border-zinc-700 rounded-xl shadow-xl py-1 text-xs text-gray-800 dark:text-zinc-200 animate-fade-in'
                ]"
              >
                <button 
                  @click="setRecurringRule(task, 'none')"
                  :class="task.recurringRule === 'none' || !task.recurringRule ? 'text-[#FF6B00] font-semibold bg-orange-50/60 dark:bg-orange-950/40' : 'text-gray-600 dark:text-zinc-300'"
                  class="w-full text-left px-3 py-2 hover:bg-gray-50 dark:hover:bg-zinc-700/60 transition"
                >
                  No repetir
                </button>
                <button 
                  @click="setRecurringRule(task, 'daily')"
                  :class="task.recurringRule === 'daily' ? 'text-[#FF6B00] font-semibold bg-orange-50/60 dark:bg-orange-950/40' : 'text-gray-600 dark:text-zinc-300'"
                  class="w-full text-left px-3 py-2 hover:bg-gray-50 dark:hover:bg-zinc-700/60 transition"
                >
                  Diariamente
                </button>
                <button 
                  @click="setRecurringRule(task, 'weekly')"
                  :class="task.recurringRule === 'weekly' ? 'text-[#FF6B00] font-semibold bg-orange-50/60 dark:bg-orange-950/40' : 'text-gray-600 dark:text-zinc-300'"
                  class="w-full text-left px-3 py-2 hover:bg-gray-50 dark:hover:bg-zinc-700/60 transition"
                >
                  Semanalmente
                </button>
                <button 
                  @click="setRecurringRule(task, 'workdays')"
                  :class="task.recurringRule === 'workdays' ? 'text-[#FF6B00] font-semibold bg-orange-50/60 dark:bg-orange-950/40' : 'text-gray-600 dark:text-zinc-300'"
                  class="w-full text-left px-3 py-2 hover:bg-gray-50 dark:hover:bg-zinc-700/60 transition"
                >
                  Días laborables (L-V)
                </button>
              </div>
            </div>
            
            <button 
              @click="handleDeleteRequest(task)"
              title="Eliminar tarea"
              class="p-1 text-gray-300 dark:text-zinc-600 hover:text-[#FF6B00] dark:hover:text-[#FF6B00] transition opacity-80 group-hover:opacity-100"
            >
              <Trash2 :size="13" stroke-width="2" />
            </button>

            <div 
              title="Arrastrar para reordenar"
              class="cursor-grab active:cursor-grabbing p-1 text-gray-300 dark:text-zinc-600 hover:text-gray-600 dark:hover:text-zinc-400 transition"
            >
              <GripVertical :size="14" />
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.custom-scrollbar::-webkit-scrollbar {
  width: 4px;
}
.custom-scrollbar::-webkit-scrollbar-track {
  background: transparent;
}
.custom-scrollbar::-webkit-scrollbar-thumb {
  background: #e5e7eb;
  border-radius: 4px;
}
.custom-scrollbar::-webkit-scrollbar-thumb:hover {
  background: #FF6B00;
}
</style>