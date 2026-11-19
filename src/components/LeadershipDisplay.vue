<script setup>
  import { ref, onUnmounted, computed, watch } from "vue";

  import fanfare from "../assets/sounds/bytt-ar-bytt.mp3";
  import logo from "../assets/images/logo-black.png";
  import leadershipObject from "../assets/images/subject.png";

  const props = defineProps({
    amounts: {
      type: Array,
      required: true,
    },
    command: {
      type: Object,
      required: true,
    },
  });

  const selectedAmount = 500000;

  const phase = ref("ready");
  const activeId = ref(null);
  const isFullscreen = ref(false);

  const winner = computed(() =>
    props.amounts.find((a) => a.value === selectedAmount),
  );

  const fanfareAudio = new Audio(fanfare);
  fanfareAudio.preload = "auto";

  const FANFARE_DURATION = 9500;
  const IMAGE_DELAY = 1300;

  const audioReady = ref(false);

  let interval = null;
  let timeout = null;
  let imageTimeout = null;
  let currentIndex = 0;

  function formatAmount(value) {
    return new Intl.NumberFormat("sv-SE").format(value);
  }

  function enableAudio() {
    fanfareAudio
      .play()
      .then(() => {
        fanfareAudio.pause();
        fanfareAudio.currentTime = 0;
        audioReady.value = true;
      })
      .catch((error) => {
        console.warn("Kunde inte aktivera ljudet:", error);
      });
  }

  function stopTimers() {
    clearInterval(interval);
    clearTimeout(timeout);
    clearTimeout(imageTimeout);

    interval = null;
    timeout = null;
    imageTimeout = null;
  }

  function stopFanfare() {
    fanfareAudio.pause();
    fanfareAudio.currentTime = 0;
  }

  function startReveal() {
    if (phase.value === "rolling" || phase.value === "selected") return;

    if (!winner.value) {
      console.error("Hittade inte valören 500 000 kr.");
      return;
    }

    stopTimers();
    stopFanfare();

    phase.value = "rolling";
    currentIndex = 0;

    // Spela fanfaren om ljudet har aktiverats på projektionsdatorn
    if (audioReady.value) {
      fanfareAudio.currentTime = 0;

      fanfareAudio.play().catch((error) => {
        console.warn("Kunde inte spela fanfaren:", error);
      });
    } else {
      console.warn("Ljudet är inte aktiverat på projektionsdatorn.");
    }

    // Börja blinka efter 750 ms
    timeout = setTimeout(() => {
      activeId.value = props.amounts[0]?.id ?? null;

      interval = setInterval(() => {
        currentIndex = (currentIndex + 1) % props.amounts.length;
        activeId.value = props.amounts[currentIndex].id;
      }, 140);
    }, 750);

    // Stanna på 500 000 kr efter 9,5 sekunder
    imageTimeout = setTimeout(() => {
      clearInterval(interval);
      interval = null;

      activeId.value = winner.value.id;
      phase.value = "selected";

      // Visa föremålet efter ytterligare 1,3 sekunder
      timeout = setTimeout(() => {
        phase.value = "revealed";
        activeId.value = null;
      }, IMAGE_DELAY);
    }, FANFARE_DURATION);
  }

  function resetReveal() {
    stopTimers();
    stopFanfare();

    phase.value = "ready";
    activeId.value = null;
  }

  let lastCommandId = props.command?.id ?? 0;

  watch(
    () => props.command,
    (newCommand) => {
      if (!newCommand?.id || newCommand.id <= lastCommandId) return;

      lastCommandId = newCommand.id;

      if (newCommand.action === "start") {
        startReveal();
      }

      if (newCommand.action === "reset") {
        resetReveal();
      }
    },
    { deep: true },
  );

  async function toggleFullscreen() {
    try {
      if (!document.fullscreenElement) {
        await document.documentElement.requestFullscreen();
      } else {
        await document.exitFullscreen();
      }
    } catch (error) {
      console.warn("Kunde inte aktivera fullskärm:", error);
    }
  }

  function updateFullscreenState() {
    isFullscreen.value = !!document.fullscreenElement;
  }

  document.addEventListener("fullscreenchange", updateFullscreenState);

  onUnmounted(() => {
    stopTimers();
    stopFanfare();
    document.removeEventListener("fullscreenchange", updateFullscreenState);
  });
</script>

<template>
  <main class="leadership-screen">
    <div class="leadership-controls">
      <button @click="toggleFullscreen">
        {{ isFullscreen ? "X" : "⛶" }}
      </button>
    </div>

    <section class="leadership-layout">
      <!-- VÄNSTER: LOGGA / FÖREMÅL -->
      <div class="leadership-left">
        <Transition name="object-reveal" mode="out-in">
          <div
            v-if="phase === 'revealed'"
            key="object"
            class="object-container"
          >
            <img
              :src="leadershipObject"
              alt="Ledningsgruppens utvalda föremål"
              class="leadership-object-image"
            />

            <p class="object-heading">LEDNINGSGRUPPEN HADE RÄTT OM...</p>
            <h2>500 000 KR</h2>
            <p class="result-caption">LEDNINGSGRUPPENS VAL</p>
          </div>

          <div v-else key="logo" class="logo-container">
            <img :src="logo" alt="BYTT är BYTT" class="leadership-logo" />
            <div class="leadership-line"></div>
            <p>LEDNINGSGRUPPENS GISSNING</p>
          </div>
        </Transition>
      </div>

      <!-- HÖGER: BELOPP -->
      <div
        class="leadership-board"
        :class="{ 'board-revealed': phase === 'revealed' }"
      >
        <div
          v-for="amount in props.amounts"
          :key="amount.id"
          class="leadership-amount"
          :class="{
            'rolling-highlight': phase === 'rolling' && activeId === amount.id,
            'winner-highlight':
              (phase === 'selected' || phase === 'revealed') &&
              winner?.id === amount.id,
            dimmed: phase === 'revealed' && winner?.id !== amount.id,
          }"
        >
          <span>{{ formatAmount(amount.value) }} kr</span>
        </div>
      </div>
    </section>

    <button class="sound-toggle" v-if="!audioReady" @click="enableAudio">
      🔊
    </button>
  </main>
