<script setup lang="ts">
import { ref, computed, onMounted, onBeforeUnmount } from "vue";
import { createClient } from "@supabase/supabase-js";

interface Item {
  id: string;
  name: string;
  quantity: number | null;
  price: number | string | null;
  added_by: string | null;
  purchased: boolean | null;
  created_at: string | null;
}

const SUPABASE_URL = import.meta.env.VITE_SUPABASE_URL as string;
const SUPABASE_ANON_KEY = import.meta.env.VITE_SUPABASE_ANON_KEY as string;

if (!SUPABASE_URL || !SUPABASE_ANON_KEY) {
  console.error(
    "Variáveis de ambiente do Supabase não configuradas (VITE_SUPABASE_URL / VITE_SUPABASE_ANON_KEY)."
  );
}

const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY);

let channel: any | null = null;

const items = ref<Item[]>([]);
const name = ref("");
const qty = ref<number | string>(1);
const price = ref("");
const userName = ref("");
const loading = ref(false);
const filter = ref<"all" | "pending" | "purchased">("all");

// Edição inline (modal simples)
const editingItem = ref<Item | null>(null);
const editName = ref("");
const editPrice = ref("");
const savingEdit = ref(false);

if (typeof window !== "undefined") {
  userName.value = window.localStorage.getItem("buyer_name") || "";
}

function handleRealtime(payload: any) {
  const type = payload.eventType;
  const newRow = payload.new as Item;
  const oldRow = payload.old as Item;

  items.value = (items.value || []).slice();

  if (type === "INSERT") {
    items.value = [newRow, ...items.value];
  } else if (type === "UPDATE") {
    items.value = items.value.map((i) => (i.id === newRow.id ? newRow : i));
  } else if (type === "DELETE") {
    items.value = items.value.filter((i) => i.id !== oldRow.id);
  }
}

async function fetchItems() {
  loading.value = true;
  const { data, error } = await supabase
    .from("items")
    .select("*")
    .order("created_at", { ascending: false });

  if (error) {
    console.error("Erro ao buscar itens:", error);
    loading.value = false;
    return;
  }

  items.value = (data || []) as Item[];
  loading.value = false;
}

async function addItem() {
  if (!name.value.trim()) return;

  const item = {
    name: name.value.trim(),
    quantity: parseInt(String(qty.value), 10) || 1,
    price: price.value ? parseFloat(price.value) : 0,
    added_by: userName.value || "Anônimo",
  };

  const { data, error } = await supabase
    .from("items")
    .insert(item)
    .select()
    .single();

  if (error) {
    console.error("Erro ao inserir:", error);
    return;
  }

  // Atualiza a lista local imediatamente
  if (data) {
    items.value = [data as Item, ...(items.value || [])];
  }

  name.value = "";
  qty.value = 1;
  price.value = "";
}

async function togglePurchased(item: Item) {
  const newValue = !item.purchased;

  const { error } = await supabase
    .from("items")
    .update({ purchased: newValue })
    .eq("id", item.id);

  if (error) {
    console.error("Erro ao atualizar:", error);
    return;
  }

  // Atualiza a lista local imediatamente
  items.value = (items.value || []).map((i) =>
    i.id === item.id ? { ...i, purchased: newValue } : i
  );
}

async function removeItem(item: Item) {
  if (!window.confirm("Remover item?")) return;

  const { error } = await supabase.from("items").delete().eq("id", item.id);

  if (error) {
    console.error("Erro ao excluir:", error);
    return;
  }

  // Atualiza a lista local imediatamente, mesmo sem depender do Realtime
  items.value = (items.value || []).filter((i) => i.id !== item.id);
}

async function editItem(item: Item, updates: Partial<Item>) {
  const { error } = await supabase
    .from("items")
    .update(updates)
    .eq("id", item.id);

  if (error) {
    console.error("Erro ao editar:", error);
    return;
  }

  // Atualiza a lista local imediatamente
  items.value = (items.value || []).map((i) =>
    i.id === item.id ? { ...i, ...updates } : i
  );
}

function openEditItem(item: Item) {
  editingItem.value = item;
  editName.value = item.name || "";
  editPrice.value = item.price ? String(item.price) : "";
}

function closeEditItem() {
  if (savingEdit.value) return;
  editingItem.value = null;
  editName.value = "";
  editPrice.value = "";
}

async function saveEditItem() {
  if (!editingItem.value) return;

  const trimmedName = editName.value.trim();
  if (!trimmedName) {
    window.alert("Informe um nome para o item.");
    return;
  }

  const normalizado = editPrice.value.replace(",", ".");
  const novoPreco = normalizado ? parseFloat(normalizado) : 0;
  if (Number.isNaN(novoPreco) || novoPreco < 0) {
    window.alert("Preço inválido");
    return;
  }

  savingEdit.value = true;
  await editItem(editingItem.value, { name: trimmedName, price: novoPreco });
  savingEdit.value = false;
  closeEditItem();
}

