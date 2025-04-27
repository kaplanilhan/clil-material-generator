<template>
  <div>
    <v-toolbar density="compact" flat class="mb-4">
       <v-btn icon @click="router.push('/materials')">
           <v-icon>mdi-arrow-left</v-icon>
       </v-btn>
        <v-toolbar-title class="text-h6">
            <span v-if="material">{{ material.title || 'Material bearbeiten' }}</span>
            <v-skeleton-loader v-else type="text" width="200"></v-skeleton-loader>
        </v-toolbar-title>
        <v-spacer></v-spacer>
        <v-chip size="small" class="mr-2" :color="saveStatusColor" variant="tonal">
          <v-icon start size="small">{{ saveStatusIcon }}</v-icon>
          {{ saveStatusText }}
        </v-chip>
        <!-- Weitere Aktionen wie Export, Teilen könnten hierhin -->
        <v-btn color="primary" @click="forceSave">
            <v-icon start>mdi-content-save</v-icon>
            Speichern
        </v-btn>
    </v-toolbar>

    <v-row>
      <!-- Haupt-Editor-Bereich -->
      <v-col cols="12" md="8">
          <v-skeleton-loader v-if="loading" type="image, article"></v-skeleton-loader>
          <material-editor
            v-else-if="material"
            v-model="editableContent"
            @save-status="updateSaveStatus"
          />
          <v-alert v-else type="error">
              Material konnte nicht geladen werden.
          </v-alert>
      </v-col>

      <!-- Sidebar für Metadaten -->
      <v-col cols="12" md="4">
        <v-card elevation="1" v-if="material">
           <v-card-title class="text-subtitle-1">Metadaten</v-card-title>
           <v-divider></v-divider>
           <v-card-text>
              <v-text-field
                v-model="editableTitle"
                label="Titel"
                variant="outlined"
                density="compact"
                class="mb-3"
                @input="markUnsaved"
                @blur="saveTitle"
              ></v-text-field>
              <v-select
                v-model="editableType"
                :items="materialTypes"
                item-title="title"
                item-value="id"
                label="Material-Typ"
                variant="outlined"
                density="compact"
                class="mb-3"
                @update:modelValue="saveMetadata('type', editableType)"
              ></v-select>
               <v-select
                v-model="editableSubject"
                :items="subjects"
                label="Fach"
                variant="outlined"
                density="compact"
                class="mb-3"
                @update:modelValue="saveMetadata('subject', editableSubject)"
              ></v-select>

               <v-divider class="my-3"></v-divider>
                <h4 class="text-subtitle-2 mb-2">CLIL-Parameter</h4>
               <v-select
                  v-model="editableLanguageLevel"
                  :items="languageLevels"
                  label="Sprachniveau"
                  variant="outlined"
                  density="compact"
                  class="mb-3"
                   @update:modelValue="saveMetadata('language.level', editableLanguageLevel)"
                ></v-select>
                <v-slider
                    v-model="editableVocabPercentage"
                    :min="10"
                    :max="50"
                    :step="5"
                    label="Fachvokabular (%)"
                    color="primary"
                    thumb-label
                    class="mb-1"
                    @update:modelValue="saveMetadata('language.vocabPercentage', editableVocabPercentage)"
                >
                    <template v-slot:append>
                        <v-chip size="small" label>{{ editableVocabPercentage }}%</v-chip>
                    </template>
                </v-slider>

               <v-divider class="my-3"></v-divider>
                <div class="text-caption">
                    Erstellt: {{ formatDate(material.created) }}
                </div>
                <div class="text-caption">
                    Zuletzt geändert: {{ formatDate(material.modified) }}
                </div>

           </v-card-text>
           <v-divider></v-divider>
           <v-card-actions>
               <v-spacer></v-spacer>
                <v-btn color="error" variant="text" @click="confirmDeleteDialog = true">
                   <v-icon start>mdi-delete-outline</v-icon>
                   Löschen
               </v-btn>
           </v-card-actions>
        </v-card>
         <v-skeleton-loader v-else type="card"></v-skeleton-loader>
      </v-col>
    </v-row>

     <!-- Delete Confirmation Dialog -->
    <v-dialog v-model="confirmDeleteDialog" max-width="400">
      <v-card>
        <v-card-title class="text-h5">Löschen bestätigen</v-card-title>
        <v-card-text>
          Sind Sie sicher, dass Sie dieses Material endgültig löschen möchten?
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn variant="text" @click="confirmDeleteDialog = false">Abbrechen</v-btn>
          <v-btn color="error" @click="deleteMaterial" :loading="deleting">Löschen</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch } from 'vue';
import { useRoute, useRouter } from 'vue-router';
import { useMaterialsStore } from '@/stores/materials';
import MaterialEditor from '@/components/Editor/MaterialEditor.vue';

const route = useRoute();
const router = useRouter();
const materialsStore = useMaterialsStore();

const loading = ref(true);
const material = ref(null);
const editableContent = ref('');
const editableTitle = ref('');
const editableType = ref('');
const editableSubject = ref('');
const editableLanguageLevel = ref('B1');
const editableVocabPercentage = ref(30);
const saveStatus = ref('saved'); // Synced with editor's status
const confirmDeleteDialog = ref(false);
const deleting = ref(false);
const titleSaveTimeout = ref(null);

