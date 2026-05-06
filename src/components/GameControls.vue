<script setup>
defineProps({
  isWin: Boolean,
  isDailyMode: Boolean,
  canUndo: Boolean
});
const emit = defineEmits(['undo', 'reset', 'reroll', 'open-shop']);
</script>

<template>
  <div class="controls">
    <button class="control-btn" @click="emit('undo')" :disabled="!canUndo || isWin">
      ↩️ Undo
    </button>
    <button class="control-btn restart" @click="emit('reset')" :disabled="isWin">
      🔄 Reset
    </button>
    <button v-if="!isDailyMode" class="control-btn reroll" @click="emit('reroll')" :disabled="isWin">
      🎲 Scrap
    </button>
    <button class="control-btn shop-toggle" @click="emit('open-shop')" :disabled="isWin">
      🛒 Shop
    </button>
  </div>
</template>