<template>
  <div>
    <h1>Download Page</h1>
    <p>Music File: {{ decodedFileName }}</p>

    <div v-if="illustrationFileName">
      <p>Found Illustration: {{ illustrationFileName }}</p>
      </div>
    <div v-else-if="illustrationPending">
      <p>Searching for illustration...</p>
    </div>
    <div v-else-if="illustrationError">
      <p>Error searching for illustration: {{ illustrationError.message }}</p>
    </div>
    <div v-else>
      <p>No illustration found for this music file.</p>
    </div>

    <hr>

    <button @click="downloadCombinedFiles" :disabled="isDownloading || !decodedFileName || (illustrationPending)">
      {{ isDownloading ? 'Preparing Download...' : 'Download Music and Illustration (ZIP)' }}
    </button>

    <p v-if="downloadError" style="color: red;">Error preparing download: {{ downloadError.message }}</p>
    <p v-if="isDownloading" style="color: blue;">Please wait, zipping files...</p>

    </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue';
import { useRoute, useFetch } from '#app'; // Assuming Nuxt 3 setup for useRoute and useFetch
import JSZip from 'jszip';
import { saveAs } from 'file-saver';

const route = useRoute();
const encodedFileName = route.params.fileName;

// Reactive state for download process
const isDownloading = ref(false);
const downloadError = ref(null);

// Decodes the music file name from the URL parameter
const decodedFileName = computed(() => {
  if (encodedFileName) {
    return decodeURIComponent(encodedFileName);
  }
  return '';
});

// Extracts the base name of the music file (e.g., "MySong" from "MySong.ogg")
const musicBaseName = computed(() => {
  if (decodedFileName.value) {
    return decodedFileName.value.replace(/\.ogg$/, '');
  }
  return '';
});

// GitHub API URL for the 'illustration' directory content
const illustrationApiUrl = 'https://api.github.com/repos/7aGiven/Phigros_Resource/git/trees/illustration?recursive=1';

// Fetches the 'illustration' directory content using Nuxt's useFetch
const { data: illustrationData, pending: illustrationPending, error: illustrationError } = useFetch(illustrationApiUrl);

// Finds the corresponding illustration file name based on the music base name
const illustrationFileName = computed(() => {
  if (illustrationData.value && illustrationData.value.tree && musicBaseName.value) {
    const foundItem = illustrationData.value.tree.find(item =>
      item.type === 'blob' && // Ensure it's a file
      item.path.startsWith(musicBaseName.value) && // File path starts with music file base name
      item.path.toLowerCase().endsWith('.png') // File path ends with .png (case-insensitive)
    );
    return foundItem ? foundItem.path : null;
  }
  return null;
});

// Constructs the raw download URL for the illustration file
const illustrationDownloadUrl = computed(() => {
  if (illustrationFileName.value) {
    return `https://raw.githubusercontent.com/7aGiven/Phigros_Resource/illustration/${illustrationFileName.value}`;
  }
  return null;
});

// Constructs the raw download URL for the music file
const musicDownloadUrl = computed(() => {
  if (decodedFileName.value) {
    return `https://raw.githubusercontent.com/7aGiven/Phigros_Resource/music/${decodedFileName.value}.ogg`;
  }
  return null;
});

// Function to handle combined download
const downloadCombinedFiles = async () => {
  isDownloading.value = true;
  downloadError.value = null;
  const zip = new JSZip();

  try {
    // 1. Fetch Music File
    if (!musicDownloadUrl.value) {
      throw new Error("Music file URL is not available.");
    }
    const musicResponse = await fetch(musicDownloadUrl.value);
    if (!musicResponse.ok) {
      throw new Error(`Failed to fetch music file: ${musicResponse.statusText}`);
    }
    const musicBlob = await musicResponse.blob();
    zip.file('music.ogg', musicBlob);

    // 2. Fetch Illustration File (if found)
    if (illustrationFileName.value && illustrationDownloadUrl.value) {
      const illustrationResponse = await fetch(illustrationDownloadUrl.value);
      if (!illustrationResponse.ok) {
        console.warn(`Warning: Failed to fetch illustration file: ${illustrationResponse.statusText}. Continuing without illustration.`);
        // Optionally, you could throw an error here if illustration is mandatory
      } else {
        const illustrationBlob = await illustrationResponse.blob();
        const illustrationBaseName = illustrationFileName.value.split('/').pop(); // Get just the file name
        zip.file('illustration.png', illustrationBlob);
      }
    } else {
      console.log('No illustration found to add to the zip.');
    }

    // 3. Generate and Save the ZIP File
    const zipBlob = await zip.generateAsync({ type: 'blob' });
    const zipFileName = `${musicBaseName.value}.pez`;
    saveAs(zipBlob, zipFileName);

  } catch (error) {
    console.error('Error during combined download:', error);
    downloadError.value = error;
  } finally {
    isDownloading.value = false;
  }
};
</script>

<style scoped>
/* Add any component-specific styles here if needed */
div {
  font-family: sans-serif;
  padding: 20px;
}

h1 {
  color: #333;
}

p {
  margin-bottom: 10px;
}

button {
  background-color: #007bff;
  color: white;
  padding: 10px 15px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-size: 16px;
  margin-top: 20px;
}

button:hover:not(:disabled) {
  background-color: #0056b3;
}

button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

a {
  color: #007bff;
  text-decoration: none;
}

a:hover {
  text-decoration: underline;
}

hr {
  margin: 20px 0;
  border: 0;
  border-top: 1px solid #eee;
}
</style>