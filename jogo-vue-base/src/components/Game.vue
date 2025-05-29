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

      <!-- Som de impacto -->
      <audio ref="somImpacto" src="/somImpacto.mp3" />

      <!-- Som de game over -->
      <audio ref="somGameOver" src="/gameover.mp3" />

      <!-- Botão de som -->
      <button @click="toggleSom" class="btn-som">
        <img
          :src="somAtivo ? '/iconSomLigado.png' : '/iconSomDesligado.png'"
          alt="Som"
        />
      </button>

      <!-- Tela de Game Over -->
      <div v-if="gameOver" class="game-over-overlay">
        <div class="game-over-box">
          <h1 class="game-over-text">GAME OVER</h1>
          <button @click="reiniciarJogo">REINICIAR</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, watch, onBeforeUnmount, onMounted } from "vue";
import Menu from "./Menu.vue";
import Hud from "./Hud.vue";
import Player from "./Player.vue";
import Boss from "./Boss.vue";

const telaAtual = ref("menu");
const vidas = reactive([true, true, true]);
const playerX = ref(50);
const jumpY = ref(0);
const playerSrc = ref("/player.png");
const bossSrc = ref("/boss.png");
const poderX = ref(0);
const poderVisivel = ref(false);
const somNivel1 = ref(null);
const somImpacto = ref(null);
const somGameOver = ref(null);
const somAtivo = ref(false);
const gameOver = ref(false);

const speed = 5;
const jumpForce = 30;
const gravity = 0.8;
const moving = { left: false, right: false };
const grounded = ref(true);
let velocityY = 0;
let frameLoop = null;
let playerAnim = null;
let bossAnim = null;
let poderLoop = null;
const poderAnims = [];

function onKeyDown(e) {
  if (e.key === "a") moving.left = true;
  if (e.key === "d") moving.right = true;
  if ((e.key === " " || e.code === "Space") && grounded.value) {
    velocityY = jumpForce;
    grounded.value = false;
  }
}

function onKeyUp(e) {
  if (e.key === "a") moving.left = false;
  if (e.key === "d") moving.right = false;
}

function gameLoop() {
  if (moving.left) playerX.value = Math.max(0, playerX.value - speed);
  if (moving.right)
    playerX.value = Math.min(window.innerWidth - 100, playerX.value + speed);
  if (!grounded.value) {
    velocityY -= gravity;
    jumpY.value += velocityY;
    if (jumpY.value <= 0) {
      jumpY.value = 0;
      grounded.value = true;
      velocityY = 0;
    }
  }
  frameLoop = requestAnimationFrame(gameLoop);
}

function verificarGameOver() {
  if (vidas.every((v) => !v)) {
    gameOver.value = true;
    limparJogo();
    if (somNivel1.value) somNivel1.value.pause();
    if (somGameOver.value && somAtivo.value) {
      somGameOver.value.currentTime = 0;
      somGameOver.value.play().catch(() => {});
    }
  }
}

function reiniciarJogo() {
  vidas.splice(0, vidas.length, true, true, true);
  playerX.value = 50;
  jumpY.value = 0;
  gameOver.value = false;

  // força a troca de tela para reiniciar a lógica do jogo
  telaAtual.value = 'menu';
  setTimeout(() => {
    telaAtual.value = 'jogo';
  }, 50); // espera 50ms e volta pro jogo
}

