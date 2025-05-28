<template>
  <div>
    <!-- Menu -->
    <div v-if="telaAtual === 'menu'" class="tela-menu">
      <img src="/menu.png" alt="Tela de Menu" class="img-menu" />

      <!-- Brilhos -->
      <img src="/brilhos.png" class="brilho" style="top: 40px; left: 690px;" />
      <img src="/brilhos.png" class="brilho" style="top: 38px; left: 910px;" />
      <img src="/brilhos.png" class="brilho" style="top: 36px; left: 1060px;" />
      <img src="/brilhos.png" class="brilho" style="top: 36px; left: 1170px;" />
      <img src="/brilhos.png" class="brilho" style="top: 123px; left: 1165px;" />

      <!-- Binários nas laterais -->
      <div class="binarios-esquerda">
        <span
          v-for="(bit, i) in bits.slice(0, bits.length / 2)"
          :key="'esq-' + i"
          class="bit"
          :style="{ left: bit.left + 'px', animationDelay: bit.delay + 's', color: bit.color }"
        >{{ bit.value }}</span>
      </div>
      <div class="binarios-direita">
        <span
          v-for="(bit, i) in bits.slice(bits.length / 2)"
          :key="'dir-' + i"
          class="bit"
          :style="{ left: bit.left + 'px', animationDelay: bit.delay + 's', color: bit.color }"
        >{{ bit.value }}</span>
      </div>

      <button class="botao" @click="telaAtual = 'jogo'">Iniciar Jogo</button>
    </div>

    <!-- Jogo -->
    <div v-else class="cenario">
      <div class="hud">
        <img
          v-for="(vida, i) in vidas"
          :key="i"
          :src="vida ? '/coracao.png' : '/coracaoVazio.png'"
          class="coracao"
        />
      </div>

      <img :src="bossSrc" alt="Chefão" class="boss" />
      <img
        ref="player"
        :src="playerSrc"
        alt="Jogador"
        class="player"
        :style="{ left: playerX + 'px' }"
      />
      <img
        v-if="poderVisivel"
        ref="poder"
        src="/poder-binario.png"
        alt="Poder"
        class="poder"
        :style="{ right: (350 + poderX) + 'px' }"
      />
    </div>
  </div>
</template>

<script>
export default {
  name: "Game",
  data() {
    return {
      telaAtual: "menu",
      playerFrame: 1,
      playerSrc: "/player.png",
      playerX: 50,
      bossFrame: 1,
      bossSrc: "/boss.png",
      poderVisivel: false,
      poderX: 0,
      vidas: [true, true, true],
      podePerderVida: true,
      bits: []
    };
  },
  watch: {
    telaAtual(nova) {
      if (nova === "jogo") this.iniciarJogo();
    }
  },
  mounted() {
    const cores = [
      "#00ff00", "#ff00ff", "#00ffff", "#ffff00", "#ff4444", "#ff8800", "#4488ff"
    ];

    for (let i = 0; i < 300; i++) {
      const isEsquerda = i < 150;
      const faixaLargura = 350; // largura do feixe de queda
      const offset = 50; // margem interna

      this.bits.push({
        value: Math.round(Math.random()),
        left: Math.random() * faixaLargura + offset,
        delay: Math.random() * 3,
        color: cores[Math.floor(Math.random() * cores.length)]
      });
    }
  },
  methods: {
    iniciarJogo() {
      setInterval(() => {
        this.playerFrame = this.playerFrame === 1 ? 2 : 1;
        this.playerSrc = `/player${this.playerFrame === 1 ? "" : "2"}.png`;
      }, 300);

      setInterval(() => {
        this.bossFrame = this.bossFrame === 1 ? 2 : 1;
        this.bossSrc = `/boss${this.bossFrame === 1 ? "" : "2"}.png`;
      }, 300);

      this.atirarPoder();
      window.addEventListener("keydown", this.moverPlayer);
    },
    moverPlayer(e) {
      if (e.key === "ArrowRight") this.playerX += 20;
      if (e.key === "ArrowLeft") this.playerX -= 20;
    },
    atirarPoder() {
      setInterval(() => {
        this.poderVisivel = true;
        this.poderX = 0;
        this.podePerderVida = true;

        const animar = setInterval(() => {
          this.poderX += 10;

          const poder = this.$refs.poder;
          const player = this.$refs.player;

          if (poder && player) {
            const poderBox = poder.getBoundingClientRect();
            const playerBox = player.getBoundingClientRect();

            const colidiu =
              poderBox.left < playerBox.right &&
              poderBox.right > playerBox.left &&
              poderBox.top < playerBox.bottom &&
              poderBox.bottom > playerBox.top;

            if (colidiu && this.podePerderVida) {
              this.perderVida();
              this.podePerderVida = false;
            }
          }

          if (this.poderX > window.innerWidth) {
            clearInterval(animar);
            this.poderVisivel = false;
          }
        }, 50);
      }, 3000);
    },
    perderVida() {
      for (let i = 0; i < this.vidas.length; i++) {
        if (this.vidas[i]) {
          this.vidas[i] = false;
          break;
        }
      }
    }
  }
};
</script>

<style scoped>
/* Menu */
.tela-menu {
  position: relative;
  width: 100%;
  height: 100vh;
  background-color: #000;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}
.img-menu {
  width: 100%;
  height: 100%;
  object-fit: contain;
  image-rendering: pixelated;
}
.botao {
  position: absolute;
  bottom: 40px;
  left: 50%;
  transform: translateX(-50%);
  font-family: 'Press Start 2P', monospace;
  font-size: 14px;
  padding: 16px 24px;
  background-color: #ff4444;
  color: white;
  border: none;
  cursor: pointer;
  image-rendering: pixelated;
  box-shadow: 4px 4px 0 #000;
  z-index: 3;
}
.brilho {
  position: absolute;
  width: 32px;
  height: 32px;
  pointer-events: none;
  image-rendering: pixelated;
  animation: piscar 2s infinite ease-in-out;
  z-index: 4;
}
@keyframes piscar {
  0%, 100% { opacity: 0.2; transform: scale(0.9); }
  50% { opacity: 1; transform: scale(1.3); }
}

/* Binários */
.binarios-esquerda,
.binarios-direita {
  position: fixed;
  top: 0;
  width: 400px;
  height: 100vh;
  z-index: 1;
  overflow: hidden;
  pointer-events: none;
}
.binarios-esquerda { left: 0; }
.binarios-direita { right: 0; }

.bit {
  position: absolute;
  top: -30px;
  font-size: 20px;
  font-family: 'Courier New', monospace;
  animation: cair 2.5s linear infinite;
  opacity: 0.95;
  text-shadow:
    -1px -1px 0 #000,
     1px -1px 0 #000,
    -1px  1px 0 #000,
     1px  1px 0 #000;
}

@keyframes cair {
  0% { top: -30px; opacity: 0.1; }
  100% { top: 100vh; opacity: 1; }
}

/* Jogo */
.cenario {
  position: relative;
  width: 100%;
  height: 100vh;
  background: url("/cenario.png") no-repeat center center;
  background-size: cover;
  overflow: hidden;
}
.boss {
  position: absolute;
  bottom: 5%;
  right: 0%;
  width: 530px;
}
.player {
  position: absolute;
  bottom: 5%;
  width: 330px;
}
.poder {
  position: absolute;
  bottom: 15%;
  width: 300px;
}
.hud {
  position: absolute;
  top: 10px;
  left: 20px;
  display: flex;
  gap: 10px;
  z-index: 999;
}
.coracao {
  width: 40px;
}
</style>


