<template>
  <div>
    <!-- Tela de Menu -->
    <Menu v-if="telaAtual === 'menu'" @start="exibirIntro" />

    <!-- HQ de Introdução -->
    <div v-else-if="telaAtual === 'intro'" class="intro-hq">
      <img src="/hq-intro.png" class="hq-img" alt="HQ Intro" />
      <button class="btn-skip" @click="pularIntro">PULAR</button>
    </div>

    <!-- Tela de Jogo -->
    <div
      v-else-if="telaAtual === 'jogo'"
      class="cenario"
      @click="onCenarioClick"
    >
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
      <div
        v-if="mostrarPergunta"
        class="pergunta-overlay"
        @click.stop
      >
        <img src="/imgPerguntaBronze.png" class="img-pergunta" />
        <div class="contador">{{ tempoRestAnte }}</div>
      </div>

      <div
        v-if="mostrarPerguntaPrata"
        class="pergunta-overlay"
        @click.stop
      >
        <img src="/perguntaPrata.png" class="img-pergunta" />
        <div class="contador">{{ tempoRestAnte }}</div>
      </div>

      <div
        v-if="mostrarPerguntaDourada"
        class="pergunta-overlay"
        @click.stop
      >
        <img src="/perguntaDourada.png" class="img-pergunta" />
        <div class="contador">{{ tempoRestAnte }}</div>
      </div>

      <!-- Player -->
      <div
        class="player-wrapper"
        :style="{ left: playerX + 'px', bottom: estaAgachado ? '-45px' : jumpY + 'px' }"
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

      <!-- Botão de Som -->
      <button @click="toggleSom" class="btn-som">
        <img
          :src="somAtivo ? '/iconSomLigado.png' : '/iconSomDesligado.png'"
          alt="Som"
        />
      </button>

      <!-- Overlay de Pause -->
      <div v-if="jogoPausado && !perguntaPausandoJogo" class="pause-overlay">
        <img src="/telaPause.png" class="img-pause" alt="Pausado" />
        <button class="btn-continuar" @click.stop="togglePause">
          CONTINUAR
        </button>
      </div>

      <!-- Tela de Game Over -->
      <div v-if="gameOver" class="game-over-overlay">
        <div class="game-over-container">
          <img src="/imgGameOver.png" class="img-game-over" alt="Game Over" />
          <button class="btn-reiniciar" @click="reiniciarJogo">
            REINICIAR
          </button>
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

// ──────────────────────────────────────────────────────────────
// Estados principais
// ──────────────────────────────────────────────────────────────
const telaAtual = ref("menu"); // "menu" | "intro" | "jogo"
let introTimeoutId = null;
const vidas = reactive([true, true, true]);
const playerX = ref(50);
const jumpY = ref(0);
const playerSrc = ref("/player.png");
const bossSrc = ref("/boss.png");
const poderX = ref(0);
const poderVisivel = ref(false);
const invulneravel = ref(false);
const somNivel1 = ref(null);
const somImpacto = ref(null);
const somGameOver = ref(null);
const somAgachando = ref(null);
const somPulo = ref(null);
const somMoeda = ref(null);
const somAcerto = ref(null);
const somPerda = ref(null);
const somRelogio = ref(null);
const somAtivo = ref(false);

const jogoPausado = ref(false);
const gameOver = ref(false);

const moedaFrame = ref(1);
const mostrarMoeda = ref(false);
const moedaPrataFrame = ref(1);
const mostrarMoedaPrata = ref(false);
const moedaDouradaFrame = ref(1);
const mostrarMoedaDourada = ref(false);

const mostrarPergunta = ref(false);
const mostrarPerguntaPrata = ref(false);
const mostrarPerguntaDourada = ref(false);
const respostaDigitada = ref("");
const tempoRestAnte = ref(10);
const perguntaPausandoJogo = ref(false);

const estaAgachado = ref(false);
const direcao = ref("direita");

const speed = 5;
const jumpForce = 30;
const gravity = 0.8;
const grounded = ref(true);

let velocityY = 0;
let frameLoop = null;
let bossAnim = null;
let poderLoop = null;
let moedaAnimacao = null;
let animacaoPrata = null;
let animacaoDourada = null;
let timerPergunta = null;
const poderAnims = [];
const moving = { left: false, right: false, down: false };


// ──────────────────────────────────────────────────────────────
// Exibe a HQ antes de iniciar o jogo
// ──────────────────────────────────────────────────────────────
function exibirIntro() {
  telaAtual.value = "intro";
  clearTimeout(introTimeoutId);
  introTimeoutId = setTimeout(() => {
    if (telaAtual.value === "intro") iniciarJogo();
  }, 8000);
}