function iniciarJogo() {
  window.addEventListener("keydown", onKeyDown);
  window.addEventListener("keyup", onKeyUp);
  frameLoop = requestAnimationFrame(gameLoop);

  if (somNivel1.value && somAtivo.value) {
    somNivel1.value.currentTime = 0;
    somNivel1.value.volume = 1.0;
    somNivel1.value.play().catch(() => {});
  }

  playerAnim = setInterval(() => {
    playerSrc.value = playerSrc.value.endsWith("2.png")
      ? "/player.png"
      : "/player2.png";
  }, 300);

  bossAnim = setInterval(() => {
    bossSrc.value = bossSrc.value.endsWith("2.png")
      ? "/boss.png"
      : "/boss2.png";
  }, 300);

  poderLoop = setInterval(() => {
    poderVisivel.value = true;
    poderX.value = 0;
    let podePerder = true;
    const anim = setInterval(() => {
      poderX.value += 10;
      const pEl = document.querySelector(".poder");
      const pl = document.querySelector(".player");
      if (pEl && pl && podePerder) {
        const r1 = pEl.getBoundingClientRect();
        const r2 = pl.getBoundingClientRect();

        if (
          r1.left < r2.right &&
          r1.right > r2.left &&
          r1.top < r2.bottom &&
          r1.bottom > r2.top
        ) {
          const idx = vidas.findIndex((v) => v);
          if (idx !== -1) vidas[idx] = false;
          podePerder = false;

          if (somImpacto.value && somAtivo.value) {
            somImpacto.value.volume = 1.0;
            somImpacto.value.currentTime = 0;
            somImpacto.value.play().catch(() => {});
          }

          poderVisivel.value = false;
          clearInterval(anim);
          verificarGameOver();
        }
      }
      if (poderX.value > window.innerWidth) {
        clearInterval(anim);
        poderVisivel.value = false;
      }
    }, 50);
    poderAnims.push(anim);
  }, 3000);
}

function limparJogo() {
  window.removeEventListener("keydown", onKeyDown);
  window.removeEventListener("keyup", onKeyUp);
  if (frameLoop) cancelAnimationFrame(frameLoop);
  if (playerAnim) clearInterval(playerAnim);
  if (bossAnim) clearInterval(bossAnim);
  if (poderLoop) clearInterval(poderLoop);
  poderAnims.forEach((id) => clearInterval(id));
  poderAnims.length = 0;
  if (somNivel1.value) somNivel1.value.pause();
}

function toggleSom() {
  if (!somNivel1.value) return;
  somAtivo.value = !somAtivo.value;
  somAtivo.value
    ? somNivel1.value.play().catch(() => {})
    : somNivel1.value.pause();
}

watch(telaAtual, (t) => (t === "jogo" ? iniciarJogo() : limparJogo()));

onBeforeUnmount(() => limparJogo());

onMounted(() => {
  const tentarTocarSom = () => {
    if (telaAtual.value === "jogo" && somNivel1.value && somAtivo.value) {
      somNivel1.value.play().catch(() => {});
    }
    window.removeEventListener("click", tentarTocarSom);
  };
  window.addEventListener("click", tentarTocarSom);
});
</script>

<style scoped>
@import url("https://fonts.googleapis.com/css2?family=Press+Start+2P&display=swap");

.cenario {
  position: relative;
  width: 100vw;
  height: 100vh;
  background: url("/cenario.png") no-repeat center center;
  background-size: contain; /* 🔄 troca ESSA linha */
  background-color: black;  /* cor de fundo se sobrar espaço */
  background-repeat: no-repeat;
  image-rendering: pixelated; /* deixa visual retrô */
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

/* Game Over - estilo retrô pixelado */
.game-over-overlay {
  position: absolute;
  inset: 0;
  background: rgba(0, 0, 0, 0.8);
  z-index: 999;
  display: flex;
  align-items: center;   /* vertical */
  justify-content: center; /* horizontal */
}

.game-over-box {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;

  height: 300px;
  width: 320px;

  background: #000;
  padding: 40px;
  border: 2px solid white;
  box-shadow: 0 0 20px #000;
  text-align: center;
}


.game-over-text {
  color: rgb(255, 255, 255);
  font-family: "Press Start 2P", monospace;
  font-size: 26px;
  text-shadow: 2px 2px rgb(0, 0, 0);
  margin-bottom: 30px;
}

.game-over-box button {
  font-family: "Press Start 2P", monospace;
  font-size: 14px;
  background-color: rgb(255, 25, 0);
  color: black;
  border: 2px solid rgb(0, 0, 0);
  padding: 12px 26px;
  border-radius: 5px;
  cursor: pointer;
  box-shadow: 2px 2px rgb(0, 0, 0);
  transition: transform 0.1s;
}

.game-over-box button:hover {
  transform: scale(1.05);
  background-color: rgba(255, 25, 0, 0.522);
}
</style>
