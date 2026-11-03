<script setup>
  import { ref, onMounted, onUnmounted } from "vue";
  import AdminPanel from "./components/AdminPanel.vue";
  import TvDisplay from "./components/TvDisplay.vue";
  import { supabase } from "./lib/supabase";

  const isTvView = window.location.hash === "#/tv";

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

  const amounts = ref(createInitialAmounts());
  const loading = ref(true);
  const errorMessage = ref("");
  const isUpdating = ref(false);

  let realtimeChannel = null;

  function applyAmounts(newAmounts) {
    if (Array.isArray(newAmounts)) {
      amounts.value = newAmounts;
    }
  }

  // Hämta aktuellt spelläge från Supabase
  async function loadGame() {
    const { data, error } = await supabase
      .from("game_state")
      .select("amounts")
      .eq("id", "main")
      .single();

    if (error) {
      console.error("Kunde inte hämta spelet:", error);
      errorMessage.value = "Kunde inte hämta spelet från databasen.";
      return;
    }

    applyAmounts(data.amounts);
  }

  // Spara nytt spelläge i Supabase
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
      console.error("Kunde inte spara spelet:", error);
      errorMessage.value =
        "Kunde inte spara ändringen. Kontrollera anslutningen.";
    } else {
      applyAmounts(data.amounts);
    }

    isUpdating.value = false;
  }

  // Ta bort eller återställa ett belopp
  async function updateAmount(id) {
    if (isUpdating.value) return;

    const newAmounts = amounts.value.map((item) =>
      item.id === id ? { ...item, removed: !item.removed } : item,
    );

    await saveGame(newAmounts);
  }

  // Återställ hela spelet
  async function resetGame() {
    if (isUpdating.value) return;

    await saveGame(createInitialAmounts());
  }

  // Lyssna på ändringar från andra enheter
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
          errorMessage.value =
            "Kunde inte ansluta till spelets realtidssynkning.";
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
    <div v-if="loading" class="connection-message">Ansluter till spelet...</div>

    <div v-else-if="errorMessage" class="connection-error">
      {{ errorMessage }}
    </div>

    <template v-else>
      <AdminPanel
        v-if="!isTvView"
        :amounts="amounts"
        @toggle="updateAmount"
        @reset="resetGame"
      />

      <TvDisplay v-else :amounts="amounts" />
    </template>
  </div>
</template>