// Pula a HQ e inicia imediatamente
function pularIntro() {
  clearTimeout(introTimeoutId);
  iniciarJogo();
}

// ──────────────────────────────────────────────────────────────
// Inicia o jogo (animações, loop, sons, etc)
// ──────────────────────────────────────────────────────────────
function iniciarJogo() {
  telaAtual.value = "jogo";

  if (somNivel1.value) {
    somNivel1.value.currentTime = 0;
    somNivel1.value.loop = true;
    somNivel1.value.play().catch(() => {});
  }
  somAtivo.value = true;
  gameOver.value = false;

  // Reset de estado
  vidas.splice(0, vidas.length, true, true, true);
  playerX.value = 50;
  jumpY.value = 0;
  poderX.value = 0;
  poderVisivel.value = false;
  moedaFrame.value = 1;
  moedaPrataFrame.value = 1;
  moedaDouradaFrame.value = 1;
  mostrarMoeda.value = false;
  mostrarMoedaPrata.value = false;
  mostrarMoedaDourada.value = false;
  mostrarPergunta.value = false;
  mostrarPerguntaPrata.value = false;
  mostrarPerguntaDourada.value = false;
  respostaDigitada.value = "";
  tempoRestAnte.value = 10;
  estaAgachado.value = false;
  grounded.value = true;
  velocityY = 0;

  window.addEventListener("keydown", onKeyDown);
  window.addEventListener("keyup", onKeyUp);
  frameLoop = requestAnimationFrame(gameLoop);

  bossAnim = setInterval(() => {
    bossSrc.value = bossSrc.value.endsWith("2.png") ? "/boss.png" : "/boss2.png";
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

      if (pEl && pl && podePerder && !invulneravel.value) {
  const r1Full = pEl.getBoundingClientRect();
  // ...
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

  // Mostra moeda bronze após 7s
  setTimeout(() => {
    mostrarMoeda.value = true;
    moedaAnimacao = setInterval(() => {
      moedaFrame.value = moedaFrame.value === 4 ? 1 : moedaFrame.value + 1;
    }, 150);
  }, 7000);
}

// ──────────────────────────────────────────────────────────────
// Tratamento de teclas
// ──────────────────────────────────────────────────────────────
function onKeyDown(e) {
  if (jogoPausado.value || gameOver.value) return;

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
    exibirIntro();
  }

  // Respostas das perguntas
  if (mostrarPergunta.value && /^[0-9]$/.test(e.key)) {
    respostaDigitada.value += e.key;
    if (respostaDigitada.value === "7") encerrarPergunta(true);
    else if (respostaDigitada.value.length >= 2) encerrarPergunta(false);
  }
  if (mostrarPerguntaPrata.value && /^[0-9]$/.test(e.key)) {
    respostaDigitada.value += e.key;
    if (respostaDigitada.value === "8") encerrarPerguntaPrata(true);
    else if (respostaDigitada.value.length >= 2)
      encerrarPerguntaPrata(false);
  }
  if (mostrarPerguntaDourada.value && /^[a-zA-Z]$/.test(e.key)) {
    respostaDigitada.value += e.key.toLowerCase();
    if (respostaDigitada.value.toLowerCase() === "x")
      encerrarPerguntaDourada(true);
    else if (respostaDigitada.value.length >= 2)
      encerrarPerguntaDourada(false);
  }

  if (e.key === "Escape") togglePause();
}

function onKeyUp(e) {
  if (jogoPausado.value || gameOver.value) return;
  if (e.key === "a") moving.left = false;
  if (e.key === "d") moving.right = false;
  if (e.key === "s") {
    moving.down = false;
    if (somAgachando.value) {
      somAgachando.value.pause();
      somAgachando.value.currentTime = 0;
    }
  }
}

// ──────────────────────────────────────────────────────────────
// Clique no cenário (para pausar / despausar)
// ──────────────────────────────────────────────────────────────
function onCenarioClick() {
  // Se alguma pergunta estiver ativa, não dispara togglePause
  if (
    mostrarPergunta.value ||
    mostrarPerguntaPrata.value ||
    mostrarPerguntaDourada.value
  ) {
    return;
  }
  togglePause();
}

// ──────────────────────────────────────────────────────────────
// Loop principal do jogo
// ──────────────────────────────────────────────────────────────
function gameLoop() {
  if (jogoPausado.value || perguntaPausandoJogo.value || gameOver.value)
    return;

  // Movimento horizontal
  if (moving.left) playerX.value = Math.max(0, playerX.value - speed);
  if (moving.right)
    playerX.value = Math.min(window.innerWidth - 100, playerX.value + speed);

  // ─── LÓGICA CORRIGIDA DE PULO, GRAVIDADE E SPRITE ───
  if (!grounded.value) {
    // Subtrai gravidade para desacelerar subida e iniciar queda
    velocityY -= gravity;
    jumpY.value += velocityY;

    // Enquanto estiver no ar, usa o sprite de pulo
    playerSrc.value =
      direcao.value === "esquerda" ? "/jumpVoltando.jpg" : "/playerGiro.png";
    estaAgachado.value = false;

    // Se bateu no chão (jumpY ≤ 0), reseta tudo
    if (jumpY.value <= 0) {
      jumpY.value = 0;
      velocityY = 0;
      grounded.value = true;
    }
  } else {
    // Está no chão, garante que jumpY fique em zero
    jumpY.value = 0;

    // Agora, se estiver se movendo para baixo (agachar), troca o sprite:
    if (moving.down) {
      estaAgachado.value = true;
      playerSrc.value =
        moving.left || direcao.value === "esquerda"
          ? "/agachado2.png"
          : "/agachado.png";
    }
    // Se estiver andando para a esquerda
    else if (moving.left) {
      estaAgachado.value = false;
      playerSrc.value = "/bonecoVoltando.jpg";
    }
    // Se estiver andando para a direita
    else if (moving.right) {
      estaAgachado.value = false;
      playerSrc.value = "/player.png";
    }
    // Caso parado em pé (nem se movendo nem agachado)
    else {
      estaAgachado.value = false;
      playerSrc.value =
        direcao.value === "esquerda" ? "/bonecoVoltando.jpg" : "/player.png";
    }
  }
  // ────────────────────────────────────────────────────────

  // Colisão moedas
  const checarColisaoMoeda = (selector, onCollect) => {
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
      if (colidiu) onCollect();
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
    });
  }

  frameLoop = requestAnimationFrame(gameLoop);
}

