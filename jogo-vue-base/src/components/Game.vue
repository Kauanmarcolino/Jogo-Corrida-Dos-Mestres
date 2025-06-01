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

      <!-- Moedas -->
      <img
        v-if="mostrarMoeda"
        :src="`/moedaBronze${moedaFrame}.png`"
        alt="Moeda Girando"
        class="moeda-girando"
      />

      <img
        v-if="mostrarMoedaPrata"
        :src="`/moedaPrata${moedaPrataFrame}.png`"
        alt="Moeda Prata Girando"
        class="moeda-girando"
      />

   <img
  v-if="mostrarMoedaDourada"
  :src="`/moedaDourada${moedaDouradaFrame}.png`"
  class="moeda-girando-dourada"
/>

      <!-- Perguntas -->
      <div v-if="mostrarPergunta" class="pergunta-overlay">
        <img src="/imgPerguntaBronze.png" class="img-pergunta" />
        <div class="contador">{{ tempoRestante }}</div>
      </div>

      <div v-if="mostrarPerguntaPrata" class="pergunta-overlay">
        <img src="/perguntaPrata.png" class="img-pergunta" />
        <div class="contador">{{ tempoRestante }}</div>
      </div>

      <div v-if="mostrarPerguntaDourada" class="pergunta-overlay">
        <img src="/perguntaDourada.png" class="img-pergunta" />
        <div class="contador">{{ tempoRestante }}</div>
      </div>

      <!-- Player -->
      <div
        class="player-wrapper"
        :style="{
          left: playerX + 'px',
          bottom: estaAgachado ? '-45px' : jumpY + 'px',
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
      <audio ref="somMoeda" src="/public/somMoeda.wav" />
      <audio ref="somAcerto" src="/somAcerto.wav" />
      <audio ref="somPerda" src="/somPerda.mp3" />
      <audio ref="somRelogio" src="/somRelogio.mp3" />

      <!-- Botões -->
      <button @click="toggleSom" class="btn-som">
        <img
          :src="somAtivo ? '/iconSomLigado.png' : '/iconSomDesligado.png'"
          alt="Som"
        />
      </button>

      <button @click="togglePause" class="btn-pause">
        <img src="/botaoPause.png" alt="Pause" />
      </button>

      <!-- Tela de Game Over -->
      <div v-if="gameOver" class="game-over-overlay">
        <img src="/imgGameOver.png" class="img-game-over" alt="Game Over" />
        <button class="btn-reiniciar" @click="reiniciarJogo">REINICIAR</button>
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
const moedaFrame = ref(1);
const mostrarMoeda = ref(false);
let moedaAnimacao = null;
const somMoeda = ref(null);
const mostrarPergunta = ref(false);
const respostaDigitada = ref("");
const tempoRestante = ref(10);
let timerPergunta = null;
const somAcerto = ref(null);
const perguntaPausandoJogo = ref(false);
const somRelogio = ref(null);
const somPerda = ref(null);
const moedaPrataFrame = ref(1);
const mostrarMoedaPrata = ref(false);
const mostrarPerguntaPrata = ref(false);
const mostrarMoedaDourada = ref(false);
const moedaDouradaFrame = ref(1);
const mostrarPerguntaDourada = ref(false);
let animacaoPrata = null;
let animacaoDourada = null; // Adicionado para controlar animação da dourada

const speed = 5;
const jumpForce = 27;
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

  // Resposta da pergunta BRONZE
  if (mostrarPergunta.value && /^[0-9]$/.test(e.key)) {
    respostaDigitada.value += e.key;

    if (respostaDigitada.value === "7") {
      encerrarPergunta(true); // Acertou
    } else if (respostaDigitada.value.length >= 2) {
      encerrarPergunta(false); // Errou
    }
  }

  // Resposta da pergunta PRATA
  if (mostrarPerguntaPrata.value && /^[0-9]$/.test(e.key)) {
    respostaDigitada.value += e.key;

    if (respostaDigitada.value === "8") {
      encerrarPerguntaPrata(true); // Acertou
    } else if (respostaDigitada.value.length >= 2) {
      encerrarPerguntaPrata(false); // Errou
    }
  }

  //Resposta da pergunta DOURADA
  if (mostrarPerguntaDourada.value && /^[a-zA-Z]$/.test(e.key)) {
    respostaDigitada.value += e.key.toLowerCase();

    if (respostaDigitada.value.toLowerCase() === "x") {
      encerrarPerguntaDourada(true); // Acertou
    } else if (respostaDigitada.value.length >= 2) {
      encerrarPerguntaDourada(false); // Errou
    }
  }
}

