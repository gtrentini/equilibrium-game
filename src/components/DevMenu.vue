<script setup>
import { ref, watch } from 'vue';

const props = defineProps({
  showDevMenu: Boolean,
  initialTargetLevel: Number
});
const emit = defineEmits(['close', 'go-to-level', 'force-win', 'add-coins', 'reset-all', 'update-target-level']);

const localTargetLevel = ref(props.initialTargetLevel || 1);

watch(localTargetLevel, (val) => {
  emit('update-target-level', val);
});
watch(() => props.initialTargetLevel, (val) => {
  localTargetLevel.value = val;
});
</script>

<template>
  <Transition name="modal">
    <div v-if="showDevMenu" class="shop-modal-overlay" style="z-index: 999">
      <div class="shop-modal-content" style="border-color: #ff0055">
        <h2 style="color: #ff0055">🛠️ Dev Menu</h2>
        <div style="display: flex; flex-direction: column; gap: 1rem; margin: 2rem 0;">
          <div style="display: flex; gap: 0.5rem;">
            <input type="number" v-model="localTargetLevel" style="width: 80px; text-align: center; background: #333; color: white; border: 1px solid #555; border-radius: 6px; font-size: 1.2rem;" min="1" />
            <button class="shop-btn" style="flex: 1; min-height: 44px; padding: 0.5rem;" @click="emit('go-to-level')">Jump to Level</button>
          </div>
          <button class="shop-btn" @click="emit('force-win')">Force Win (Current Level)</button>
          <button class="shop-btn" @click="emit('add-coins')">+1000 Coins</button>
          <button class="shop-btn" style="border-color: #ff0055; color: #ffcaea" @click="emit('reset-all')">⚠️ HARD RESET ALL DATA</button>
        </div>
        <button class="close-shop-btn" @click="emit('close')">Close</button>
      </div>
    </div>
  </Transition>
</template>