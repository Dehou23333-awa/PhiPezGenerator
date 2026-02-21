<template>
  <div class="download-page-container">
    <h1>Download Song</h1>

    <section class="song-info-section">
      <h2>Song Information</h2>
      <div v-if="infoPending || difficultyPending">
        <p class="status-message">Loading song details...</p>
      </div>
      <div v-else-if="infoError || difficultyError">
        <p class="error-message">Error loading song details: {{ infoError?.message || difficultyError?.message }}</p>
      </div>
      <div v-else-if="currentSongInfo">
        <p><strong>Name:</strong> {{ currentSongInfo.Name }}</p>
        <p><strong>Composer:</strong> {{ currentSongInfo.Composer }}</p>
        <p><strong>Illustrator:</strong> {{ currentSongInfo.illustrator }}</p>
      </div>
      <div v-else>
        <p class="status-message">Song details not found for this ID.</p>
      </div>
    </section>

    <hr>

    <section class="media-files-section">
      <h2>Media Files</h2>
      <p><strong>Music File:</strong> {{ decodedFileName }}.ogg</p>

      <div v-if="illustrationPending">
        <p class="status-message">Searching for illustration...</p>
      </div>
      <div v-else-if="illustrationError">
        <p class="error-message">Error searching for illustration: {{ illustrationError.message }}</p>
      </div>
      <div v-else-if="illustrationFileName">
        <p><strong>Illustration:</strong> {{ illustrationFileName.split('/').pop() }}</p>
      </div>
      <div v-else>
        <p class="status-message">No illustration found for this song.</p>
      </div>
    </section>

    <hr>

    <section class="chart-files-section">
      <h2>Chart Files</h2>
      <div v-if="chartPending">
        <p class="status-message">Searching for chart files...</p>
      </div>
      <div v-else-if="chartError">
        <p class="error-message">Error searching for charts: {{ chartError.message }}</p>
      </div>
      <div v-else-if="chartFiles.length > 0">
        <label for="chart-select">Select a chart difficulty:</label>
        <select id="chart-select" v-model="selectedChartFileName" :disabled="isDownloading">
          <option disabled value="">Please select one</option>
          <option v-for="chart in chartFiles" :key="chart.path" :value="chart.path">
            {{ formatChartOption(chart.path) }}
          </option>
        </select>
        <p v-if="selectedChartFileName"><strong>Selected Chart:</strong> {{ selectedChartFileName.replace(/\.json$/, '').toUpperCase() }}</p>
        <p v-if="selectedChartInfo"><strong>Level:</strong> Lv.{{ selectedChartInfo.level }} | <strong>Charter:</strong> {{ selectedChartInfo.charter }}</p>
      </div>
      <div v-else>
        <p class="status-message">No chart files found for this song.</p>
      </div>
    </section>

    <hr>

    <div class="download-action-area">
      <button @click="downloadCombinedFiles" :disabled="isDownloadDisabled">
        {{ isDownloading ? 'Preparing Download...' : 'Download Song Package (.PEZ)' }}
      </button>

      <p v-if="downloadError" class="error-message">Error preparing download: {{ downloadError.message }}</p>
      <p v-if="isDownloading" class="status-message download-message">
        Please wait, zipping files... {{ downloadProgress > 0 ? downloadProgress + '%' : '' }}
      </p>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue';
import { useRoute, useFetch } from '#app';
import JSZip from 'jszip';
import { saveAs } from 'file-saver';

const levels = ["EZ", "HD", "IN", "AT"];

const route = useRoute();
const encodedFileName = route.params.fileName;

// Reactive state for download process
const isDownloading = ref(false);
const downloadError = ref(null);
// New reactive variable for download progress percentage
const downloadProgress = ref(0);

// Decoded music file name (song ID)
const decodedFileName = computed(() => {
  if (encodedFileName) {
    // Remove .ogg extension if present in the route param for consistent ID usage
    return decodeURIComponent(encodedFileName).replace(/\.ogg$/, '');
  }
  return '';
});

// --- Chart Info Data Fetching ---
const chartInfoUrl = 'https://raw.githubusercontent.com/Dehou23333-awa/PhiResources/main/Chart_info.json';
const { data: chartInfoData, pending: infoPending, error: infoError } = useFetch(chartInfoUrl, { parseResponse: JSON.parse });

