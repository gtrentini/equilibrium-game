<script setup>
defineProps({
  visibleInventory: Array,
  activeTokenId: [String, null]
});
const emit = defineEmits(['select-token']);
</script>

<template>
  <div class="tokens">
    <button
      v-for="token in visibleInventory"
      :key="token.id"
      class="token-btn"
      :class="{
        active: activeTokenId === token.id,
        disabled: token.amount === 0
      }"
      @click="emit('select-token', token.id)"
      :disabled="token.amount === 0"
    >
      <span class="token-name">{{ token.name }}</span>
      <span class="token-amount">x{{ token.amount }}</span>
    </button>
  </div>
</template>