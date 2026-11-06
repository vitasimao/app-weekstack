<script setup lang="ts">
import { Check } from 'lucide-vue-next';
</script>

<template>
  <div class="fixed inset-0 bg-[#E5E5E5] flex items-center justify-center z-50">
    <!-- Tarjeta central con sombra sutil -->
    <div class="w-28 h-28 bg-[#EDEDED] rounded-2xl flex items-center justify-center shadow-sm animate-pulse">
      <div class="relative flex items-center">
        <!-- Letra W -->
        <span class="text-4xl font-extrabold text-[#111827] tracking-tighter select-none">W</span>
        
        <!-- Cuadro naranja con check -->
        <div class="absolute -top-1 -right-3.5 w-4 h-4 bg-[#FF6B00] rounded-[3px] flex items-center justify-center shadow-sm animate-bounce">
          <Check :size="11" class="text-white stroke-[3.5]" />
        </div>
      </div>
    </div>
  </div>
</template>