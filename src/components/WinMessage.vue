<script setup>
defineProps({
  isWin: Boolean,
  earnedStars: Number,
  movesTaken: Number,
  levelPar: Number,
  earnedCoins: Number
});
const emit = defineEmits(['next-level']);
</script>

<template>
  <div v-if="isWin" class="win-message">
    <h2>Level Complete!</h2>
    <div class="stars-display">
      {{ "⭐".repeat(earnedStars) }}{{ "☆".repeat(3 - earnedStars) }}
    </div>
    <p>Perfect harmony achieved.</p>
    <p class="stats-text">Moves: {{ movesTaken }} / Par: {{ levelPar }}</p>
    <div class="reward">
      <span class="reward-text">+{{ earnedCoins }} Coins!</span>
    </div>
    <button @click="emit('next-level')">Next Level</button>
  </div>
</template>