const totalSum = computed(() => {
  return (items.value || []).reduce((acc, it) => {
    const priceNumber = Number(it.price || 0);
    const quantityNumber = it.quantity || 1;
    return acc + priceNumber * quantityNumber;
  }, 0);
});

const visibleItems = computed(() => {
  if (filter.value === "all") return items.value;
  if (filter.value === "purchased")
    return items.value.filter((i) => i.purchased);
  return items.value.filter((i) => !i.purchased);
});

function saveUserName() {
  if (typeof window !== "undefined") {
    window.localStorage.setItem("buyer_name", userName.value);
  }
  window.alert("Nome salvo: " + userName.value);
}

onMounted(() => {
  fetchItems();

  channel = supabase
    .channel("public:items")
    .on(
      "postgres_changes",
      { event: "*", schema: "public", table: "items" },
      (payload) => {
        handleRealtime(payload);
      }
    )
    .subscribe();
});

onBeforeUnmount(() => {
  if (channel) {
    supabase.removeChannel(channel);
  }
});
</script>

<template>
  <div class="min-h-screen bg-gray-50 p-4 sm:p-6">
    <div class="max-w-5xl mx-auto bg-white shadow rounded-lg p-4 sm:p-6">
      <div class="mb-4 overflow-hidden rounded-lg">
        <img
          src="/banner-ets.svg"
          alt="Grupo de extraterrestres viajando juntos"
          class="h-32 w-full object-cover sm:h-40"
        />
      </div>
      <div class="mb-4 rounded-lg bg-gradient-to-r from-blue-500 to-indigo-500 px-4 py-3 text-white flex flex-col gap-1 sm:flex-row sm:items-center sm:justify-between">
        <p class="text-sm sm:text-base font-medium">
          Lista de compras coletiva da viagem para Pirenópolis
        </p>
        <p class="text-xs sm:text-sm text-blue-100">
          Adicione seus itens e veja tudo atualizar em tempo real
        </p>
      </div>
      <header class="mb-4"></header>

      <!-- Card do usuário / total / filtro -->
      <section class="mb-6">
        <div class="p-4 border rounded">
          <label class="text-sm"
            >Seu nome (para identificar quem adicionou)</label
          >
          <div class="flex flex-col gap-2 mt-2 sm:flex-row">
            <input v-model="userName" class="flex-1 border rounded px-2 py-2" />
            <button
              type="button"
              class="w-full sm:w-auto px-3 py-2 rounded bg-green-600 text-white"
              @click="saveUserName"
            >
              Salvar
            </button>
          </div>

          <div class="mt-4 text-sm">
            <div>Total estimado: R$ {{ totalSum.toFixed(2) }}</div>
            <div class="text-gray-500 text-xs mt-2">
              Divisão por 20 pessoas: R$ {{ (totalSum / 20).toFixed(2) }}
            </div>
          </div>

          <div class="mt-4">
            <label class="text-sm">Filtro</label>
            <select
              v-model="filter"
              class="w-full mt-2 border rounded px-2 py-2"
            >
              <option value="all">Todos</option>
              <option value="pending">Pendentes</option>
              <option value="purchased">Comprados</option>
            </select>
          </div>
        </div>
      </section>

      <!-- Formulário de adicionar item -->
      <section class="mb-6">
        <form
          class="p-4 border rounded"
          @submit.prevent="addItem"
        >
          <div class="flex flex-col gap-2 mb-2 sm:flex-row">
            <input
              v-model="name"
              placeholder="Nome do item"
              class="w-full flex-1 border rounded px-3 py-2"
            />
            <input
              v-model="qty"
              type="number"
              min="1"
              class="w-full sm:w-20 border rounded px-2 py-2"
            />
            <input
              v-model="price"
              placeholder="Preço (R$)"
              class="w-full sm:w-28 border rounded px-2 py-2"
            />
            <button class="w-full sm:w-auto bg-blue-600 text-white px-4 py-2 rounded">
              Adicionar
            </button>
          </div>
          <div class="text-xs text-gray-500">
            Dica: deixe o preço em 0 se não souber. Todos os updates aparecem em
            tempo real.
          </div>
        </form>
      </section>

      <section>
        <div v-if="loading">Carregando...</div>
        <ul v-else class="space-y-2">
          <li
            v-for="item in visibleItems"
            :key="item.id"
            :class="[
              'flex flex-col gap-2 rounded-lg border border-gray-200 px-3 py-2 shadow-sm transition-all',
              item.purchased
                ? 'bg-slate-50 opacity-90'
                : 'bg-blue-50 hover:bg-blue-100 hover:border-blue-200'
            ]"
          >
            <!-- Linha única: info à esquerda, preço + quantidade + lixeira à direita -->
            <div class="flex items-start justify-between gap-2">
              <div class="flex items-start gap-2">
                <input
                  type="checkbox"
                  :checked="!!item.purchased"
                  @change="() => togglePurchased(item)"
                  class="mt-1"
                />
                <div>
                  <div class="flex items-center gap-1 text-sm font-medium">
                    <span
                      :class="item.purchased ? 'line-through text-gray-400' : 'text-gray-900'"
                    >
                      {{ item.name }}
                    </span>
                    <span class="text-xs text-gray-500">
                      x{{ item.quantity || 1 }}
                    </span>
                  </div>
                  <div class="mt-0.5 flex items-center gap-1 text-[11px] text-gray-500">
                    <button
                      type="button"
                      class="inline-flex h-7 w-7 items-center justify-center rounded-full text-[14px] text-green-600 hover:text-green-700 hover:bg-green-50 -ml-6"
                      @click="openEditItem(item)"
                      aria-label="Editar item"
                    >
                      ✎
                    </button>
                    <span>Adicionado por: {{ item.added_by || 'Anônimo' }}</span>
                  </div>
                </div>
              </div>

              <div class="flex items-center gap-1 sm:gap-1.5">
                <span
                  class="text-sm font-medium"
                  :class="item.purchased ? 'text-gray-400' : 'text-gray-800'"
                >
                  R$ {{ Number(item.price || 0).toFixed(2) }}
                </span>
                <button
                  class="inline-flex h-6 w-6 items-center justify-center border rounded-full text-[10px]"
                  type="button"
                  @click="editItem(item, { quantity: (item.quantity || 1) + 1 })"
                >
                  +
                </button>
                <button
                  class="inline-flex h-6 w-6 items-center justify-center border rounded-full text-[10px]"
                  type="button"
                  @click="
                    editItem(item, {
                      quantity: Math.max(1, (item.quantity || 1) - 1),
                    })
                  "
                >
                  -
                </button>
                <button
                  class="ml-1 inline-flex h-7 w-7 items-center justify-center rounded-full text-red-500 hover:text-red-600 hover:bg-red-50 transition"
                  type="button"
                  @click="removeItem(item)"
                  aria-label="Remover item"
                >
                  <svg
                    xmlns="http://www.w3.org/2000/svg"
                    viewBox="0 0 24 24"
                    fill="currentColor"
                    class="h-4 w-4"
                  >
                    <path
                      d="M9.5 4.75A.75.75 0 0 1 10.25 4h3.5a.75.75 0 0 1 .75.75V5.5h4a.75.75 0 0 1 0 1.5h-.442l-.73 10.077A2.25 2.25 0 0 1 15.087 19H8.913a2.25 2.25 0 0 1-2.241-1.923L5.94 7H5.5a.75.75 0 0 1 0-1.5h4V4.75Z"
                    />
                  </svg>
                </button>
              </div>
            </div>
          </li>
        </ul>
      </section>

      <!-- Modal de edição de item -->
      <div
        v-if="editingItem"
        class="fixed inset-0 z-20 flex items-center justify-center bg-black/40 px-4"
      >
        <div class="w-full max-w-md rounded-lg bg-white p-4 shadow-lg">
          <h2 class="mb-3 text-base font-semibold">Editar item</h2>

          <div class="mb-3 space-y-2 text-sm">
            <label class="block text-xs font-medium text-gray-600">Nome</label>
            <input
              v-model="editName"
              class="w-full rounded border px-3 py-2 text-sm"
              placeholder="Nome do item"
            />
          </div>

          <div class="mb-4 space-y-2 text-sm">
            <label class="block text-xs font-medium text-gray-600">Preço (R$)</label>
            <input
              v-model="editPrice"
              class="w-full rounded border px-3 py-2 text-sm"
              placeholder="Ex: 25,90"
            />
          </div>

          <div class="flex justify-end gap-2 text-sm">
            <button
              type="button"
              class="rounded border px-3 py-1.5 text-gray-600 hover:bg-gray-50"
              @click="closeEditItem"
              :disabled="savingEdit"
            >
              Cancelar
            </button>
            <button
              type="button"
              class="rounded bg-blue-600 px-3 py-1.5 font-medium text-white hover:bg-blue-700 disabled:opacity-60"
              @click="saveEditItem"
              :disabled="savingEdit"
            >
              Salvar
            </button>
          </div>
        </div>
      </div>

      <footer class="mt-6 text-center sm:text-right text-xs text-gray-500">
        Dica: compartilhe a URL desta página com o grupo — todos podem adicionar
        itens.
      </footer>
    </div>
  </div>
</template>

<style scoped></style>
