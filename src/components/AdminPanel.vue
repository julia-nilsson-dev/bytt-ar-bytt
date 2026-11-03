<script setup>
  const props = defineProps({
    amounts: {
      type: Array,
      required: true,
    },
  });

  const emit = defineEmits(["toggle", "reset"]);

  function formatAmount(value) {
    return new Intl.NumberFormat("sv-SE").format(value);
  }
</script>

<template>
  <main class="admin">
    <header class="admin-header">
      <div>
        <p class="eyebrow">SPELEDARVY</p>
        <h1>Bytt är Bytt</h1>
        <p>Kontrollpanel</p>
      </div>

      <button class="reset-button" @click="emit('reset')">
        Återställ spel
      </button>
    </header>

    <section class="admin-content">
      <h2>Belopp</h2>
      <p class="description">
        Klicka på ett belopp när motsvarande föremål har plockats bort.
      </p>

      <div class="amount-grid">
        <button
          v-for="amount in props.amounts"
          :key="amount.id"
          class="amount-button"
          :class="{ removed: amount.removed }"
          @click="emit('toggle', amount.id)"
        >
          <span>{{ formatAmount(amount.value) }} kr</span>
          <small>
            {{ amount.removed ? "BORTTAGET" : "Klicka för att ta bort" }}
          </small>
        </button>
      </div>
    </section>

    <footer>
      <span>
        {{ props.amounts.filter((a) => !a.removed).length }}
        av {{ props.amounts.length }} belopp kvar
      </span>
      <a href="/tv" target="_blank">Öppna TV-vy ↗</a>
    </footer>
  </main>
</template>
