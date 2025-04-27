<template>
  <div>
    <!-- Begrüßungs-Card -->
    <v-row>
      <v-col cols="12">
        <v-card elevation="1" color="primary" class="text-white pa-4 rounded-lg">
          <v-card-title class="text-h5 font-weight-medium">
            Willkommen zurück! 
          </v-card-title>
          <v-card-text class="pb-2">
            <p class="text-body-1">Erstellen Sie CLIL-Unterrichtsmaterialien mit KI-Unterstützung für Ihre Technik-Klassen.</p>
          </v-card-text>
          <v-card-actions>
            <v-btn
              variant="elevated"
              color="white"
              class="text-primary font-weight-bold"
              to="/create"
              size="large"
            >
              <v-icon start>mdi-plus-box</v-icon>
              Neues Material erstellen
            </v-btn>
          </v-card-actions>
        </v-card>
      </v-col>
    </v-row>

    <!-- Statistik-Übersicht -->
    <v-row class="mt-4">
      <v-col
        v-for="(stat, index) in statsData"
        :key="`stat-${index}`"
        cols="12"
        sm="6"
        md="3"
      >
        <v-card elevation="1" class="rounded-lg fill-height d-flex flex-column">
          <v-card-text class="pa-4 flex-grow-1">
            <div class="d-flex justify-space-between align-start">
              <div>
                <div class="text-h4 font-weight-bold">{{ stat.value }}</div>
                <div class="text-subtitle-1 text-medium-emphasis">{{ stat.label }}</div>
              </div>
              <v-avatar
                :color="stat.color + '-lighten-4'" 
                size="48"
                class="elevation-0 rounded-circle"
              >
                <v-icon size="24" :color="stat.color">{{ stat.icon }}</v-icon>
              </v-avatar>
            </div>
            <div
              v-if="stat.trend !== undefined" 
              class="mt-2 text-caption d-flex align-center"
              :class="stat.trend >= 0 ? 'text-success' : 'text-error'"
            >
              <v-icon size="small" :color="stat.trend >= 0 ? 'success' : 'error'">
                {{ stat.trend >= 0 ? 'mdi-arrow-up' : 'mdi-arrow-down' }}
              </v-icon>
              <span class="ml-1">{{ Math.abs(stat.trend) }}% vs. Vormonat</span>
            </div>
          </v-card-text>
        </v-card>
      </v-col>
    </v-row>

    <v-row class="mt-4">
      <!-- Schnellzugriff -->
      <v-col cols="12" md="6">
        <v-card elevation="1" class="rounded-lg fill-height">
          <v-card-title class="d-flex align-center text-h6 font-weight-medium">
            <v-icon color="primary" class="mr-2">mdi-flash</v-icon>
            Schnellzugriff
          </v-card-title>
          <v-card-text>
            <v-row dense>
              <v-col
                v-for="(action, index) in quickActions"
                :key="`quick-${index}`"
                cols="6" 
                sm="4"
              >
                <v-card
                  elevation="0"
                  class="pa-3 text-center rounded-lg"
                  :color="action.color + '-lighten-5'"
                  @click="navigateToCreate(action.type, action.templateId)"
                  hover
                >
                  <v-avatar :color="action.color" size="48" class="mb-2 elevation-1">
                    <v-icon color="white">{{ action.icon }}</v-icon>
                  </v-avatar>
                  <div class="text-subtitle-2 font-weight-medium text-high-emphasis">{{ action.title }}</div>
                </v-card>
              </v-col>
            </v-row>
          </v-card-text>
        </v-card>
      </v-col>

      <!-- Letzte Materialien -->
      <v-col cols="12" md="6">
        <v-card elevation="1" class="rounded-lg fill-height">
          <v-card-title class="d-flex align-center text-h6 font-weight-medium">
            <v-icon color="primary" class="mr-2">mdi-history</v-icon>
            Zuletzt bearbeitet
          </v-card-title>
          <v-list lines="two" v-if="!loading && recentMaterials.length > 0">
            <v-list-item
              v-for="(material, index) in recentMaterials"
              :key="`recent-${material.id}`"
              :to="`/edit/${material.id}`"
              :title="material.title"
              :subtitle="`Typ: ${material.type} | Geändert: ${material.modified}`"
            >
              <template v-slot:prepend>
                 <v-icon :color="getIconColor(material.type)">{{ getIconForType(material.type) }}</v-icon>
              </template>
              <template v-slot:append>
                <v-btn
                  icon
                  variant="text"
                  color="grey"
                  :to="`/edit/${material.id}`"
                  density="comfortable"
                >
                  <v-icon>mdi-pencil-outline</v-icon>
                </v-btn>
              </template>
            </v-list-item>
          </v-list>
           <v-card-text v-else-if="loading" class="text-center">
             <v-progress-circular indeterminate color="primary"></v-progress-circular>
             <p class="mt-2">Lade Materialien...</p>
           </v-card-text>
           <v-card-text v-else class="text-center text-medium-emphasis">
             Keine kürzlich bearbeiteten Materialien gefunden.
           </v-card-text>

          <v-card-actions v-if="!loading && recentMaterials.length > 0">
            <v-spacer></v-spacer>
            <v-btn
              color="primary"
              variant="text"
              to="/materials"
              class="font-weight-bold"
            >
              Alle Materialien
              <v-icon end>mdi-arrow-right</v-icon>
            </v-btn>
          </v-card-actions>
        </v-card>
      </v-col>
    </v-row>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue';