</template>

<style scoped>
  .leadership-screen {
    width: 100vw;
    height: 100vh;
    min-height: 100vh;
    padding: 6vh 5vw;
    overflow: hidden;
    position: relative;

    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;

    background:
      radial-gradient(
        ellipse at 18% 30%,
        rgba(255, 224, 207, 0.65),
        transparent 40%
      ),
      radial-gradient(
        ellipse at 85% 85%,
        rgba(190, 100, 99, 0.35),
        transparent 45%
      ),
      linear-gradient(135deg, #f3b2a1 0%, #e99b8d 45%, #d98179 100%);
  }

  .leadership-controls {
    position: absolute;
    top: 20px;
    left: 20px;
    right: 20px;
    display: flex;
    justify-content: space-between;
    z-index: 10;
  }

  .leadership-controls button {
    padding: 10px 18px;
    border: 1px solid rgba(255, 255, 255, 0.5);
    border-radius: 30px;
    background: rgba(0, 0, 0, 0.2);
    color: white;
    cursor: pointer;
  }

  .leadership-layout {
    width: 100%;
    max-width: 1500px;
    height: 80vh;
    display: grid;
    grid-template-columns: minmax(0, 0.85fr) minmax(0, 1.15fr);
    align-items: center;
    gap: 5vw;
  }

  .leadership-left {
    min-width: 0;
    height: 100%;
    display: flex;
    align-items: center;
    justify-content: center;
    text-align: center;
  }

  .logo-container,
  .object-container {
    width: 100%;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 18px;
  }

  .leadership-logo {
    width: min(360px, 100%);
    max-height: 35vh;
    object-fit: contain;
  }

  .leadership-line {
    width: 80px;
    height: 2px;
    background: #171313;
  }

  .logo-container p {
    font-size: clamp(11px, 1vw, 15px);
    letter-spacing: 3px;
    font-weight: 700;
  }

  .object-heading {
    font-size: clamp(12px, 1.2vw, 18px);
    font-weight: 700;
    letter-spacing: 2px;
    margin: 0;
  }

  .leadership-object-image {
    width: 100%;
    max-height: 42vh;
    object-fit: contain;
    border-radius: 10px;
    filter: drop-shadow(0 15px 30px rgba(65, 28, 25, 0.25));
  }

  .object-container h2 {
    font-size: clamp(38px, 5vw, 72px);
    margin: 0;
    font-weight: 800;
  }

  .result-caption {
    font-size: 11px;
    letter-spacing: 3px;
    font-weight: 700;
    margin: 0;
  }

  .leadership-board {
    width: 100%;
    height: 100%;
    max-height: 720px;
    padding: 24px;

    display: grid;
    grid-template-columns: 1fr;
    grid-auto-rows: minmax(0, 1fr);
    gap: 9px;

    background: linear-gradient(145deg, #242323, #080808);
    border: 1px solid rgba(255, 255, 255, 0.12);
    border-radius: 10px;
    box-shadow: 0 25px 60px rgba(65, 28, 25, 0.28);

    transition: opacity 0.5s ease;
  }

  .leadership-amount {
    display: flex;
    align-items: center;
    justify-content: center;

    background: linear-gradient(135deg, #353333, #242323);
    border: 1px solid rgba(255, 255, 255, 0.1);
    border-radius: 5px;
    color: white;

    font-size: clamp(18px, 2.2vw, 34px);
    font-weight: 600;

    transition: all 0.35s ease;
  }

  .leadership-amount.rolling-highlight {
    background: #e99b8d;
    color: #080808;
    border-color: white;
    transform: scale(1.04);
    z-index: 2;
    box-shadow: 0 0 25px rgba(255, 255, 255, 0.4);
  }

  .leadership-amount.winner-highlight {
    background: white;
    color: #080808;
    border-color: #e99b8d;
    transform: scale(1.1);
    z-index: 3;
    box-shadow: 0 0 40px rgba(0, 0, 0, 0.35);
  }

  .leadership-amount.dimmed {
    opacity: 0.15;
    transform: scale(0.95);
  }

  .leadership-instruction {
    position: absolute;
    bottom: 20px;
    font-size: 14px;
    letter-spacing: 1px;
    color: #241918;
  }

  .object-reveal-enter-active,
  .object-reveal-leave-active {
    transition: all 0.7s ease;
  }

  .object-reveal-enter-from,
  .object-reveal-leave-to {
    opacity: 0;
    transform: scale(0.85);
  }

  @media (max-width: 750px) {
    .leadership-layout {
      grid-template-columns: 1fr 1.2fr;
      gap: 3vw;
    }

    .leadership-board {
      padding: 12px;
      gap: 6px;
    }

    .leadership-amount {
      font-size: clamp(14px, 2.5vw, 22px);
    }

    .leadership-logo {
      width: 90%;
    }
  }

  @media (max-height: 750px) {
    .leadership-layout {
      height: 76vh;
    }

    .leadership-board {
      padding: 14px;
      gap: 6px;
    }

    .leadership-logo {
      max-height: 25vh;
    }

    .leadership-object-image {
      max-height: 32vh;
    }
  }
</style>
