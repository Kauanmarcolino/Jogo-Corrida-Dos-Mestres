<template>
  <div>
    <Menu v-if="telaAtual === 'menu'" @start="telaAtual = 'jogo'" />
    <div v-else class="cenario">
      <Hud :vidas="vidas" />
      <Boss :bossSrc="bossSrc" />
      <Player :playerX="playerX" :jumpY="jumpY" :src="playerSrc" />

      <img
        v-if="poderVisivel"
        ref="poder"
        src="/poder-binario.png"
        alt="Poder"
        class="poder"
        :style="{ right: poderX + 'px' }"
      />

      <!-- Som do nível 1 -->
      <audio ref="somNivel1" src="/nivel1.mp3" loop />

      <!-- Botão de som -->
      <button @click="toggleSom" class="btn-som">
        <img :src="somAtivo ? '/iconSomLigado.png' : '/iconSomDesligado.png'" alt="Som" />
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, watch, onBeforeUnmount, onMounted } from 'vue'
import Menu from './Menu.vue'
import Hud from './Hud.vue'
import Player from './Player.vue'
import Boss from './Boss.vue'

const telaAtual    = ref('menu')
const vidas        = reactive([true, true, true])
const playerX      = ref(50)
const jumpY        = ref(0)
const playerSrc    = ref('/player.png')
const bossSrc      = ref('/boss.png')
const poderX       = ref(0)
const poderVisivel = ref(false)
const somNivel1    = ref(null)
const somAtivo     = ref(false) // começa mutado

const speed     = 5
const jumpForce = 30
const gravity   = 0.8
const moving    = { left: false, right: false }
const grounded  = ref(true)
let   velocityY = 0
let   frameLoop = null
let   playerAnim= null
let   bossAnim  = null
let   poderLoop = null
const poderAnims= []

function onKeyDown(e) {
  if (e.key === 'a') moving.left  = true
  if (e.key === 'd') moving.right = true
  if ((e.key === ' '|| e.code==='Space') && grounded.value) {
    velocityY = jumpForce
    grounded.value = false
  }
}

function onKeyUp(e) {
  if (e.key === 'a') moving.left  = false
  if (e.key === 'd') moving.right = false
}

function gameLoop() {
  if (moving.left)  playerX.value = Math.max(0, playerX.value - speed)
  if (moving.right) playerX.value = Math.min(window.innerWidth - 100, playerX.value + speed)
  if (!grounded.value) {
    velocityY -= gravity
    jumpY.value += velocityY
    if (jumpY.value <= 0) {
      jumpY.value = 0
      grounded.value = true
      velocityY = 0
    }
  }
  frameLoop = requestAnimationFrame(gameLoop)
}

function iniciarJogo() {
  window.addEventListener('keydown', onKeyDown)
  window.addEventListener('keyup', onKeyUp)
  frameLoop = requestAnimationFrame(gameLoop)

  if (somNivel1.value && somAtivo.value) {
    somNivel1.value.currentTime = 0
    somNivel1.value.volume = 0.4
    somNivel1.value.play().catch(() => {})
  }

  playerAnim = setInterval(() => {
    playerSrc.value = playerSrc.value.endsWith('2.png')
      ? '/player.png' : '/player2.png'
  }, 300)

  bossAnim = setInterval(() => {
    bossSrc.value = bossSrc.value.endsWith('2.png')
      ? '/boss.png' : '/boss2.png'
  }, 300)

  poderLoop = setInterval(() => {
    poderVisivel.value = true
    poderX.value = 0
    let podePerder = true
    const anim = setInterval(() => {
      poderX.value += 10
      const pEl = document.querySelector('.poder')
      const pl  = document.querySelector('.player')
      if (pEl && pl && podePerder) {
        const r1 = pEl.getBoundingClientRect()
        const r2 = pl.getBoundingClientRect()
        if (
          r1.left < r2.right &&
          r1.right > r2.left &&
          r1.top < r2.bottom &&
          r1.bottom > r2.top
        ) {
          const idx = vidas.findIndex(v => v)
          if (idx !== -1) vidas[idx] = false
          podePerder = false
        }
      }
      if (poderX.value > window.innerWidth) {
        clearInterval(anim)
        poderVisivel.value = false
      }
    }, 50)
    poderAnims.push(anim)
  }, 3000)
}

function limparJogo() {
  window.removeEventListener('keydown', onKeyDown)
  window.removeEventListener('keyup', onKeyUp)
  if (frameLoop) cancelAnimationFrame(frameLoop)
  if (playerAnim) clearInterval(playerAnim)
  if (bossAnim) clearInterval(bossAnim)
  if (poderLoop) clearInterval(poderLoop)
  poderAnims.forEach(id => clearInterval(id))
  poderAnims.length = 0
  if (somNivel1.value) somNivel1.value.pause()
}

function toggleSom() {
  if (!somNivel1.value) return
  somAtivo.value = !somAtivo.value
  somAtivo.value ? somNivel1.value.play().catch(() => {}) : somNivel1.value.pause()
}

watch(telaAtual, t => t === 'jogo' ? iniciarJogo() : limparJogo())

onBeforeUnmount(() => limparJogo())

onMounted(() => {
  const tentarTocarSom = () => {
    if (telaAtual.value === 'jogo' && somNivel1.value && somAtivo.value) {
      somNivel1.value.play().catch(() => {})
    }
    window.removeEventListener('click', tentarTocarSom)
  }

  window.addEventListener('click', tentarTocarSom)
})
</script>

<style scoped>
.cenario {
  position: relative;
  width: 100vw;
  height: 100vh;
  background: url('/cenario.png') no-repeat center center;
  background-size: cover;
  overflow: hidden;
}

.poder {
  position: absolute;
  bottom: 160px;
  width: 80px;
  transition: right 0.05s;
  z-index: 2;
}

.btn-som {
  position: absolute;
  top: 10px;
  right: 20px;
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




