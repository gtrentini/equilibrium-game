<script setup>
defineProps({
  grid: Array,
  isWin: Boolean,
  animatingCells: Object
});
const emit = defineEmits(['apply-token']);
</script>

<template>
  <div class="grid-wrapper">
    <div class="grid" :class="{ 'game-won': isWin }" :style="{ gridTemplateColumns: `repeat(${grid.length}, 1fr)` }">
      <template v-for="(row, rIndex) in grid" :key="rIndex">
        <div
          v-for="(cell, cIndex) in row"
          :key="cell.id"
          class="cell"
          :class="{
            void: cell.type === 'void',
            inverter: cell.type === 'inverter',
            quantum: cell.type === 'quantum_a' || cell.type === 'quantum_b',
            zero: cell.value === 0 && cell.type !== 'void',
            positive: cell.value > 0 && cell.type !== 'void',
            negative: cell.value < 0 && cell.type !== 'void',
            animating: animatingCells[cell.id]
          }"
          @click="cell.type !== 'void' ? emit('apply-token', rIndex, cIndex) : null"
        >
          {{ cell.type !== "void" ? cell.value : "" }}
          <div v-if="cell.type === 'inverter'" class="inverter-icon">🔃</div>
          <div v-if="cell.type === 'quantum_a' || cell.type === 'quantum_b'" class="quantum-icon">⚛️</div>
        </div>
      </template>
    </div>
  </div>
</template>