<template>
  <div>
    <h1>Phigros Music Files</h1>
    <ul v-if="fileNames.length">
      <li v-for="(fileName, index) in fileNames" :key="index">
        <NuxtLink :to="`/download/${encodeURIComponent(fileName)}`">
          {{ fileName }}
        </NuxtLink>
      </li>
    </ul>
    <p v-else-if="pending">Loading...</p>
    <p v-else>Error loading files.</p>
  </div>
</template>

<script setup>
import { ref } from 'vue';
import { useFetch } from '#app';
import { NuxtLink } from '#components';

const url = 'https://api.github.com/repos/7aGiven/Phigros_Resource/git/trees/music';
const fileNames = ref([]);

const { data, pending, error } = await useFetch(url);

if (data.value && data.value.tree) {
  fileNames.value = data.value.tree
    .filter(item => item.type === 'blob')
    .map(item => item.path.replace(/\.ogg$/, ''));
} else if (error.value) {
  console.error('Error fetching data:', error.value);
}
</script>