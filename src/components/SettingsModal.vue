<script setup>
defineProps({
  showSettings: Boolean,
  settings: Object
});
const emit = defineEmits(['close', 'update-settings', 'wipe-data']);
</script>

<template>
  <Transition name="modal">
    <div v-if="showSettings" class="shop-modal-overlay">
      <div class="shop-modal-content">
        <div class="shop-modal-header">
          <h2>⚙️ Settings</h2>
          <button class="close-shop-btn" @click="emit('close')">Close</button>
        </div>
        
        <div class="shop-items">
          <button 
            class="shop-btn" 
            :style="settings.soundEnabled ? 'border-color: #4ade80; color: #4ade80;' : 'border-color: #f87171; color: #f87171;'"
            @click="emit('update-settings', { ...settings, soundEnabled: !settings.soundEnabled })"
          >
            <span class="shop-title">Sound Effects</span>
            <span class="shop-cost">{{ settings.soundEnabled ? 'ON' : 'OFF' }}</span>
          </button>

          <button 
            class="shop-btn" 
            :style="settings.hapticsEnabled ? 'border-color: #4ade80; color: #4ade80;' : 'border-color: #f87171; color: #f87171;'"
            @click="emit('update-settings', { ...settings, hapticsEnabled: !settings.hapticsEnabled })"
          >
            <span class="shop-title">Haptic Vibrations</span>
            <span class="shop-cost">{{ settings.hapticsEnabled ? 'ON' : 'OFF' }}</span>
          </button>

          <button 
            class="shop-btn" 
            style="border-color: #ef4444; color: #fecaca; background: rgba(239, 68, 68, 0.1); width: 100%;"
            @click="emit('wipe-data')"
          >
            <span class="shop-title">⚠️ WIPE SAVE DATA ⚠️</span>
          </button>
        </div>
        <div style="margin-top: 2rem; font-size: 0.8rem; color: #6b7280; text-align: center; font-family: monospace;">v1.0.0 - Equilibrium</div>
      </div>
    </div>
  </Transition>
</template>