<template>
  <img :src="src" :key="src" class="player" alt="Jogador" />
</template>

<script setup>
const props = defineProps({
  src: { type: String, required: true },
  jumpY: Number // mesmo se não estiver usando, mantenha
  });
</script>

<style scoped>
.player {
  position: absolute;
  width: 230px;
  transition: left 0.1s, transform 0.1s;
  z-index: 2;
}
</style>