// ──────────────────────────────────────────────────────────────
// Pergunta Bronze
// ──────────────────────────────────────────────────────────────
function iniciarPergunta() {
  mostrarPergunta.value = true;
  respostaDigitada.value = "";
  tempoRestAnte.value = 10;
  perguntaPausandoJogo.value = true;
  // NÃO colocamos jogoPausado = true aqui, pois já está pausado por perguntaPausandoJogo
  if (somRelogio.value && somAtivo.value) {
    somRelogio.value.currentTime = 0;
    somRelogio.value.play().catch(() => {});
  }

  timerPergunta = setInterval(() => {
    tempoRestAnte.value--;
    if (tempoRestAnte.value <= 0) {
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

  // Libera o jogo imediatamente
  // Não setamos jogoPausado aqui; já estava false após perguntaPausandoJogo false
  frameLoop = requestAnimationFrame(gameLoop);

  // Mostra moeda prata após 2s
  setTimeout(() => {
    mostrarMoedaPrata.value = true;
    moedaPrataFrame.value = 1;
    animacaoPrata = setInterval(() => {
      moedaPrataFrame.value =
        moedaPrataFrame.value === 4 ? 1 : moedaPrataFrame.value + 1;
    }, 150);
  }, 2000);
}

// ──────────────────────────────────────────────────────────────
// Pergunta Prata
// ──────────────────────────────────────────────────────────────
function iniciarPerguntaPrata() {
  mostrarPerguntaPrata.value = true;
  respostaDigitada.value = "";
  tempoRestAnte.value = 10;
  perguntaPausandoJogo.value = true;
  // NÃO setamos jogoPausado aqui.
  if (somRelogio.value && somAtivo.value) {
    somRelogio.value.currentTime = 0;
    somRelogio.value.play().catch(() => {});
  }

  timerPergunta = setInterval(() => {
    tempoRestAnte.value--;
    if (tempoRestAnte.value <= 0) {
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

  frameLoop = requestAnimationFrame(gameLoop);

  // Mostra moeda dourada após 4s
  setTimeout(() => {
    mostrarMoedaDourada.value = true;
    moedaDouradaFrame.value = 1;
    animacaoDourada = setInterval(() => {
      moedaDouradaFrame.value =
        moedaDouradaFrame.value === 4 ? 1 : moedaDouradaFrame.value + 1;
    }, 150);
  }, 4000);
}

// ──────────────────────────────────────────────────────────────
// Pergunta Dourada
// ──────────────────────────────────────────────────────────────
function iniciarPerguntaDourada() {
  mostrarPerguntaDourada.value = true;
  respostaDigitada.value = "";
  tempoRestAnte.value = 10;
  perguntaPausandoJogo.value = true;
  // NÃO setamos jogoPausado aqui.
  if (somRelogio.value && somAtivo.value) {
    somRelogio.value.currentTime = 0;
    somRelogio.value.play().catch(() => {});
  }

  timerPergunta = setInterval(() => {
    tempoRestAnte.value--;
    if (tempoRestAnte.value <= 0) {
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

  frameLoop = requestAnimationFrame(gameLoop);
}

// ──────────────────────────────────────────────────────────────
// Verifica Game Over
// ──────────────────────────────────────────────────────────────
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

// ──────────────────────────────────────────────────────────────
// Reiniciar Jogo
// ──────────────────────────────────────────────────────────────
function reiniciarJogo() {
  vidas.splice(0, vidas.length, true, true, true);
  mostrarMoeda.value = false;
  mostrarMoedaPrata.value = false;
  mostrarMoedaDourada.value = false;
  mostrarPergunta.value = false;
  mostrarPerguntaPrata.value = false;
  mostrarPerguntaDourada.value = false;
  respostaDigitada.value = "";
  tempoRestAnte.value = 10;
  playerX.value = 50;
  jumpY.value = 0;
  gameOver.value = false;
  telaAtual.value = "menu";
  setTimeout(() => {
    telaAtual.value = "jogo";
  }, 50);
}

// ──────────────────────────────────────────────────────────────
// Limpa tudo ao sair
// ──────────────────────────────────────────────────────────────
function limparJogo() {
  window.removeEventListener("keydown", onKeyDown);
  window.removeEventListener("keyup", onKeyUp);
  if (frameLoop) cancelAnimationFrame(frameLoop);
  if (bossAnim) clearInterval(bossAnim);
  if (poderLoop) clearInterval(poderLoop);
  poderAnims.forEach((id) => clearInterval(id));
  poderAnims.length = 0;
  if (somNivel1.value) somNivel1.value.pause();

  //faz pausar todos os sons qnd vier a tela de gameover
  [somImpacto, somAgachando, somRelogio, somPulo, somMoeda, somAcerto, somPerda].forEach((som) => {
  if (som.value) {
    som.value.pause();
    som.value.currentTime = 0;
  }
});
}

// ──────────────────────────────────────────────────────────────
// Liga/desliga som
// ──────────────────────────────────────────────────────────────
function toggleSom() {
  if (!somNivel1.value) return;
  somAtivo.value = !somAtivo.value;
  if (somAtivo.value) somNivel1.value.play().catch(() => {});
  else somNivel1.value.pause();
}

// ──────────────────────────────────────────────────────────────
// Liga/desliga pause
// ──────────────────────────────────────────────────────────────
function togglePause() {
  jogoPausado.value = !jogoPausado.value;
  if (jogoPausado.value) {
    if (somNivel1.value) somNivel1.value.pause();
    if (timerPergunta) clearInterval(timerPergunta);
  } else {
    if (somAtivo.value && somNivel1.value) somNivel1.value.play().catch(() => {});
    if (!perguntaPausandoJogo.value) frameLoop = requestAnimationFrame(gameLoop);
  }
}

// ──────────────────────────────────────────────────────────────
// “Watcher” para trocar de tela
// ──────────────────────────────────────────────────────────────
watch(telaAtual, (t) => {
  if (t === "jogo") iniciarJogo();
  else limparJogo();
});

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
  bottom: -134px; /* ajuste conforme necessário */
  right: 70px;    /* ajuste conforme necessário */
  width: 320px;   /* ajuste conforme necessário */
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
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.9);
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  z-index: 9998;
}

.img-pause {
  max-width: 700px;
  image-rendering: pixelated;
  pointer-events: none;
}

.btn-continuar {
  margin-top: 100px; /* Aumente esse valor se ainda estiver muito acima */
  font-family: "Press Start 2P", monospace;
  font-size: 16px;
  background: limegreen;
  color: white;
  padding: 16px 32px;
  border: 4px solid black;
  box-shadow: 4px 4px black;
  cursor: pointer;
  transition: transform 0.1s;
  z-index: 1000;
}

.btn-continuar:hover {
  transform: scale(1.05);
  background: green;
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

.moeda-girando-dourada {
  width: 190px;
  height: auto;
  position: absolute;
  left: 30%;
  bottom: 600px;
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

.intro-hq {
  position: fixed;
  inset: 0;
  background-color: black;
  display: flex;
  justify-content: center;
  align-items: center;
  flex-direction: column;
  z-index: 9999;
}

.hq-img {
  max-width: 100%;
  max-height: 100vh;
  image-rendering: pixelated;
}

.btn-skip {
  margin-top: 20px;
  font-family: 'Press Start 2P', monospace;
  font-size: 14px;
  padding: 12px 24px;
  background: red;
  color: white;
  border: 4px solid black;
  box-shadow: 4px 4px black;
  cursor: pointer;
}
</style>