// Kept for template compatibility (no separate difficulty fetch needed)
const difficultyPending = ref(false);
const difficultyError = ref(null);

const currentSongInfo = computed(() => {
  if (chartInfoData.value && chartInfoData.value.Songs && decodedFileName.value) {
    return chartInfoData.value.Songs[decodedFileName.value] || null;
  }
  return null;
});

// --- Music File URL ---
const musicDownloadUrl = computed(() => {
  return decodedFileName.value
    ? `https://raw.githubusercontent.com/Dehou23333-awa/PhiResources/main/music/${decodedFileName.value}.ogg`
    : null;
});

// --- Illustration Logic ---
// Kept for template compatibility (illustration URL is computed directly)
const illustrationPending = ref(false);
const illustrationError = ref(null);

const illustrationFileName = computed(() => {
  return decodedFileName.value ? `${decodedFileName.value}.png` : null;
});

const illustrationDownloadUrl = computed(() => {
  return illustrationFileName.value
    ? `https://raw.githubusercontent.com/Dehou23333-awa/PhiResources/main/ill/${illustrationFileName.value}`
    : null;
});

// --- Chart Logic ---
const chartPending = computed(() => infoPending.value);
const chartError = computed(() => infoError.value);

const chartFiles = computed(() => {
  if (!currentSongInfo.value) return [];
  return levels
    .filter(level => currentSongInfo.value[level])
    .map(level => ({ path: `${level}.json` }));
});

const selectedChartFileName = ref('');

const selectedChartInfo = computed(() => {
  if (selectedChartFileName.value && currentSongInfo.value) {
    const difficultyKey = selectedChartFileName.value.replace('.json', '').toUpperCase();
    const levelData = currentSongInfo.value[difficultyKey];
    if (levelData) {
      return {
        level: levelData.difficulty || '?',
        charter: levelData.charter || '未知'
      };
    }
  }
  return null;
});

const selectedChartDownloadUrl = computed(() => {
  if (selectedChartFileName.value && decodedFileName.value) {
    return `https://raw.githubusercontent.com/Dehou23333-awa/PhiResources/main/chart/${decodedFileName.value}.0/${selectedChartFileName.value}`;
  }
  return null;
});

const formatChartOption = (chartPath) => {
  const difficultyKey = chartPath.replace('.json', '').toUpperCase();
  let display = difficultyKey;
  if (currentSongInfo.value && currentSongInfo.value[difficultyKey]) {
    const levelData = currentSongInfo.value[difficultyKey];
    display += ` Lv.${levelData.difficulty || '?'}`;
    if (levelData.charter) {
      display += ` - ${levelData.charter}`;
    }
  }
  return display;
};

// --- Download Logic ---

// Helper function to fetch a file and add it to the zip
const fetchAndAddFileToZip = async (zipInstance, url, fileNameInZip, friendlyName, isOptional = false) => {
  if (!url) {
    if (!isOptional) {
      throw new Error(`${friendlyName} URL is not available.`);
    } else {
      console.log(`Optional file ${friendlyName} URL not found, skipping.`);
      return;
    }
  }
  console.log(`Fetching ${friendlyName}: ${url}`);
  try {
    const response = await fetch(url);
    if (!response.ok) {
      throw new Error(`Status: ${response.status} ${response.statusText}`);
    }
    const blob = await response.blob();
    zipInstance.file(fileNameInZip, blob);
    console.log(`Added ${fileNameInZip} to zip.`);
  } catch (error) {
    if (!isOptional) {
      throw new Error(`Failed to fetch ${friendlyName}: ${error.message}`);
    } else {
      console.warn(`Warning: Could not fetch optional file ${friendlyName} (${url}): ${error.message}. Continuing without it.`);
    }
  }
};

// Determines if the download button should be disabled
const isDownloadDisabled = computed(() => {
  return (
    isDownloading.value ||
    !decodedFileName.value ||
    !currentSongInfo.value || // Ensure song info is loaded
    infoPending.value ||
    infoError.value ||
    (chartFiles.value.length > 0 && !selectedChartFileName.value) || // If charts exist, one must be selected
    (chartFiles.value.length === 0 && !musicDownloadUrl.value) // If no charts, at least music file should be available
  );
});