function onKeyUp(e) {
  if (e.key === "a") moving.left = false;
  if (e.key === "d") moving.right = false;
  if (e.key === "s") moving.down = false;
}

function gameLoop() {
  if (jogoPausado.value || perguntaPausandoJogo.value) return;


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

  if (!moving.down) estaAgachado.value = false;

  if (jumpY.value > 0) {
    playerSrc.value =
      direcao.value === "esquerda" ? "/jumpVoltando.jpg" : "/playerGiro.png";
    estaAgachado.value = false;
  } else if (moving.down && grounded.value) {
    estaAgachado.value = true;
    playerSrc.value =
      moving.left || direcao.value === "esquerda"
        ? "/agachado2.png"
        : "/agachado.png";
  } else if (moving.left) {
    estaAgachado.value = false;
    playerSrc.value = "/bonecoVoltando.jpg";
  } else if (moving.right) {
    estaAgachado.value = false;
    playerSrc.value = "/player.png";
  } else {
    estaAgachado.value = false;
    playerSrc.value =
      direcao.value === "esquerda" ? "/bonecoVoltando.jpg" : "/player.png";
  }

  const checarColisaoMoeda = (selector, callback) => {
    const moedaEl = document.querySelector(selector);
    const playerEl = document.querySelector(".player");
    if (moedaEl && playerEl) {
      const r1 = moedaEl.getBoundingClientRect();
      const r2 = playerEl.getBoundingClientRect();
      const colidiu =
        r1.left < r2.right &&
        r1.right > r2.left &&
        r1.top < r2.bottom &&
        r1.bottom > r2.top;
      if (colidiu) {
        callback();
      }
    }
  };

  if (mostrarMoeda.value) {
    checarColisaoMoeda(".moeda-girando", () => {
      mostrarMoeda.value = false;
      if (somMoeda.value && somAtivo.value) {
        somMoeda.value.currentTime = 0;
        somMoeda.value.play().catch(() => {});
      }
      iniciarPergunta();
      return;
    });
  }

  if (mostrarMoedaPrata.value) {
    checarColisaoMoeda(".moeda-girando", () => {
      mostrarMoedaPrata.value = false;
      if (somMoeda.value && somAtivo.value) {
        somMoeda.value.currentTime = 0;
        somMoeda.value.play().catch(() => {});
      }
      iniciarPerguntaPrata();
      return;
    });
  }

  if (mostrarMoedaDourada.value) {
    checarColisaoMoeda(".moeda-girando-dourada", () => {
      mostrarMoedaDourada.value = false;
      if (somMoeda.value && somAtivo.value) {
        somMoeda.value.currentTime = 0;
        somMoeda.value.play().catch(() => {});
      }
      iniciarPerguntaDourada();
      return;
    });
  }
  frameLoop = requestAnimationFrame(gameLoop);
}

function iniciarPerguntaPrata() {
  mostrarPerguntaPrata.value = true;
  respostaDigitada.value = "";
  tempoRestante.value = 10;
  perguntaPausandoJogo.value = true;
  jogoPausado.value = true;

  if (somRelogio.value && somAtivo.value) {
    somRelogio.value.currentTime = 0;
    somRelogio.value.play().catch(() => {});
  }

  timerPergunta = setInterval(() => {
    tempoRestante.value--;
    if (tempoRestante.value <= 0) {
      encerrarPerguntaPrata(false);
    }
  }, 1000);
}

