<template>
  <div class="home-page-container">
    <h1>Phigros Music Library</h1>

    <div v-if="isLoading" class="status-area">
      <p class="status-message">Loading music library...</p>
    </div>
    <div v-else-if="hasError" class="status-area">
      <p class="error-message">Error loading music library: {{ musicError?.message || infoError?.message }}</p>
      <p class="status-message">Please try refreshing the page.</p>
    </div>
    <div v-else class="content-wrapper"> <div class="search-area">
        <input
          type="text"
          v-model="searchQuery"
          placeholder="Search for songs by name or ID..."
          class="search-input"
          aria-label="Search for songs"
        />
      </div>

      <div v-if="filteredFiles.length > 0" class="file-list-area">
        <p class="list-intro">Select a song to download its music, illustration, and chart files:</p>
        <ul class="music-list">
          <li v-for="song in filteredFiles" :key="song.id" class="music-list-item">
            <NuxtLink :to="`/download/${encodeURIComponent(song.id)}`" class="music-link">
              {{ song.name }}
            </NuxtLink>
          </li>
        </ul>
      </div>
      <div v-else-if="searchQuery && filteredFiles.length === 0" class="status-area">
        <p class="status-message">No results found for "{{ searchQuery }}".</p>
      </div>
      <div v-else-if="displayFiles.length === 0" class="status-area">
        <p class="status-message">No music files found.</p>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue';
import { useFetch } from '#app';

// --- API URLs ---
const musicFilesApiUrl = 'https://api.github.com/repos/7aGiven/Phigros_Resource/git/trees/music';
const infoTsvUrl = 'https://raw.githubusercontent.com/7aGiven/Phigros_Resource/info/info.tsv';

// --- Fetch Music Files ---
const {
  data: musicData,
  pending: musicPending,
  error: musicError
} = await useFetch(musicFilesApiUrl);

// --- Fetch Info TSV Data ---
const {
  data: infoTsvData,
  pending: infoPending,
  error: infoError
} = await useFetch(infoTsvUrl);

// --- Parse Info TSV Data ---
const parsedSongNames = computed(() => {
  const namesMap = {};
  if (infoTsvData.value) {
    const lines = infoTsvData.value.trim().split('\n');
    lines.forEach(line => {
      const parts = line.split('\t');
      if (parts.length > 1) { // Ensure at least song ID and Name exist
        const songId = parts[0];
        const songName = parts[1];
        namesMap[songId] = songName;
      }
    });
  }
  return namesMap;
});

// --- Prepare Display Files (All songs, before filtering) ---
const displayFiles = computed(() => {
  if (musicData.value && musicData.value.tree && parsedSongNames.value) {
    return musicData.value.tree
      .filter(item => item.type === 'blob' && item.path.endsWith('.ogg')) // Only consider .ogg files
      .map(item => {
        const fileName = item.path.replace(/\.ogg$/, ''); // Get the song ID (e.g., "Calamity")
        const songName = parsedSongNames.value[fileName] || fileName; // Use name from info.tsv, fallback to ID if not found
        return { id: fileName, name: songName };
      })
      .sort((a, b) => a.name.localeCompare(b.name)); // Sort alphabetically by song name
  }
  return [];
});

// --- Search Functionality ---
const searchQuery = ref(''); // Reactive variable to store the search input

const filteredFiles = computed(() => {
  if (!searchQuery.value) {
    return displayFiles.value; // If no search query, return all songs
  }
  const query = searchQuery.value.toLowerCase();
  return displayFiles.value.filter(song =>
    song.name.toLowerCase().includes(query) || // Search by song name
    song.id.toLowerCase().includes(query)     // Search by song ID
  );
});

// --- Consolidated Loading and Error States ---
const isLoading = computed(() => musicPending.value || infoPending.value);
const hasError = computed(() => musicError.value || infoError.value);
</script>

<style scoped>
.home-page-container {
  max-width: 700px;
  margin: 40px auto;
  padding: 30px;
  background-color: #f9f9f9;
  border-radius: 10px;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  color: #333;
  line-height: 1.6;
}

h1 {
  font-size: 2.2em;
  color: #2c3e50;
  text-align: center;
  margin-bottom: 30px;
  border-bottom: 2px solid #eee;
  padding-bottom: 15px;
}

.status-area {
  text-align: center;
  padding: 20px;
  background-color: #fff;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
  margin-bottom: 20px; /* Add margin for spacing */
}

.status-message {
  color: #666;
  font-style: italic;
  font-size: 1.1em;
}

.error-message {
  color: #e74c3c; /* Red for errors */
  font-weight: bold;
  font-size: 1.1em;
}

/* New wrapper for search and list content */
.content-wrapper {
  padding: 10px; /* Add some internal padding */
}

.search-area {
  margin-bottom: 20px;
  padding: 15px 20px;
  background-color: #ffffff;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
  text-align: center;
}

.search-input {
  width: calc(100% - 40px); /* Account for padding */
  max-width: 400px; /* Optional: limit width of search input */
  padding: 12px 20px;
  border: 1px solid #ddd;
  border-radius: 5px;
  font-size: 1.1em;
  box-shadow: inset 0 1px 3px rgba(0, 0, 0, 0.05);
  transition: border-color 0.3s ease, box-shadow 0.3s ease;
}

.search-input::placeholder {
  color: #999;
}

.search-input:focus {
  border-color: #42b983; /* Vue green focus */
  outline: none;
  box-shadow: 0 0 0 3px rgba(66, 185, 131, 0.2); /* Subtle focus glow */
}

.list-intro {
  margin-top: 20px;
  margin-bottom: 15px;
  font-size: 1.1em;
  color: #555;
  text-align: center;
}

.file-list-area {
  background-color: #ffffff;
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
}

.music-list {
  list-style: none; /* Remove default bullet points */
  padding: 0;
  margin: 0;
  max-height: 500px; /* Limit height for scrollability */
  overflow-y: auto; /* Enable scrolling */
  border: 1px solid #eee;
  border-radius: 5px;
}

.music-list-item {
  padding: 12px 15px;
  border-bottom: 1px solid #eee;
}

.music-list-item:last-child {
  border-bottom: none; /* No border for the last item */
}

.music-list-item:hover {
  background-color: #f0f8f4; /* Light green hover effect */
}

.music-link {
  text-decoration: none;
  color: #007bff; /* Blue link color */
  font-weight: 500;
  font-size: 1.05em;
  display: block; /* Make the entire list item clickable area */
  transition: color 0.2s ease;
}

.music-link:hover {
  color: #0056b3; /* Darker blue on hover */
  text-decoration: underline;
}

/* Basic responsiveness */
@media (max-width: 600px) {
  .home-page-container {
    margin: 20px;
    padding: 20px;
  }

  h1 {
    font-size: 1.8em;
  }

  .search-input {
    width: 100%; /* Full width on small screens */
    max-width: none;
  }

  .music-list-item {
    padding: 10px 12px;
    font-size: 0.95em;
  }
}
</style>