const downloadCombinedFiles = async () => {
  isDownloading.value = true;
  downloadError.value = null; // Clear previous errors
  downloadProgress.value = 0; // Reset progress at the start of a new download

  const zip = new JSZip();

  try {
    // Essential pre-checks
    if (!currentSongInfo.value || !decodedFileName.value) {
      throw new Error("Essential song information or file ID is missing.");
    }
    if (chartFiles.value.length > 0 && (!selectedChartInfo.value || !selectedChartFileName.value)) {
      throw new Error("Please select a chart to download.");
    }

    const selectedDifficultyKey = selectedChartFileName.value ? selectedChartFileName.value.replace('.json', '').toUpperCase() : 'N/A';

    // 1. Generate and add info.txt
    const infoTxtContent = `#
Name: ${currentSongInfo.value.Name}
Song: ${decodedFileName.value}.ogg
Picture: ${decodedFileName.value}.png
Chart: ${selectedChartFileName.value || 'N/A'}
Level: ${selectedDifficultyKey} Lv.${selectedChartInfo.value?.level || '?'}
Composer: ${currentSongInfo.value.Composer}
Illustrator: ${currentSongInfo.value.illustrator}
Charter: ${selectedChartInfo.value?.charter || '未知'}`;

    zip.file("info.txt", infoTxtContent);
    console.log("Generated info.txt");

    // 2. Fetch and Add Music File (mandatory)
    await fetchAndAddFileToZip(zip, musicDownloadUrl.value, `${decodedFileName.value}.ogg`, 'music file');

    // 3. Fetch and Add Illustration File (optional)
    await fetchAndAddFileToZip(zip, illustrationDownloadUrl.value, `${decodedFileName.value}.png`, 'illustration file', true);

    // 4. Fetch and Add Selected Chart File (optional if no charts exist for the song)
    if (selectedChartFileName.value && selectedChartDownloadUrl.value) {
      await fetchAndAddFileToZip(zip, selectedChartDownloadUrl.value, selectedChartFileName.value, 'chart file');
    } else if (chartFiles.value.length > 0) {
        // This branch should ideally not be hit if previous checks are correct, but for safety:
        console.warn("Chart file selected but URL is missing, or no chart was selected despite charts existing. Skipping chart file.");
    } else {
      console.log("No chart files available for this song, skipping chart inclusion.");
    }


    // 5. Generate and save ZIP file (PEZ)
    const zipFileName = `${decodedFileName.value}${selectedChartFileName.value ? `.${selectedDifficultyKey}` : ''}.pez`;
    console.log(`Generating zip file: ${zipFileName}`);
    const zipBlob = await zip.generateAsync({
      type: 'blob',
      compression: 'DEFLATE', // Use DEFLATE compression
      compressionOptions: {
        level: 9 // Max compression level
      }
    }, (metadata) => {
      // Update download progress based on JSZip's internal progress
      downloadProgress.value = Math.floor(metadata.percent);
      console.log(`Zip progress: ${downloadProgress.value}% - Current file: ${metadata.currentFile}`);
    });
    saveAs(zipBlob, zipFileName);
    console.log(`Download initiated for ${zipFileName}`);

  } catch (error) {
    console.error('Error during combined download:', error);
    downloadError.value = error;
  } finally {
    isDownloading.value = false;
    downloadProgress.value = 0; // Reset progress when download finishes or fails
  }
};

// Watch for chartFiles change and auto-select a default chart
watch(chartFiles, (newChartFiles) => {
  if (newChartFiles.length > 0 && !selectedChartFileName.value) {
    // Prefer 'EZ.json', otherwise select the first chart in the sorted list
    const defaultChart = newChartFiles.find(chart => chart.path.toLowerCase() === 'ez.json') || newChartFiles[0];
    if (defaultChart) {
      selectedChartFileName.value = defaultChart.path;
    }
  } else if (newChartFiles.length === 0) {
    selectedChartFileName.value = ''; // Clear selection if no charts are found
  }
}, { immediate: true }); // Run immediately on component mount

