<script setup>
defineProps({
  showShop: Boolean,
  shopFeedback: String,
  visibleInventory: Array,
  availableThemes: Array,
  coinBank: Number,
  playerProfile: Object
});
const emit = defineEmits(['close', 'buy-token', 'set-theme']);
</script>

<template>
  <Transition name="modal">
    <div v-if="showShop" class="shop-modal-overlay">
      <div class="shop-modal-content">
        <div class="shop-modal-header">
          <h2>Shop</h2>
          <button class="close-shop-btn" @click="emit('close')">Close</button>
        </div>
        <div v-if="shopFeedback" style="color: #4ade80; margin-bottom: 1rem; font-weight: bold">
          {{ shopFeedback }}
        </div>
        <div class="shop-items">
          <button
            v-for="token in visibleInventory"
            :key="'shop-' + token.id"
            class="shop-btn"
            @click="emit('buy-token', token.id)"
            :disabled="coinBank < token.cost"
          >
            <span class="shop-title">Buy {{ token.name.split(' ')[0] }}</span>
            <span class="shop-cost">{{ token.cost }} 🪙</span>
          </button>
        </div>

        <h3 style="color: #ffd700; margin-top: 2rem; margin-bottom: 1rem; text-transform: uppercase;">
          Cosmetic Themes
        </h3>
        <div class="shop-items">
          <button
            v-for="theme in availableThemes"
            :key="'theme-' + theme.id"
            class="shop-btn"
            @click="emit('set-theme', theme.id)"
            :disabled="!playerProfile.themes?.unlocked.includes(theme.id) && coinBank < theme.cost"
            :style="playerProfile.themes?.active === theme.id ? 'border-color: #00d2ff; background: rgba(0, 210, 255, 0.1);' : ''"
          >
            <span class="shop-title">{{ theme.name }}</span>
            <span class="shop-cost" v-if="!playerProfile.themes?.unlocked.includes(theme.id)">{{ theme.cost }} 🪙</span>
            <span class="shop-cost" v-else-if="playerProfile.themes?.active === theme.id" style="color: #00d2ff">Active</span>
            <span class="shop-cost" v-else style="color: #a0a0a0">Equip</span>
          </button>
        </div>
      </div>
    </div>
  </Transition>
</template>