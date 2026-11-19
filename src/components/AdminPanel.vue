<script setup>
  import { ref } from "vue";

  const tvUrl = `${import.meta.env.BASE_URL}#/tv`;
  const leadershipUrl = `${import.meta.env.BASE_URL}#/ledning`;

  const props = defineProps({
    amounts: {
      type: Array,
      required: true,
    },
  });

  const emit = defineEmits([
    "toggle",
    "reset",
    "add",
    "edit",
    "delete",
    "move",
    "start-leadership",
    "reset-leadership",
  ]);

  const newAmount = ref("");
  const editingId = ref(null);
  const editingValue = ref("");

  function formatAmount(value) {
    return new Intl.NumberFormat("sv-SE").format(value);
  }

  function parseAmount(value) {
    const parsed = Number(String(value).replace(/\s/g, "").replace(",", "."));

    return Number.isInteger(parsed) && parsed > 0 ? parsed : null;
  }

  function addAmount() {
    const value = parseAmount(newAmount.value);

    if (value === null) {
      alert("Ange ett giltigt belopp.");
      return;
    }

    emit("add", value);
    newAmount.value = "";
  }

  function startEditing(amount) {
    editingId.value = amount.id;
    editingValue.value = amount.value;
  }

  function saveEdit(id) {
    const value = parseAmount(editingValue.value);

    if (value === null) {
      alert("Ange ett giltigt belopp.");
      return;
    }

    emit("edit", { id, value });
    cancelEdit();
  }

  function cancelEdit() {
    editingId.value = null;
    editingValue.value = "";
  }

  function deleteAmount(id) {
    if (confirm("Vill du verkligen ta bort den här valören?")) {
      emit("delete", id);
    }
  }

  function moveAmount(id, direction) {
    emit("move", { id, direction });
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

    <!-- LEDNINGSGRUPPENS PRESENTATION -->
    <section class="leadership-admin">
      <div>
        <p class="eyebrow">SEPARAT PRESENTATION</p>
        <h2>Ledningsgruppens gissning</h2>
        <p>Starta avslöjandet av ledningsgruppens val: 500 000 kr.</p>
      </div>

      <div class="leadership-admin-actions">
        <button
          class="leadership-start-button"
          @click="emit('start-leadership')"
        >
          ▶ Starta avslöjandet
        </button>

        <button
          class="leadership-reset-button"
          @click="emit('reset-leadership')"
        >
          Kör igen
        </button>
      </div>
    </section>

    <section class="admin-content">
      <h2>Belopp</h2>

      <p class="description">
        Klicka på ett belopp när motsvarande föremål har plockats bort.
      </p>

      <div class="amount-grid">
        <div
          v-for="(amount, index) in props.amounts"
          :key="amount.id"
          class="amount-card"
        >
          <button
            class="amount-button"
            :class="{ removed: amount.removed }"
            @click="emit('toggle', amount.id)"
          >
            <span>{{ formatAmount(amount.value) }}</span>
            <small>
              {{ amount.removed ? "BORTTAGET" : "Klicka för att ta bort" }}
            </small>
          </button>

          <div class="amount-actions">
            <template v-if="editingId === amount.id">
              <form class="edit-form" @submit.prevent="saveEdit(amount.id)">
                <input
                  v-model="editingValue"
                  type="number"
                  min="1"
                  step="1"
                  aria-label="Nytt belopp"
                />

                <button type="submit" class="small-button">Spara</button>

                <button type="button" class="small-button" @click="cancelEdit">
                  Avbryt
                </button>
              </form>
            </template>

            <template v-else>
              <button class="small-button" @click="startEditing(amount)">
                Ändra
              </button>

              <button
                class="small-button"
                :disabled="index === 0"
                @click="moveAmount(amount.id, -1)"
                aria-label="Flytta upp"
              >
                ↑
              </button>

              <button
                class="small-button"
                :disabled="index === props.amounts.length - 1"
                @click="moveAmount(amount.id, 1)"
                aria-label="Flytta ned"
              >
                ↓
              </button>

              <button
                class="small-button delete-button"
                @click="deleteAmount(amount.id)"
              >
                Ta bort
              </button>
            </template>
          </div>
        </div>
      </div>

      <div class="add-amount-section">
        <h3>Lägg till ny valör</h3>

        <form class="add-amount-form" @submit.prevent="addAmount">
          <input
            v-model="newAmount"
            type="number"
            min="1"
            step="1"
            placeholder="Ange belopp, t.ex. 75000"
            aria-label="Nytt belopp"
          />

          <button type="submit" class="small-button add-button">
            + Lägg till
          </button>
        </form>
      </div>
    </section>

    <footer>
      <span>
        {{ props.amounts.filter((a) => !a.removed).length }}
        av {{ props.amounts.length }} belopp kvar
      </span>

      <div class="view-links">
        <a :href="tvUrl" target="_blank">Öppna TV-vy ↗</a>
        <a :href="leadershipUrl" target="_blank">
          Öppna ledningsgruppens vy ↗
        </a>
      </div>
    </footer>
  </main>
</template>

<style scoped>
  .amount-card {
    display: flex;
    flex-direction: column;
    gap: 8px;
    min-width: 0;
  }

  .amount-actions {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 5px;
  }

  .small-button {
    padding: 6px 10px;
    border: 1px solid #ccc;
    border-radius: 5px;
    background: transparent;
    color: inherit;
    cursor: pointer;
    font-size: 12px;
  }

  .small-button:disabled {
    opacity: 0.3;
    cursor: not-allowed;
  }

  .delete-button {
    color: #a33;
  }

  .edit-form,
  .add-amount-form {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    align-items: center;
  }

  .edit-form input,
  .add-amount-form input {
    min-width: 0;
    width: 130px;
    padding: 8px;
    border: 1px solid #ccc;
    border-radius: 5px;
    font: inherit;
  }

  .add-amount-section {
    margin-top: 32px;
  }

  .add-amount-section h3 {
    margin-bottom: 12px;
  }

  .add-button {
    font-weight: 600;
  }

  /* Ledningsgruppens presentation */
  .leadership-admin {
    margin: 30px 0;
    padding: 24px;
    border: 1px solid #e3b1a7;
    border-radius: 10px;
    background: #fff8f5;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 20px;
    flex-wrap: wrap;
  }

  .leadership-admin h2 {
    margin: 6px 0;
  }

  .leadership-admin p {
    margin-bottom: 0;
  }

  .leadership-admin-actions {
    display: flex;
    gap: 10px;
    flex-wrap: wrap;
  }

  .leadership-start-button,
  .leadership-reset-button {
    padding: 12px 18px;
    border: none;
    border-radius: 6px;
    cursor: pointer;
    font: inherit;
    font-weight: 600;
  }

  .leadership-start-button {
    background: #a64f47;
    color: white;
  }

  .leadership-reset-button {
    background: #ead8d3;
    color: #352321;
  }

  .view-links {
    display: flex;
    flex-wrap: wrap;
    gap: 18px;
  }
</style>