function encerrarPerguntaPrata(acertou) {
  if (!mostrarPerguntaPrata.value) return;
  clearInterval(timerPergunta);
  mostrarPerguntaPrata.value = false;
  perguntaPausandoJogo.value = false;

  if (somRelogio.value) somRelogio.value.pause();

  if (somAtivo.value) {
    if (acertou && somAcerto.value) {
      somAcerto.value.currentTime = 0;
      somAcerto.value.play().catch(() => {});
    } else if (!acertou && somPerda.value) {
      somPerda.value.currentTime = 0;
      somPerda.value.play().catch(() => {});
      const idx = vidas.findIndex((v) => v);
      if (idx !== -1) vidas[idx] = false;
      verificarGameOver();
    }
  }

  if (!grounded.value) {
    jumpY.value = 0;
    grounded.value = true;
    velocityY = 0;
  }

  // ✅ Libera o jogo
  jogoPausado.value = false;
  frameLoop = requestAnimationFrame(gameLoop);

  // ⏱️ Mostra a moeda dourada depois de 4s
  setTimeout(() => {
    mostrarMoedaDourada.value = true;
    moedaDouradaFrame.value = 1;
    if (animacaoDourada) clearInterval(animacaoDourada);
    animacaoDourada = setInterval(() => {
      moedaDouradaFrame.value = moedaDouradaFrame.value === 4 ? 1 : moedaDouradaFrame.value + 1;
    }, 150);
  }, 4000);
}


function togglePause() {
  jogoPausado.value = !jogoPausado.value;

  if (jogoPausado.value) {
    if (somNivel1.value) somNivel1.value.pause();
    if (bossAnim) clearInterval(bossAnim);
  } else {
    if (somNivel1.value && somAtivo.value)
      somNivel1.value.play().catch(() => {});
    bossAnim = setInterval(() => {
      bossSrc.value = bossSrc.value.endsWith("2.png")
        ? "/boss.png"
        : "/boss2.png";
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
  mostrarMoeda.value = false;
mostrarMoedaPrata.value = false;
mostrarMoedaDourada.value = false;
mostrarPergunta.value = false;
mostrarPerguntaPrata.value = false;
mostrarPerguntaDourada.value = false;
respostaDigitada.value = "";
tempoRestante.value = 10;
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
    bossSrc.value = bossSrc.value.endsWith("2.png")
      ? "/boss.png"
      : "/boss2.png";
  }, 300);

  poderLoop = setInterval(() => {
    if (jogoPausado.value) return;
    poderVisivel.value = true;
    poderX.value = 0;
    let podePerder = true;

    function animLoop() {
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
          right: r1Full.right - 20,
        };

        const r2Full = pl.getBoundingClientRect();
        const r2 = {
          top: r2Full.top + 20,
          bottom: r2Full.bottom - 20,
          left: r2Full.left + 20,
          right: r2Full.right - 20,
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
          verificarGameOver();
          return;
        }
      }

      if (poderX.value > window.innerWidth) {
        poderVisivel.value = false;
        return;
      }

      requestAnimationFrame(animLoop);
    }

    requestAnimationFrame(animLoop);
  }, 5000);

  //TEMPO MOEDA BRONZE
  setTimeout(() => {
    mostrarMoeda.value = true;
    moedaAnimacao = setInterval(() => {
      moedaFrame.value = moedaFrame.value === 4 ? 1 : moedaFrame.value + 1;
    }, 150); // troca o frame a cada 150ms
  }, 7000); // começa após 7 segundos
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

function iniciarPergunta() {
  mostrarPergunta.value = true;
  respostaDigitada.value = "";
  tempoRestante.value = 10;

  perguntaPausandoJogo.value = true;
  jogoPausado.value = true;

  // Toca som do relógio
  if (somRelogio.value && somAtivo.value) {
    somRelogio.value.currentTime = 0;
    somRelogio.value.play().catch(() => {});
  }

  timerPergunta = setInterval(() => {
    tempoRestante.value--;
    if (tempoRestante.value <= 0) {
      encerrarPergunta(false);
    }
  }, 1000);
}

