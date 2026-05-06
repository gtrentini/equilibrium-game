<script setup>
defineProps({
  reversedLevels: Array,
  playerProfile: Object
});
const emit = defineEmits(['select-level', 'back-to-menu']);
</script>

<template>
  <div class="level-select-wrapper">
    <header>
      <h1>Select Level</h1>
    </header>
    <div class="level-grid">
      <button
        v-for="lvl in reversedLevels"
        :key="lvl"
        class="level-btn"
        :class="{ cleared: playerProfile.levelStats[lvl]?.cleared }"
        @click="emit('select-level', lvl)"
      >
        <span class="lvl-number">Level {{ lvl }}</span>
        <span class="lvl-stars" v-if="playerProfile.levelStats[lvl]?.cleared">
          [{{ "⭐".repeat(playerProfile.levelStats[lvl]?.stars || 0) }}{{ "☆".repeat(3 - (playerProfile.levelStats[lvl]?.stars || 0)) }}]
        </span>
      </button>
    </div>
    <button class="back-btn" @click="emit('back-to-menu')">
      Back to Menu
    </button>
  </div>
</template>