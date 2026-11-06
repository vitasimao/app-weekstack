<script setup lang="ts">
import { X } from 'lucide-vue-next';

const props = defineProps<{
  isOpen: boolean;
  startOfWeek: 'MONDAY' | 'SUNDAY';
  autoCleanCompleted: boolean;
  darkMode: boolean;
}>();

const emit = defineEmits<{
  (e: 'close'): void;
  (e: 'update:startOfWeek', value: 'MONDAY' | 'SUNDAY'): void;
  (e: 'update:autoCleanCompleted', value: boolean): void;
  (e: 'update:darkMode', value: boolean): void;
}>();
</script>

<template>
  <Transition name="slide">
    <div 
      v-if="isOpen" 
      class="absolute inset-0 z-50 bg-white dark:bg-[#18181b] text-gray-800 dark:text-zinc-100 flex flex-col font-sans select-none transition-colors duration-300"
    >
      <header class="flex items-center justify-between px-6 py-5 border-b border-gray-100 dark:border-zinc-800">
        <button 
          @click="emit('close')" 
          class="text-gray-800 dark:text-zinc-200 hover:opacity-70 transition p-1 -ml-1 focus:outline-none"
        >
          <X :size="20" stroke-width="1.8" />
        </button>
        <h2 class="text-xs font-bold tracking-widest uppercase text-gray-800 dark:text-zinc-100">
          Settings
        </h2>
        <div class="w-5"></div>
      </header>

      <div class="flex-1 overflow-y-auto px-6 py-4 divide-y divide-gray-100 dark:divide-zinc-800">
        
        <div class="py-4">
          <span class="block text-[9px] font-bold tracking-wider text-gray-400 dark:text-zinc-500 uppercase mb-2">
            Preferences
          </span>
          <div class="flex items-center justify-between">
            <span class="text-xs font-medium tracking-wide uppercase">
              Start of the Week
            </span>
            <div class="bg-gray-200 dark:bg-zinc-800 p-0.5 rounded-full flex items-center text-[10px] font-bold tracking-wider">
              <button 
                @click="emit('update:startOfWeek', 'MONDAY')"
                :class="[
                  startOfWeek === 'MONDAY' 
                    ? 'bg-[#111827] dark:bg-zinc-100 text-white dark:text-[#18181b] shadow-xs' 
                    : 'text-gray-500 dark:text-zinc-400 hover:text-gray-800 dark:hover:text-white'
                ]"
                class="px-3 py-1 rounded-full transition-all duration-200"
              >
                MONDAY
              </button>
              <button 
                @click="emit('update:startOfWeek', 'SUNDAY')"
                :class="[
                  startOfWeek === 'SUNDAY' 
                    ? 'bg-[#111827] dark:bg-zinc-100 text-white dark:text-[#18181b] shadow-xs' 
                    : 'text-gray-500 dark:text-zinc-400 hover:text-gray-800 dark:hover:text-white'
                ]"
                class="px-3 py-1 rounded-full transition-all duration-200"
              >
                SUNDAY
              </button>
            </div>
          </div>
        </div>

        <div class="py-4">
          <span class="block text-[9px] font-bold tracking-wider text-gray-400 dark:text-zinc-500 uppercase mb-2">
            Task Management
          </span>
          <div class="flex items-center justify-between">
            <span class="text-xs font-medium tracking-wide uppercase">
              Auto-Clean Completed Tasks
            </span>
            <button 
              @click="emit('update:autoCleanCompleted', !autoCleanCompleted)"
              :class="autoCleanCompleted ? 'bg-[#FF6B00]' : 'bg-gray-300 dark:bg-zinc-700'"
              class="relative inline-flex h-5 w-9 items-center rounded-full transition-colors duration-200 focus:outline-none"
            >
              <span 
                :class="autoCleanCompleted ? 'translate-x-4' : 'translate-x-1'"
                class="inline-block h-3.5 w-3.5 transform rounded-full bg-white transition-transform duration-200 shadow-xs"
              />
            </button>
          </div>
        </div>

        <div class="py-4">
          <span class="block text-[9px] font-bold tracking-wider text-gray-400 dark:text-zinc-500 uppercase mb-2">
            Appearance
          </span>
          <div class="flex items-center justify-between">
            <span class="text-xs font-medium tracking-wide uppercase">
              Dark Mode
            </span>
            <button 
              @click="emit('update:darkMode', !darkMode)"
              :class="darkMode ? 'bg-[#FF6B00]' : 'bg-gray-300 dark:bg-zinc-700'"
              class="relative inline-flex h-5 w-9 items-center rounded-full transition-colors duration-200 focus:outline-none"
            >
              <span 
                :class="darkMode ? 'translate-x-4' : 'translate-x-1'"
                class="inline-block h-3.5 w-3.5 transform rounded-full bg-white transition-transform duration-200 shadow-xs"
              />
            </button>
          </div>
        </div>

        <div class="py-4">
          <span class="block text-[9px] font-bold tracking-wider text-gray-400 dark:text-zinc-500 uppercase mb-2">
            System
          </span>
          <div class="flex items-center justify-between">
            <span class="text-xs font-medium tracking-wide uppercase">
              Local Storage
            </span>
            <span class="bg-[#111827] dark:bg-zinc-800 text-white dark:text-zinc-200 border dark:border-zinc-700 text-[9px] font-bold tracking-widest px-2.5 py-1 rounded-full uppercase">
              Active
            </span>
          </div>
        </div>

      </div>
    </div>
  </Transition>
</template>

<style scoped>
.slide-enter-active,
.slide-leave-active {
  transition: transform 0.25s cubic-bezier(0.16, 1, 0.3, 1);
}

.slide-enter-from,
.slide-leave-to {
  transform: translateY(100%);
}
</style>