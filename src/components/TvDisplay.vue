<script setup>
  import { ref, watch, onUnmounted } from "vue";
  import fanfare from "../assets/sounds/Grand-Theatral-Accent.mp3";
  import logo from "../assets/images/logo.png";

  const props = defineProps({
    amounts: {
      type: Array,
      required: true,
    },
  });

  const activeId = ref(null);
  const isRolling = ref(false);
  const winnerId = ref(null);
  const finalId = ref(null);

  const displayedRemoved = ref(
    new Set(props.amounts.filter((a) => a.removed).map((a) => a.id)),
  );

  let interval = null;
  let timeout = null;
  let currentIndex = 0;

  const soundEnabled = ref(false);
  const fanfareAudio = new Audio(fanfare);
  fanfareAudio.preload = "auto";

  const FANFARE_DURATION = 5260;

  function enableSound() {
    soundEnabled.value = true;
    fanfareAudio
      .play()
      .then(() => {
        fanfareAudio.pause();
        fanfareAudio.currentTime = 0;
      })
      .catch((error) => {
        console.warn("Kunde inte aktivera ljudet:", error);
      });
  }

  function playFanfare() {
    if (!soundEnabled.value) return;

    fanfareAudio.pause();
    fanfareAudio.currentTime = 0;

    fanfareAudio.play().catch((error) => {
      console.warn("Kunde inte spela fanfaren:", error);
    });
  }

  function stopFanfare() {
    fanfareAudio.pause();
    fanfareAudio.currentTime = 0;
  }

  function formatAmount(value) {
    return new Intl.NumberFormat("sv-SE").format(value);
  }

  function stopAnimation() {
    clearInterval(interval);
    clearTimeout(timeout);

    interval = null;
    timeout = null;

    stopFanfare();
  }

  function playRemovalAnimation(id) {
    stopAnimation();

    isRolling.value = true;
    winnerId.value = null;

    const remaining = props.amounts.filter((a) => !a.removed || a.id === id);

    if (!remaining.length) return;

    const isFinalRound = remaining.length === 2;

    const finalAmount = isFinalRound
      ? remaining.find((a) => a.id !== id)
      : null;

    currentIndex = 0;
    activeId.value = remaining[0].id;

    playFanfare();

    interval = setInterval(() => {
      currentIndex = (currentIndex + 1) % remaining.length;
      activeId.value = remaining[currentIndex].id;
    }, 140);

    timeout = setTimeout(() => {
      clearInterval(interval);
      interval = null;

      isRolling.value = false;
      activeId.value = null;

      if (!isFinalRound) {
        winnerId.value = id;
      }

      timeout = setTimeout(
        () => {
          displayedRemoved.value = new Set([...displayedRemoved.value, id]);

          winnerId.value = null;

          if (isFinalRound && finalAmount) {
            timeout = setTimeout(() => {
              finalId.value = finalAmount.id;
            }, 450);
          }
        },
        isFinalRound ? 0 : 650,
      );
    }, FANFARE_DURATION);
  }

  watch(
    () =>
      props.amounts.map((a) => ({
        id: a.id,
        removed: a.removed,
      })),
    (newAmounts, oldAmounts) => {
      if (!oldAmounts) return;

      const newlyRemoved = newAmounts.find((item) => {
        const previous = oldAmounts.find((old) => old.id === item.id);

        return item.removed && previous && !previous.removed;
      });

      if (newlyRemoved) {
        playRemovalAnimation(newlyRemoved.id);
      }

      const resetOrUndone = newAmounts.every((item) => !item.removed);

      if (resetOrUndone) {
        stopAnimation();

        finalId.value = null;
        winnerId.value = null;
        activeId.value = null;
        isRolling.value = false;
      }

      displayedRemoved.value = new Set(
        [...displayedRemoved.value].filter((id) =>
          newAmounts.some((a) => a.id === id && a.removed),
        ),
      );
    },
  );

  onUnmounted(() => {
    stopAnimation();
    fanfareAudio.pause();
  });
</script>

<template>
  <main class="tv-screen">
    <button class="sound-toggle" @click="enableSound">
      {{ soundEnabled ? "Ljud på ✓" : "Aktivera ljud 🔊" }}
    </button>

    <section class="tv-layout" :class="{ 'final-layout': finalId !== null }">
      <div class="tv-brand" v-if="finalId === null">
        <img :src="logo" alt="BYTT är BYTT" class="tv-logo" />

        <div class="brand-line"></div>
        <p>SPELET PÅGÅR</p>
      </div>

      <div class="tv-board" :class="{ 'final-board': finalId !== null }">
        <div
          v-for="amount in props.amounts"
          :key="amount.id"
          v-show="finalId === null || amount.id === finalId"
          class="tv-amount"
          :class="{
            'rolling-highlight': isRolling && activeId === amount.id,
            'winner-highlight': winnerId === amount.id,
            removed: displayedRemoved.has(amount.id),
            'final-amount': finalId === amount.id,
          }"
        >
          <span class="amount-value">
            {{ formatAmount(amount.value) }}
          </span>
        </div>
      </div>
    </section>
  </main>
</template>
