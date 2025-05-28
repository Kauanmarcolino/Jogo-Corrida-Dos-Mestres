<template>
    <div class="hud">
        <img v-for="(vida, i) in vidas" :key="i" :src="vida ? '/coracao.png' : '/coracaoVazio.png'" class="coracao" />
    </div>
</template>

<script setup>
defineProps(['vidas'])
</script>
