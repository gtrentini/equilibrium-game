<script setup>
defineProps({
  hasPlayedDailyToday: Boolean,
  dailyStreak: Number,
});
const emit = defineEmits(["start-game", "start-daily", "open-settings", "open-tutorial"]);
</script>

<template>
  <div class="title-screen">
    <h1>Equilibrium</h1>
    <button class="start-btn" @click="emit('start-game')">Start Game</button>
    <div
      style="
        display: flex;
        flex-direction: column;
        align-items: center;
        gap: 0.5rem;
      "
    >
      <button
        class="start-btn daily-btn"
        @click="emit('start-daily')"
        :disabled="hasPlayedDailyToday"
      >
        {{ hasPlayedDailyToday ? "Daily Complete ✅" : "Daily Challenge 📅" }}
      </button>
      <span
        v-if="dailyStreak > 0"
        style="
          font-size: 1.1rem;
          color: #ffd700;
          font-weight: bold;
          text-shadow: 0 2px 4px rgba(0, 0, 0, 0.5);
        "
      >
        🔥 Streak: {{ dailyStreak }}
      </span>
    </div>
    <button class="settings-btn" @click="emit('open-tutorial')">
      📖 How to Play
    </button>
    <button class="settings-btn" @click="emit('open-settings')">
      ⚙️ Settings
    </button>
  </div>
</template>