// Watch for decodedFileName change and clear selected chart
watch(decodedFileName, () => {
  selectedChartFileName.value = '';
});
</script>

<style scoped>
/* ... (保持原有的 CSS 样式不变) ... */
.download-page-container {
  max-width: 800px;
  margin: 40px auto;
  padding: 30px;
  background-color: #f9f9f9;
  border-radius: 10px;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
  line-height: 1.6;
  color: #333;
}

h1 {
  font-size: 2.2em;
  color: #2c3e50;
  text-align: center;
  margin-bottom: 30px;
  border-bottom: 2px solid #eee;
  padding-bottom: 15px;
}

h2 {
  font-size: 1.5em;
  color: #34495e;
  margin-top: 25px;
  margin-bottom: 15px;
  padding-left: 10px;
  border-left: 4px solid #42b983; /* Vue green accent */
}

p {
  margin-bottom: 10px;
  font-size: 1em;
}

p strong {
  color: #555;
}

.status-message {
  color: #666;
  font-style: italic;
}

.error-message {
  color: #e74c3c; /* Red for errors */
  font-weight: bold;
}

.download-message {
  color: #3498db; /* Blue for downloading status */
  font-weight: bold;
  margin-top: 15px;
}

hr {
  margin: 30px 0;
  border: 0;
  border-top: 1px dashed #ddd;
}

label {
  display: block;
  margin-bottom: 8px;
  font-weight: bold;
  color: #34495e;
}

select {
  width: 100%;
  padding: 12px 15px;
  border-radius: 8px;
  border: 1px solid #ccc;
  font-size: 1em;
  background-color: #fff;
  box-shadow: inset 0 1px 3px rgba(0, 0, 0, 0.05);
  appearance: none; /* Removes default browser styling */
  /* Custom arrow for select */
  background-image: url('data:image/svg+xml;charset=US-ASCII,%3Csvg%20xmlns%3D%22http%3A%2F%2Fwww.w3.org%2F2000%2Fsvg%22%20width%3D%22292.4%22%20height%3D%22292.4%22%3E%3Cpath%20fill%3D%22%23555%22%20d%3D%22M287%2069.4a17.6%2017.6%200%200%200-13.6-6.4c-5%200-9.9%201.7-13.6%206.4L146.2%20220.2%2042.2%2069.4c-3.7-4.7-8.6-6.4-13.6-6.4s-9.9%201.7-13.6%206.4c-4.7%203.7-6.4%208.6-6.4%2013.6s1.7%209.9%206.4%2013.6l135.9%20135.9c4.7%204.7%2010.8%207.2%2017.2%207.2s12.5-2.5%2017.2-7.2L287%2096.6c4.7-3.7%206.4-8.6%206.4-13.6s-1.7-9.9-6.4-13.6z%22%2F%3E%3C%2Fsvg%3E');
  background-repeat: no-repeat;
  background-position: right 15px center;
  background-size: 12px auto;
}

select:disabled {
  background-color: #e9ecef;
  cursor: not-allowed;
  color: #6c757d;
}

button {
  background-color: #42b983; /* Vue green */
  color: white;
  padding: 12px 25px;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 1.1em;
  font-weight: bold;
  margin-top: 20px;
  transition: background-color 0.3s ease, transform 0.1s ease;
  width: 100%; /* Make button full width within its container */
}

button:hover:not(:disabled) {
  background-color: #368a68;
  transform: translateY(-2px); /* Subtle lift effect */
}

button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
  opacity: 0.7;
}

.download-action-area {
  text-align: center;
  margin-top: 40px;
}

/* Section styling for visual grouping */
.song-info-section,
.media-files-section,
.chart-files-section,
.download-action-area {
  background-color: #ffffff;
  padding: 20px;
  border-radius: 8px;
  margin-bottom: 20px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
}

/* Responsive adjustments */
@media (max-width: 600px) {
  .download-page-container {
    margin: 20px;
    padding: 20px;
  }

  h1 {
    font-size: 1.8em;
  }

  h2 {
    font-size: 1.3em;
  }

  button {
    padding: 10px 20px;
    font-size: 1em;
  }
}
</style>