function encerrarPergunta(acertou) {
  if (!mostrarPergunta.value) return;
  clearInterval(timerPergunta);
  mostrarPergunta.value = false;
  perguntaPausandoJogo.value = false;

  if (somRelogio.value) somRelogio.value.pause();

  if (somAtivo.value) {
    if (acertou && somAcerto.value) {
      somAcerto.value.currentTime = 0;
      somAcerto.value.play().catch(() => {});
    } else if (!acertou && somPerda.value) {
      somPerda.value.currentTime = 0;
      somPerda.value.play().catch(() => {});
      const idx = vidas.findIndex((v) => v);
      if (idx !== -1) vidas[idx] = false;
      verificarGameOver();
    }
  }

  if (!grounded.value) {
    jumpY.value = 0;
    grounded.value = true;
    velocityY = 0;
  }

  // ✅ Libera o jogo imediatamente
  jogoPausado.value = false;
  frameLoop = requestAnimationFrame(gameLoop);

  // ⏱️ Mostra a moeda prata após 2s
  setTimeout(() => {
    mostrarMoedaPrata.value = true;
    moedaPrataFrame.value = 1;
    setInterval(() => {
      moedaPrataFrame.value = moedaPrataFrame.value === 4 ? 1 : moedaPrataFrame.value + 1;
    }, 150);
  }, 2000);
}


function iniciarPerguntaDourada() {
  mostrarPerguntaDourada.value = true;
  respostaDigitada.value = "";
  tempoRestante.value = 10;

  perguntaPausandoJogo.value = true;
  jogoPausado.value = true;

  if (somRelogio.value && somAtivo.value) {
    somRelogio.value.currentTime = 0;
    somRelogio.value.play().catch(() => {});
  }

  timerPergunta = setInterval(() => {
    tempoRestante.value--;
    if (tempoRestante.value <= 0) {
      encerrarPerguntaDourada(false);
    }
  }, 1000);
}

function encerrarPerguntaDourada(acertou) {
  if (!mostrarPerguntaDourada.value) return;
  clearInterval(timerPergunta);
  mostrarPerguntaDourada.value = false;
  perguntaPausandoJogo.value = false;

  if (somRelogio.value) somRelogio.value.pause();

  if (somAtivo.value) {
    if (acertou && somAcerto.value) {
      somAcerto.value.currentTime = 0;
      somAcerto.value.play().catch(() => {});
    } else if (!acertou && somPerda.value) {
      somPerda.value.currentTime = 0;
      somPerda.value.play().catch(() => {});
      let perdidas = 0;
      for (let i = 0; i < vidas.length && perdidas < 3; i++) {
        if (vidas[i]) {
          vidas[i] = false;
          perdidas++;
        }
      }
      verificarGameOver();
    }
  }

  // ✅ Libera o jogo na hora
  jogoPausado.value = false;
  frameLoop = requestAnimationFrame(gameLoop);
}

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
  bottom: -85px; /* ou ajuste fino conforme necessário */
  right: 70px; /* ou ajuste fino conforme necessário */
  width: 320px; /* <<< aqui você aumenta o tamanho da sombra */
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

.img-game-over {
  width: 100%;
  max-width: 700px;
  image-rendering: pixelated;
  pointer-events: none;
}

.btn-reiniciar {
  position: absolute;
  bottom: px;
  top: 53%;
  left: 45%;
  font-family: "Press Start 2P", monospace;
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
  font-family: "Press Start 2P", monospace;
  color: white;
  font-size: 20px;
}

.moeda-girando {
  position: absolute;
  top: 30px;
  left: 50%;
  transform: translateX(-50%);
  width: 180px;
  height: auto;
  image-rendering: pixelated;
  z-index: 10;
}

.pergunta-overlay {
  position: fixed;
  inset: 0;
  background-color: rgba(0, 0, 0, 0.85);
  display: flex;
  justify-content: center;
  align-items: center;
  flex-direction: column;
  z-index: 9998;
}

.img-pergunta {
  width: 500px;
  image-rendering: pixelated;
  pointer-events: none;
}

.contador {
  font-family: "Press Start 2P", monospace;
  font-size: 28px;
  color: #00ff88;
  margin-top: 30px;
}

.moeda-girando-dourada {
  width: 190px;
  height: auto;
  position: absolute;
  left: 30%;
  bottom: 600px;
  z-index: 10;
}
</style>
