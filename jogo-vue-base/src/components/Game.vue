<template>
  <div>
    <Menu v-if="telaAtual === 'menu'" @start="telaAtual = 'jogo'" />
    <div v-else class="cenario">
      <Hud :vidas="vidas" />

      <!-- Sombra do Boss -->
      <img src="/sombra.png" class="sombra sombra-boss" />

      <!-- Boss -->
      <Boss :bossSrc="bossSrc" />

      <!-- Sombra do Player (fora do wrapper para não subir junto) -->
      <img
        src="/sombra.png"
        class="sombra sombra-player"
        :style="{ left: playerX + 80 + 'px', bottom: '-50px' }"
      />

      <!-- Player (que pula ou agacha) -->
      <div
        class="player-wrapper"
        :style="{
          left: playerX + 'px',
          bottom: estaAgachado ? '-45px' : jumpY + 'px'
        }"
      >
        <Player :jumpY="jumpY" :src="playerSrc" />
      </div>

      <!-- Poder -->
      <img
        v-if="poderVisivel"
        ref="poder"
        src="/poder-binario.png"
        alt="Poder"
        class="poder"
        :style="{ right: poderX + 'px' }"
      />

      <!-- Áudios -->
      <audio ref="somNivel1" src="/nivel1.mp3" loop />
      <audio ref="somImpacto" src="/somImpacto.mp3" />
      <audio ref="somGameOver" src="/gameover.mp3" />
      <audio ref="somAgachando" src="/somAgachando.mp3" />
      <audio ref="somPulo" src="/somPulo.mp3" />

      <!-- Botões -->
      <button @click="toggleSom" class="btn-som">
        <img :src="somAtivo ? '/iconSomLigado.png' : '/iconSomDesligado.png'" alt="Som" />
      </button>

      <button @click="togglePause" class="btn-pause">
        <img src="/botaoPause.png" alt="Pause" />
      </button>

      <!-- Tela de Game Over -->
      <div v-if="gameOver" class="game-over-overlay">
        <div class="game-over-container">
          <img src="/imgGameOver.png" class="img-game-over" alt="Game Over" />
          <button class="btn-reiniciar" @click="reiniciarJogo">REINICIAR</button>
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
const jogoPausado = ref(false);
const gameOver = ref(false);
const somAgachando = ref(null);
const somPulo = ref(null);

const speed = 5;
const jumpForce = 25;
const gravity = 0.8;
const grounded = ref(true);
let velocityY = 0;
let frameLoop = null;
let bossAnim = null;
let poderLoop = null;
const poderAnims = [];
const moving = { left: false, right: false, down: false };
const direcao = ref("direita");
const estaAgachado = ref(false);

function onKeyDown(e) {
  if (e.key === "a") {
    moving.left = true;
    direcao.value = "esquerda";
  }

  if (e.key === "d") {
    moving.right = true;
    direcao.value = "direita";
  }

  if (e.key === "s") {
    moving.down = true;

    if (somAgachando.value && somAtivo.value) {
      somAgachando.value.currentTime = 0;
      somAgachando.value.play().catch(() => {});
    }
  }

  if ((e.key === " " || e.code === "Space") && grounded.value) {
    velocityY = jumpForce;
    grounded.value = false;

    if (somPulo.value && somAtivo.value) {
      somPulo.value.currentTime = 0;
      somPulo.value.play().catch(() => {});
    }
  }

  if (e.key === "Enter" && telaAtual.value === "menu") {
    telaAtual.value = "jogo";
  }
}



function onKeyUp(e) {
  if (e.key === "a") moving.left = false;
  if (e.key === "d") moving.right = false;
  if (e.key === "s") moving.down = false;
}

function gameLoop() {
  if (jogoPausado.value) {
    frameLoop = requestAnimationFrame(gameLoop);
    return;
  }

  if (moving.left) playerX.value = Math.max(0, playerX.value - speed);
  if (moving.right) playerX.value = Math.min(window.innerWidth - 100, playerX.value + speed);

  if (!grounded.value) {
    velocityY -= gravity;
    jumpY.value += velocityY;
    if (jumpY.value <= 0) {
      jumpY.value = 0;
      grounded.value = true;
      velocityY = 0;
    }
  }

  if (jumpY.value > 0) {
    playerSrc.value = direcao.value === "esquerda" ? "/jumpVoltando.jpg" : "/playerGiro.png";
    estaAgachado.value = false;
  } else if (moving.down && grounded.value) {
    estaAgachado.value = true;
    playerSrc.value = (moving.left || direcao.value === "esquerda") ? "/agachado2.png" : "/agachado.png";
  } else if (moving.left) {
    estaAgachado.value = false;
    playerSrc.value = "/bonecoVoltando.jpg";
  } else if (moving.right) {
    estaAgachado.value = false;
    playerSrc.value = "/player.png";
  } else {
    estaAgachado.value = false;
    playerSrc.value = direcao.value === "esquerda" ? "/bonecoVoltando.jpg" : "/player.png";
  }

  frameLoop = requestAnimationFrame(gameLoop);
}

