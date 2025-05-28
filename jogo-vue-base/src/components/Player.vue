// Player.vue
<template>
    <img :src="src" class="player" :style="{ left: playerX + 'px', bottom: '5%', transform: `translateY(${-jumpY}px)` }"
        alt="Jogador" />
</template>

<script setup>
const props = defineProps({
    playerX: { type: Number, required: true },
    jumpY: { type: Number, required: true },
    src: { type: String, required: true }
})
</script>

<style scoped>
.player {
    position: absolute;
    width: 230px;
    transition: left 0.1s, transform 0.1s;
    z-index: 2;
}
</style>