import { useRouter } from 'vue-router';
import { useMaterialsStore } from '@/stores/materials';

const router = useRouter();
const materialsStore = useMaterialsStore();

// Ladezustand aus dem Store holen
const loading = computed(() => materialsStore.loading);

// Daten für Statistiken (Beispiel)
const statsData = computed(() => [
  { label: 'Materialien Gesamt', value: materialsStore.materials.length, trend: 5, color: 'primary', icon: 'mdi-folder-text-outline' },
  { label: 'Worksheets', value: materialsStore.materialsByType['worksheet']?.length || 0, color: 'success', icon: 'mdi-file-document-outline' },
  { label: 'Quizzes', value: materialsStore.materialsByType['quiz']?.length || 0, color: 'info', icon: 'mdi-help-circle-outline' },
  { label: 'Favoriten', value: materialsStore.materials.filter(m => m.favorite).length, trend: -2, color: 'warning', icon: 'mdi-star-outline' },
]);

// Daten für Schnellzugriff (Beispiel)
const quickActions = ref([
  { title: 'Leeres Worksheet', icon: 'mdi-file-document-plus-outline', type: 'worksheet', color: 'green' },
  { title: 'Leeres Quiz', icon: 'mdi-comment-question-outline', type: 'quiz', color: 'blue' },
  { title: 'Glossar erstellen', icon: 'mdi-book-open-page-variant-outline', type: 'glossary', color: 'orange' },
  { title: 'Vorlage: Technik B1', icon: 'mdi-file-star-outline', type: 'worksheet', templateId: 'tpl-tech-b1', color: 'purple' },
]);

// Kürzlich bearbeitete Materialien aus dem Store
const recentMaterials = computed(() => materialsStore.recentMaterials);

// Funktion zum Navigieren zur Erstellungsseite
function navigateToCreate(type, templateId = null) {
  const query = { type };
  if (templateId) {
    query.template = templateId;
  }
  router.push({ name: 'create', query });
}

// Hilfsfunktionen für Icons und Farben (Beispiel)
function getIconForType(type) {
  switch (type) {
    case 'worksheet': return 'mdi-file-document-outline';
    case 'quiz': return 'mdi-help-circle-outline';
    case 'glossary': return 'mdi-book-open-variant';
    default: return 'mdi-file-outline';
  }
}

function getIconColor(type) {
  switch (type) {
    case 'worksheet': return 'success';
    case 'quiz': return 'info';
    case 'glossary': return 'warning';
    default: return 'grey';
  }
}

// Materialien beim Laden der Komponente abrufen (falls noch nicht geschehen)
onMounted(() => {
  if (materialsStore.materials.length === 0) { // Nur laden, wenn Store leer ist
    materialsStore.fetchMaterials();
  }
});

</script>

<style scoped>
.v-card {
  transition: box-shadow 0.2s ease-in-out;
}
.v-card:hover {
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1) !important;
}
.quick-action-card:hover {
   transform: translateY(-2px);
}
</style> 