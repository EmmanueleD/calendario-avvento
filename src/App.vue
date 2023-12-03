<script setup>
import { ref, onMounted, computed } from "vue";
import { supabase } from "@/composables/useSupabase";

const loadingLeads = ref(false);

const secretLeads = ref([]);
const notSecretLeads = ref([]);

const error = ref(null);

const numberOfNoSecrets = computed(() => {
  return notSecretLeads.value.length;
});

const numberOfDaysFromFirstOfDecember = computed(() => {
  return (
    Math.floor((new Date() - new Date(2023, 11, 1)) / (1000 * 60 * 60 * 24)) + 1
  );
});

const numberOfSecretsThatCanBeRevealed = computed(() => {
  return numberOfDaysFromFirstOfDecember.value - numberOfNoSecrets.value;
});

onMounted(async () => {
  await getSecretLeads();
  await getNotSecretLeads();
});

async function getSecretLeads() {
  loadingLeads.value = true;
  secretLeads.value.splice(0);
  try {
    let { data, error } = await supabase
      .from("leads")
      .select("id, is_secret, encrypted_text")
      .eq("is_secret", true);

    data.map((lead) => {
      lead = { ...lead, loading: false };
    });

    console.log("getSecretLeads", data);
    if (error) throw error;
    secretLeads.value = data;
  } catch (error) {
    console.log(error);
  } finally {
    orederById(secretLeads.value);
    loadingLeads.value = false;
  }
}

async function getNotSecretLeads() {
  loadingLeads.value = true;
  notSecretLeads.value.splice(0);
  try {
    let { data, error } = await supabase
      .from("leads")
      .select("id, is_secret, full_text")
      .eq("is_secret", false);

    data.map((lead) => {
      lead = { ...lead, loading: false };
    });

    console.log("getNotSecretLeads", data);
    if (error) throw error;
    notSecretLeads.value = data;
  } catch (error) {
    console.log(error);
  } finally {
    orederById(notSecretLeads.value);
    loadingLeads.value = false;
  }
}

function orederById(array) {
  array = array.sort((a, b) => a.id - b.id);
}

async function revealSecret(lead) {
  console.log(lead);
  lead.laoding = true;

  try {
    const { data, error } = await supabase
      .from("leads")
      .update({ is_secret: false })
      .eq("id", lead.id)
      .select();
  } catch (error) {
    console.log(error);
  } finally {
    lead.laoding = false;
    getNotSecretLeads();
    getSecretLeads();
  }
}
</script>

<template>
  <div
    class="w-full h-screen overflow-y-scroll flex flex-col items-center px-6 py-4 bg-emerald-800 text-white"
  >
    <h1 class="font-semibold text-4xl mt-8 mb-4">Secretos revelados</h1>
    <div
      v-if="loadingLeads"
      class="w-full flex flex-col items-start mb-2 bg-zinc-100 p-4 rounded-3xl border-red-950 border-4 text-emerald-900"
    >
      Cargando...
    </div>
    <div
      v-else
      v-for="lead in notSecretLeads"
      :key="lead.id"
      class="w-full flex flex-col items-start mb-2 bg-zinc-100 p-4 rounded-3xl border-red-950 border-4 text-emerald-900 max-w-lg"
    >
      <span class="w-full">{{ lead.full_text }}</span>
    </div>

    <h1 class="font-semibold text-4xl my-4">Secretos</h1>

    <span v-if="loadingLeads"></span>
    <span
      v-else-if="numberOfSecretsThatCanBeRevealed > 0"
      class="text-sm my-4 text-zinc-200"
    >
      Numero de secretos que puedes revelar:
      <span class="text-zinc-50 font-bold text-lg">
        {{ numberOfSecretsThatCanBeRevealed }}</span
      >
    </span>

    <span v-else class="text-sm my-4 text-zinc-200"
      >No puedes revelar secretos, vuelve mañana
    </span>

    <div
      v-if="loadingLeads"
      class="w-full flex flex-col items-start mb-2 bg-zinc-100 p-4 rounded-3xl border-red-950 border-4 text-emerald-900"
    >
      Cargando...
    </div>

    <div
      v-else
      v-for="lead in secretLeads"
      :key="lead.id"
      class="w-full flex flex-col items-start mb-2 bg-zinc-100 p-4 rounded-3xl border-red-950 border-4 text-emerald-900 max-w-lg"
    >
      <span v-if="lead.loading" class="w-full">Cargando...</span>
      <span v-else class="w-full">{{ lead.encrypted_text }}</span>

      <button
        class="mt-4 text-sm text-red-950 font-semibold border border-red-950 rounded-3xl px-4 py-2 background-transparent hover:bg-red-900 hover:border-red-900 hover:text-red-900 hover:cursor-pointer hover:text-white disabled:bg-zinc-100 disabled:border-zinc-400 disabled:text-zinc-500 disabled:cursor-not-allowed disabled:font-normal disabled:line-through"
        :disabled="!numberOfSecretsThatCanBeRevealed > 0"
        @click="revealSecret(lead)"
      >
        {{ lead.loading ? "Cargando..." : "Revela el secreto" }}
      </button>
    </div>
  </div>
</template>
