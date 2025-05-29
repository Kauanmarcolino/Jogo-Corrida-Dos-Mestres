<template>
  <div class="tela-menu">
    <!-- Mensagem no topo -->
    <div class="msg-f11">{{ textoTopo }}</div>

    <!-- Texto para iniciar -->
    <div class="msg-enter">Digite ENTER para iniciar</div>

    <!-- Imagem central -->
    <div class="menu-center">
      <img src="/menu.png" alt="Menu" class="img-menu" />
    </div>

    <!-- Botão de som -->
    <button @click="toggleSom" class="btn-som">
      <img
        :src="somAtivo ? '/iconSomLigado.png' : '/iconSomDesligado.png'"
        alt="Som"
      />
    </button>

    <!-- Música de fundo -->
    <audio ref="musica" src="/musica.mp3" autoplay loop />

    <!-- Som de entrada -->
    <audio ref="somEnter" src="/som-enter.mp3" />
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from "vue";
const emit = defineEmits(["start"]);

const somAtivo = ref(false);
const musica = ref(null);
const somEnter = ref(null);
const textoTopo = ref("F11 para uma melhor experiência =)");

let intervaloMensagem = null;

function toggleSom() {
  if (!musica.value) return;
  somAtivo.value = !somAtivo.value;
  somAtivo.value ? musica.value.play().catch(() => {}) : musica.value.pause();
}

function ativarSomAoClique() {
  if (musica.value && somAtivo.value) {
    musica.value.play().catch(() => {});
  }
  window.removeEventListener("click", ativarSomAoClique);
}

function handleTecla(e) {
  if (e.key === "Enter") {
    if (somEnter.value) {
      // Reduz o volume da música de fundo
      if (musica.value) {
        musica.value.volume = 0.1;
      }

      somEnter.value.currentTime = 0;
      somEnter.value.play().catch(() => {});

      // Espera o som terminar, depois +1s pra começar o jogo
      somEnter.value.onended = () => {
        setTimeout(() => {
          emit("start");
        }, 600); // 1 segundo após fim do som
      };
    } else {
      // fallback caso som não carregue
      emit("start");
    }
  }
}

// Alterna a mensagem quando estiver em fullscreen
function iniciarAlternanciaMensagem() {
  if (intervaloMensagem) return;

  intervaloMensagem = setInterval(() => {
    textoTopo.value =
      textoTopo.value === "F11 para uma melhor experiência =)"
        ? "Agora sim, tudo pronto!"
        : "F11 para uma melhor experiência =)";
  }, 3000);
}

function estaEmFullscreen() {
  return window.innerHeight === screen.height;
}

function verificarFullscreenPeriodicamente() {
  setInterval(() => {
    if (estaEmFullscreen()) {
      iniciarAlternanciaMensagem();
    }
  }, 1000);
}

onMounted(() => {
  if (musica.value) musica.value.volume = 0.4;
  if (somEnter.value) somEnter.value.volume = 1.0;

  window.addEventListener("click", ativarSomAoClique);
  window.addEventListener("keydown", handleTecla);
  verificarFullscreenPeriodicamente();
});

onBeforeUnmount(() => {
  window.removeEventListener("keydown", handleTecla);
  clearInterval(intervaloMensagem);
});
</script>

<style scoped>
@import url("https://fonts.googleapis.com/css2?family=Press+Start+2P&display=swap");

.tela-menu {
  display: flex;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  background: #000;
  justify-content: center;
  align-items: center;
  position: relative;
}

.menu-center {
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

/* Botão de som no canto direito */
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
  width: 60px;
  height: 60px;
  padding: 3em;
}

/* Mensagem RGB do topo */
.msg-f11 {
  position: absolute;
  top: 20px;
  left: 50%;
  transform: translateX(-50%);
  font-size: 14px;
  font-family: "Press Start 2P", monospace;
  font-weight: bold;
  z-index: 10;
  pointer-events: none;
  animation: corRGB 5s infinite linear;
  text-shadow: 1px 1px 0 #000;
}

/* Mensagem ENTER com fade fluido */
.msg-enter {
  position: absolute;
  bottom: 60px;
  left: 48%;
  transform: translateX(-50%);
  font-size: 16px;
  font-family: "Press Start 2P", monospace;
  z-index: 10;
  animation: fadeBlink 1.5s infinite ease-in-out;
  color: #ffffff;
  text-shadow: 1px 1px 0 #000;
}

/* RGB alternando */
@keyframes corRGB {
  0%   { color: #ff5555; }
  25%  { color: #55aaff; }
  50%  { color: #55ff55; }
  75%  { color: #ffff55; }
  100% { color: #ff5555; }
}

/* Fade suave piscando */
@keyframes fadeBlink {
  0%   { opacity: 1; }
  50%  { opacity: 0.2; }
  100% { opacity: 1; }
}
</style>