// Fetch material data when component mounts
onMounted(async () => {
  loading.value = true;
  const materialId = route.params.id;
  // Try to get from store first
  material.value = materialsStore.getMaterialById(materialId);
  if (!material.value) {
    // If not in store (e.g., direct navigation), try fetching (if API exists)
    // For mock, we assume it must be in the store if valid
    console.warn(`Material with ID ${materialId} not found in store.`);
    // Optionally redirect or show error
    // router.push('/materials');
  }
  if(material.value) {
      editableContent.value = material.value.content || '';
      editableTitle.value = material.value.title || '';
      editableType.value = material.value.type || '';
      editableSubject.value = material.value.subject || '';
      editableLanguageLevel.value = material.value.language?.level || 'B1';
      editableVocabPercentage.value = material.value.language?.vocabPercentage || 30;
  }
  loading.value = false;
});

// Data for selects (could be fetched or defined globally)
const materialTypes = [
  { id: 'worksheet', title: 'Arbeitsblatt' },
  { id: 'quiz', title: 'Quiz' },
  { id: 'glossary', title: 'Glossar' },
  // Add other types
];
const subjects = [
  'Informatik', 'Elektrotechnik', 'Maschinenbau', 'Mechatronik', 'Netzwerktechnik', 'Elektronik', 'Datenbanken', 'Webentwicklung', 'Mathematik', 'Physik'
];
const languageLevels = [
  { title: 'A1', value: 'A1' }, { title: 'A2', value: 'A2' }, { title: 'B1', value: 'B1' }, { title: 'B2', value: 'B2' }, { title: 'C1', value: 'C1' }
];

// Update save status from editor component
const updateSaveStatus = (status) => {
  saveStatus.value = status;
};

// Computed properties for save status display
const saveStatusText = computed(() => {
    switch(saveStatus.value) {
        case 'saved': return 'Gespeichert';
        case 'unsaved': return 'Ungespeichert';
        case 'saving': return 'Speichern...';
        case 'error': return 'Fehler';
        default: return ''
    }
});
const saveStatusIcon = computed(() => {
     switch(saveStatus.value) {
        case 'saved': return 'mdi-check-circle-outline';
        case 'unsaved': return 'mdi-alert-circle-outline';
        case 'saving': return 'mdi-sync';
        case 'error': return 'mdi-close-circle-outline';
        default: return ''
    }
});
const saveStatusColor = computed(() => {
     switch(saveStatus.value) {
        case 'saved': return 'success';
        case 'unsaved': return 'warning';
        case 'saving': return 'info';
        case 'error': return 'error';
        default: return 'grey'
    }
});

// --- Metadata Saving --- //

const markUnsaved = () => {
    if (saveStatus.value === 'saved') {
        saveStatus.value = 'unsaved';
    }
}

const saveMetadata = async (field, value) => {
    if (!material.value) return;
    markUnsaved(); // Mark as unsaved first
    saveStatus.value = 'saving';
    try {
        let updateData = { id: material.value.id };
        // Handle nested language object
        if (field.startsWith('language.')) {
            const langField = field.split('.')[1];
            updateData.language = { ...material.value.language, [langField]: value };
        } else {
            updateData[field] = value;
        }

        await materialsStore.updateMaterial(updateData);
        // Update local ref *after* successful save
        if (field.startsWith('language.')) {
             const langField = field.split('.')[1];
             if (material.value.language) material.value.language[langField] = value;
        } else {
            material.value[field] = value;
        }
        saveStatus.value = 'saved';
    } catch (error) {
        console.error(`Error saving metadata field ${field}:`, error);
        saveStatus.value = 'error';
         // Show error message to user
    }
};

// Debounced title saving
const saveTitle = () => {
    if (titleSaveTimeout.value) clearTimeout(titleSaveTimeout.value);
    titleSaveTimeout.value = setTimeout(() => {
        if (material.value && editableTitle.value !== material.value.title) {
            saveMetadata('title', editableTitle.value);
        } else if (saveStatus.value === 'unsaved') {
             // If only title changed and is now back to original, mark as saved
             // This needs more robust checking if other fields could be unsaved
             // saveStatus.value = 'saved';
        }
    }, 800); // Debounce delay
};

watch(editableTitle, () => {
    markUnsaved();
    if (titleSaveTimeout.value) clearTimeout(titleSaveTimeout.value);
    // Start timer on input change
    titleSaveTimeout.value = setTimeout(saveTitle, 1500);
});


// Trigger save (e.g., by button click)
const forceSave = async () => {
    if (!material.value) return;
    saveStatus.value = 'saving';
    try {
        await materialsStore.updateMaterial({
            id: material.value.id,
            content: editableContent.value, // Save latest content
            title: editableTitle.value, // Ensure title is saved too
             // Include other potentially changed metadata if needed
        });
        saveStatus.value = 'saved';
    } catch (error) {
        console.error("Error force saving:", error);
        saveStatus.value = 'error';
    }
}

// Delete material
const deleteMaterial = async () => {
  if (!material.value) return;
  deleting.value = true;
  try {
    await materialsStore.deleteMaterial(material.value.id);
    confirmDeleteDialog.value = false;
    router.push('/materials'); // Navigate back to list after deletion
  } catch (error) {
    console.error("Error deleting material:", error);
    // Show error message
  } finally {
    deleting.value = false;
  }
};

// Helper to format date
const formatDate = (dateString) => {
  if (!dateString) return 'N/A';
  try {
      return new Date(dateString).toLocaleString('de-DE', { dateStyle: 'short', timeStyle: 'short' });
  } catch (e) {
      return dateString; // Return original if parsing fails
  }
};

</script>

<style scoped>
/* Add specific styles for the edit view if needed */
.v-toolbar-title {
    flex: 0 1 auto; /* Prevent title from shrinking too much */
    min-width: 150px;
}
</style> 