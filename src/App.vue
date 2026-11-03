<script setup>
  import { ref, onMounted, onUnmounted } from "vue";
  import AdminPanel from "./components/AdminPanel.vue";
  import TvDisplay from "./components/TvDisplay.vue";

  const isTvView = window.location.pathname.includes("/tv");

  const STORAGE_KEY = "bytt-ar-bytt-game";

  const initialAmounts = [
    1000000, 500000, 175000, 100000, 50000, 25000, 10000, 500,
  ];

  function createInitialAmounts() {
    return initialAmounts.map((value, index) => ({
      id: index + 1,
      value,
      removed: false,
    }));
  }

  function getSavedAmounts() {
    try {
      const saved = localStorage.getItem(STORAGE_KEY);
      return saved ? JSON.parse(saved) : createInitialAmounts();
    } catch {
      return createInitialAmounts();
    }
  }

  const amounts = ref(getSavedAmounts());

  const channel = new BroadcastChannel("bytt-ar-bytt");

  function sync() {
    const state = JSON.parse(JSON.stringify(amounts.value));

    // Spara gemensamt spelläge
    localStorage.setItem(STORAGE_KEY, JSON.stringify(state));

    // Skicka uppdatering till andra fönstret
    channel.postMessage(state);
  }

  function updateAmount(id) {
    amounts.value = amounts.value.map((item) =>
      item.id === id ? { ...item, removed: !item.removed } : item,
    );

    sync();
  }

  function resetGame() {
    amounts.value = createInitialAmounts();
    sync();
  }

  function handleStorage(event) {
    if (event.key === STORAGE_KEY && event.newValue) {
      amounts.value = JSON.parse(event.newValue);
    }
  }

  onMounted(() => {
    // Ta emot uppdateringar från andra fönster
    channel.onmessage = (event) => {
      amounts.value = event.data;
    };

    // Lyssna även på localStorage-ändringar
    window.addEventListener("storage", handleStorage);

    // TV-fönstret hämtar det senaste spelläget
    if (isTvView) {
      amounts.value = getSavedAmounts();
    }
  });

  onUnmounted(() => {
    window.removeEventListener("storage", handleStorage);
    channel.close();
  });
</script>

<template>
  <div>
    <AdminPanel
      v-if="!isTvView"
      :amounts="amounts"
      @toggle="updateAmount"
      @reset="resetGame"
    />

    <TvDisplay v-else :amounts="amounts" />
  </div>
</template>
