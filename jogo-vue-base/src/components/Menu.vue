<template>
  <div class="tela-menu">
    <div class="binarios binarios-esquerda">
      <span
        v-for="(bit, i) in bitsEsq"
        :key="'esq-' + i"
        class="bit"
        :style="{
          top: bit.top + 'px',
          left: bit.left + '%',
          fontSize: bit.size + 'px',
          color: bit.color
        }"
      >
        {{ bit.value }}
      </span>
    </div>

    <div class="menu-center">
      <img src="/menu.png" alt="Menu" class="img-menu" />
      <img src="/brilhos.png" class="brilho" style="top: 40px; left: 690px;" />
      <button class="botao" @click="emit('start')">Iniciar Jogo</button>
    </div>

    <div class="binarios binarios-direita">
      <span
        v-for="(bit, i) in bitsDir"
        :key="'dir-' + i"
        class="bit"
        :style="{
          top: bit.top + 'px',
          left: bit.left + '%',
          fontSize: bit.size + 'px',
          color: bit.color
        }"
      >
        {{ bit.value }}
      </span>
    </div>

    <!-- Botão de som -->
    <button @click="toggleSom" class="btn-som">
      <img :src="somAtivo ? '/iconSomLigado.png' : '/iconSomDesligado.png'" alt="Som" />
    </button>

    <!-- Música -->
    <audio ref="musica" src="/musica.mp3" autoplay loop />
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
const emit = defineEmits(['start'])

const bitsEsq = ref([])
const bitsDir = ref([])
const cores = ['#00ff00', '#ff00ff', '#00ffff', '#ffff00', '#ff4444', '#ff8800', '#4488ff']

// Som
const somAtivo = ref(true)
const musica = ref(null)

function gerarBits() {
  const criar = () => ({
    value: Math.round(Math.random()),
    top: Math.random() * window.innerHeight,
    left: Math.random() * 100,
    speed: Math.random() * 2 + 1,
    color: cores[Math.floor(Math.random() * cores.length)],
    size: Math.random() * 10 + 10
  })

  bitsEsq.value = Array.from({ length: 50 }, criar)
  bitsDir.value = Array.from({ length: 50 }, criar)

  setInterval(() => {
    bitsEsq.value.forEach(b => {
      b.top += b.speed
      if (b.top > window.innerHeight) {
        b.top = -20
        b.left = Math.random() * 100
        b.value = Math.round(Math.random())
      }
    })
    bitsDir.value.forEach(b => {
      b.top += b.speed
      if (b.top > window.innerHeight) {
        b.top = -20
        b.left = Math.random() * 100
        b.value = Math.round(Math.random())
      }
    })
  }, 50)
}

onMounted(() => {
  gerarBits()

  if (musica.value) {
    musica.value.volume = 0.4
    musica.value.play().catch(() => {})
  }
})

function toggleSom() {
  if (!musica.value) return
  somAtivo.value = !somAtivo.value
  somAtivo.value ? musica.value.play() : musica.value.pause()
}
</script>

<style scoped>
.tela-menu {
  display: flex;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
}

.binarios {
  flex: 1;
  position: relative;
  background: #000;
  overflow: hidden;
  pointer-events: none;
}

.menu-center {
  flex: 0 0 auto;
  position: relative;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.img-menu {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
}

.brilho {
  position: absolute;
  width: 50px;
  height: auto;
  animation: piscar 2s infinite alternate;
}

@keyframes piscar {
  0% {
    opacity: 0.3;
  }

  100% {
    opacity: 1;
  }
}

.botao {
  margin-top: -10%;
  padding: 15px 30px;
  background: rgb(143, 9, 9);
  border: none;
  font-size: 20px;
  color: #ffffff;
  cursor: pointer;
  border-radius: 0px;
  font-weight: bold;
  transition: background 0.3s, transform 0.1s;
  box-shadow: 4px 4px 0px #000000;
}

.botao:hover {
  background: rgb(104, 1, 1);
}

.botao:active {
  background: #090;
  transform: scale(0.95);
}

.bit {
  position: absolute;
  white-space: nowrap;
  font-family: monospace;
  font-weight: bold;
  text-shadow: 0 0 4px #0f0;
}

.btn-som {
  position: absolute;
  top: 4px;
  right: 680px;
  background: none;
  border: none;
  cursor: pointer;
  z-index: 20;
}

.btn-som img {
  width: 52px;
  height: 52px;
}
</style>