function togglePause() {
  jogoPausado.value = !jogoPausado.value;

  if (jogoPausado.value) {
    if (somNivel1.value) somNivel1.value.pause();
    if (bossAnim) clearInterval(bossAnim);
  } else {
    if (somNivel1.value && somAtivo.value) somNivel1.value.play().catch(() => {});
    bossAnim = setInterval(() => {
      bossSrc.value = bossSrc.value.endsWith("2.png") ? "/boss.png" : "/boss2.png";
    }, 300);
  }
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
  telaAtual.value = "menu";
  setTimeout(() => {
    telaAtual.value = "jogo";
  }, 50);
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

  bossAnim = setInterval(() => {
    bossSrc.value = bossSrc.value.endsWith("2.png") ? "/boss.png" : "/boss2.png";
  }, 300);

  poderLoop = setInterval(() => {
    if (jogoPausado.value) return;
    poderVisivel.value = true;
    poderX.value = 0;
    let podePerder = true;

    const anim = setInterval(() => {
      if (jogoPausado.value) return;
      poderX.value += 10;

      const pEl = document.querySelector(".poder");
      const pl = document.querySelector(".player");

      if (pEl && pl && podePerder) {
        const r1Full = pEl.getBoundingClientRect();
        const r1 = {
          top: r1Full.top + 20,
          bottom: r1Full.bottom - 20,
          left: r1Full.left + 20,
          right: r1Full.right - 20
        };

        const r2Full = pl.getBoundingClientRect();
        const r2 = {
          top: r2Full.top + 20,
          bottom: r2Full.bottom - 20,
          left: r2Full.left + 20,
          right: r2Full.right - 20
        };

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
  background-size: cover;
  image-rendering: pixelated;
  overflow: hidden;
}

.player-wrapper {
  position: absolute;
  bottom: 0;
  z-index: 2;
}

.sombra {
  width: 150px;
  height: auto;
  image-rendering: pixelated;
  pointer-events: none;
  opacity: 0.4;
  filter: brightness(0.2);
  z-index: 1;
}

.sombra-player {
  position: absolute;
  bottom: -4px;
  left: 84px;
  z-index: 1;
}

.sombra-boss {
  position: absolute;
  bottom: -85px;     /* ou ajuste fino conforme necessário */
  right: 70px;     /* ou ajuste fino conforme necessário */
  width: 320px;     /* <<< aqui você aumenta o tamanho da sombra */
  height: auto;
  image-rendering: pixelated;
  pointer-events: none;
  opacity: 0.2;
  filter: brightness(0.2);
  z-index: 1;
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
  padding: 3em;
}

.btn-som img {
  width: 60px;
  height: 60px;
}

.game-over-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.9);
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  z-index: 9999;
}

.game-over-container {
  position: relative;
  display: flex;
  justify-content: center;
  align-items: center;
  flex-direction: column;
}

.img-game-over {
  width: 100%;
  max-width: 700px;
  image-rendering: pixelated;
  pointer-events: none;
}

.btn-reiniciar {
  position: absolute;
  bottom: 38.8%;
  font-family: 'Press Start 2P', monospace;
  font-size: 16px;
  background: red;
  color: #fff;
  padding: 16px 32px;
  border: 4px solid black;
  box-shadow: 4px 4px black;
  cursor: pointer;
  transition: transform 0.1s;
  z-index: 1000;
}

.btn-reiniciar:hover {
  transform: scale(1.05);
  background: rgb(100, 0, 0);
}

.btn-pause {
  position: absolute;
  top: 10px;
  right: 100px;
  background: none;
  border: none;
  cursor: pointer;
  z-index: 20;
}

.btn-pause img {
  width: 52px;
  height: 52px;
}


.pause-overlay {
  position: absolute;
  inset: 0;
  background-color: rgba(0, 0, 0, 0.8);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 9998;
}

.texto-pause {
  font-family: 'Press Start 2P', monospace;
  color: white;
  font-size: 20px;
}
</style>
