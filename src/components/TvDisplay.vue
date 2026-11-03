<script setup>
  const props = defineProps({
    amounts: {
      type: Array,
      required: true,
    },
  });

  function formatAmount(value) {
    return new Intl.NumberFormat("sv-SE").format(value);
  }
</script>

<template>
  <main class="tv-screen">
    <section class="tv-layout">
      <div class="tv-brand">
        <div class="brand-circle">
          <span>B&B</span>
        </div>

        <h1>
          BYTT
          <span>är</span>
          BYTT
        </h1>

        <div class="brand-line"></div>
        <p>SPELET PÅGÅR</p>
      </div>

      <div class="tv-board">
        <div
          v-for="amount in props.amounts"
          :key="amount.id"
          class="tv-amount"
          :class="{ removed: amount.removed }"
        >
          <span class="amount-value">
            {{ formatAmount(amount.value) }}
          </span>
          <span class="amount-currency">kr</span>
        </div>
      </div>
    </section>
  </main>
</template>
