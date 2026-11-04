<script setup>
  import { ref, onMounted, onUnmounted } from "vue";
  import AdminPanel from "./components/AdminPanel.vue";
  import TvDisplay from "./components/TvDisplay.vue";
  import { supabase } from "./lib/supabase";

  const isTvView = window.location.hash === "#/tv";

  const amounts = ref([]);
  const loading = ref(true);
  const errorMessage = ref("");
  const isUpdating = ref(false);

  let realtimeChannel = null;

  function applyAmounts(newAmounts) {
    if (Array.isArray(newAmounts)) {
      amounts.value = newAmounts;
    }
  }

  async function loadGame() {
    const { data, error } = await supabase
      .from("game_state")
      .select("amounts")
      .eq("id", "main")
      .single();

    if (error) {
      console.error(error);
      errorMessage.value = "Kunde inte hämta spelet.";
      return;
    }

    applyAmounts(data.amounts);
  }

  async function saveGame(newAmounts) {
    if (isUpdating.value) return;

    isUpdating.value = true;
    errorMessage.value = "";

    const { data, error } = await supabase
      .from("game_state")
      .update({
        amounts: newAmounts,
        updated_at: new Date().toISOString(),
      })
      .eq("id", "main")
      .select("amounts")
      .single();

    if (error) {
      console.error(error);
      errorMessage.value = "Kunde inte spara ändringen.";
    } else {
      applyAmounts(data.amounts);
    }

    isUpdating.value = false;
  }

  function updateAmount(id) {
    const updated = amounts.value.map((item) =>
      item.id === id ? { ...item, removed: !item.removed } : item,
    );

    saveGame(updated);
  }

  // Lägg till en ny valör
  function addAmount(value) {
    const newValue = Number(value);

    if (!Number.isFinite(newValue) || newValue <= 0) return;

    const nextId = Math.max(0, ...amounts.value.map((a) => a.id)) + 1;

    saveGame([
      ...amounts.value,
      {
        id: nextId,
        value: newValue,
        removed: false,
      },
    ]);
  }

  // Redigera en befintlig valör
  function editAmount({ id, value }) {
    const newValue = Number(value);

    if (!Number.isFinite(newValue) || newValue <= 0) return;

    const updated = amounts.value.map((item) =>
      item.id === id ? { ...item, value: newValue } : item,
    );

    saveGame(updated);
  }

  // Ta bort en valör
  function deleteAmount(id) {
    const updated = amounts.value.filter((item) => item.id !== id);
    saveGame(updated);
  }

  // Flytta en valör upp eller ned
  function moveAmount({ id, direction }) {
    const updated = [...amounts.value];
    const index = updated.findIndex((item) => item.id === id);
    const newIndex = index + direction;

    if (newIndex < 0 || newIndex >= updated.length) return;

    [updated[index], updated[newIndex]] = [updated[newIndex], updated[index]];

    saveGame(updated);
  }

  // Återställ borttagna belopp, men behåll valörerna
  function resetGame() {
    const updated = amounts.value.map((item) => ({
      ...item,
      removed: false,
    }));

    saveGame(updated);
  }

  // Ny spelomgång med samma valörer
  function newGame() {
    resetGame();
  }

  function subscribeToGame() {
    realtimeChannel = supabase
      .channel("bytt-ar-bytt-game")
      .on(
        "postgres_changes",
        {
          event: "UPDATE",
          schema: "public",
          table: "game_state",
          filter: "id=eq.main",
        },
        (payload) => {
          applyAmounts(payload.new.amounts);
        },
      )
      .subscribe((status) => {
        if (status === "CHANNEL_ERROR") {
          errorMessage.value = "Kunde inte ansluta till realtidssynkningen.";
        }
      });
  }

  onMounted(async () => {
    await loadGame();
    loading.value = false;
    subscribeToGame();
  });

  onUnmounted(() => {
    if (realtimeChannel) {
      supabase.removeChannel(realtimeChannel);
    }
  });
</script>

<template>
  <div>
    <div v-if="loading">Ansluter till spelet...</div>

    <div v-else-if="errorMessage">
      {{ errorMessage }}
    </div>

    <template v-else>
      <AdminPanel
        v-if="!isTvView"
        :amounts="amounts"
        @toggle="updateAmount"
        @reset="resetGame"
        @new-game="newGame"
        @add="addAmount"
        @edit="editAmount"
        @delete="deleteAmount"
        @move="moveAmount"
      />

      <TvDisplay v-else :amounts="amounts" />
    </template>
  </div>
</template>
