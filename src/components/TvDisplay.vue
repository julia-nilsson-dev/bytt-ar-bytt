<script setup>
  import { ref, watch, onUnmounted } from "vue";

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

  function formatAmount(value) {
    return new Intl.NumberFormat("sv-SE").format(value);
  }

  function stopAnimation() {
    clearInterval(interval);
    clearTimeout(timeout);
    interval = null;
    timeout = null;
  }

  function playRemovalAnimation(id) {
    stopAnimation();

    isRolling.value = true;
    winnerId.value = null;

    const remaining = props.amounts.filter((a) => !a.removed);

    // Beloppen som ska vara med i rullningen
    const items = remaining;

    if (!items.length) return;

    currentIndex = 0;
    activeId.value = items[0].id;

    interval = setInterval(() => {
      currentIndex = (currentIndex + 1) % items.length;
      activeId.value = items[currentIndex].id;
    }, 100);

    timeout = setTimeout(() => {
      clearInterval(interval);
      interval = null;

      isRolling.value = false;
      activeId.value = id;
      winnerId.value = id;

      timeout = setTimeout(() => {
        displayedRemoved.value = new Set([...displayedRemoved.value, id]);

        winnerId.value = null;
        activeId.value = null;

        // Kontrollera om endast ett belopp återstår
        const remainingAfterRemoval = props.amounts.filter((a) => !a.removed);

        if (remainingAfterRemoval.length === 1) {
          finalId.value = remainingAfterRemoval[0].id;
        }
      }, 650);
    }, 1600);
  }

  watch(
    () => props.amounts.map((a) => ({ id: a.id, removed: a.removed })),
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
        finalId.value = null;
        winnerId.value = null;
        activeId.value = null;
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
  });
</script>

<template>
  <main class="tv-screen">
    <section class="tv-layout" :class="{ 'final-layout': finalId !== null }">
      <div class="tv-brand" v-if="finalId === null">
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

      <div class="tv-board" :class="{ 'final-board': finalId !== null }">
        <div
          v-for="amount in props.amounts"
          :key="amount.id"
          v-show="!finalId || amount.id === finalId"
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

          <span class="amount-currency">kr</span>
        </div>
      </div>
    </section>
  </main>
</template>
