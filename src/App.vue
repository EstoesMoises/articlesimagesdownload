<template>
  <div class="p-5">
    <h1 class="text-2xl font-bold mb-4">Stack Overflow Articles Fetcher</h1>

    <form @submit.prevent="fetchArticles">
      <!-- Team Name Input -->
      <div class="mb-4">
        <label class="block text-sm font-bold mb-2" for="team">instance FQDN:</label>
        <input v-model="fqdn" type="text" id="team" class="border p-2 w-full text-gray-800" required />
      </div>

      <!-- Author ID Input -->
      <div class="mb-4">
        <label class="block text-sm font-bold mb-2" for="authorId">Author ID:</label>
        <input v-model="authorId" type="number" id="authorId" class="border p-2 w-full text-gray-800" required />
      </div>

      <!-- Bearer Token Input -->
      <div class="mb-4">
        <label class="block text-sm font-bold mb-2" for="bearerToken">Bearer Token:</label>
        <input v-model="bearerToken" type="text" id="bearerToken" class="border p-2 w-full text-gray-800" required />
      </div>

      <!-- Submit Button -->
      <button type="submit" class="bg-blue-500 text-white py-2 px-4">Fetch Articles</button>
    </form>

    <!-- Download Images Button -->
    <button v-if="articles.length" @click="downloadImages" class="bg-green-500 text-white py-2 px-4 mt-4">Download All Images</button>

    <!-- Articles Section -->
    <div v-if="articles.length" class="mt-8">
      <h2 class="text-xl font-bold">Articles</h2>
      <ul>
        <li v-for="article in articles" :key="article.id" class="mb-4">
          <h3 class="font-bold">{{ article.title }}</h3>
          <div v-if="article.imageUrls.length" class="mt-2">
            <h4>Images:</h4>
            <ul>
              <li v-for="img in article.imageUrls" :key="img">
                <img :src="img" alt="Article Image" class="max-w-xs">
              </li>
            </ul>
          </div>
          <div>
            <strong>Owner:</strong> {{ article.owner.name }} <br />
          </div>
          <a :href="article.shareUrl" target="_blank" class="text-blue-500 underline">View Article</a>
        </li>
      </ul>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue';
import axios from 'axios';

// Reactive state variables
const fqdn = ref<string>('');
const authorId = ref<number | null>(null);
const bearerToken = ref<string>('');
const articles = ref<Article[]>([]);

// Define TypeScript interfaces
interface Article {
  id: number;
  title: string;
  content: string;
  imageUrls: string[];
  owner: Owner;
  viewCount: number;
  score: number;
  shareUrl: string;
}

interface Owner {
  id: number;
  accountId: number;
  name: string;
  avatarUrl: string;
  reputation: number;
  role: string;
}

// Function to fetch articles
const fetchArticles = async () => {
  if (!fqdn.value || authorId.value === null || !bearerToken.value) {
    console.error('Missing required fields');
    return;
  }

  const apiUrl = `https://${fqdn.value}/api/v3/articles`;
  const params = new URLSearchParams({
    authorId: authorId.value.toString(),
    sort: 'creation',
    order: 'desc',
  });

  console.log(apiUrl)

  try {
    const response = await axios.get(`${apiUrl}?${params}`, {
      headers: {
        Authorization: `Bearer ${bearerToken.value}`,
        'Content-Type': 'application/json',
      },
    });

    // Map the articles data with extracted image URLs and additional properties
    articles.value = response.data.items.map((article: any) => ({
      id: article.id,
      title: article.title,
      content: article.body,
      imageUrls: extractImageUrls(article.body),
      owner: article.owner,
      viewCount: article.viewCount,
      score: article.score,
      shareUrl: article.shareUrl,
    }));
  } catch (error) {
    console.error('Error fetching articles:', error);
  }
};

// Helper function to extract image URLs from article content
const extractImageUrls = (content: string): string[] => {
  const imgUrls: string[] = [];
  const imgRegex = /<img[^>]+src="([^">]+)"/g;
  let match;

  while ((match = imgRegex.exec(content)) !== null) {
    imgUrls.push(match[1]);
  }

  return imgUrls;
};

// Function to download all images
const downloadImages = async () => {
  const allImageUrls = articles.value.flatMap(article => article.imageUrls);
  
  for (const url of allImageUrls) {
    try {
      const response = await fetch(url);
      if (response.ok) {
        const blob = await response.blob();
        const link = document.createElement('a');
        link.href = URL.createObjectURL(blob);
        link.download = url.split('/').pop() || 'download'; // Extract the file name from the URL
        document.body.appendChild(link);
        link.click();
        document.body.removeChild(link);
        URL.revokeObjectURL(link.href); // Clean up URL object
      } else {
        console.error(`Failed to download image: ${url}`);
      }
    } catch (error) {
      console.error(`Error downloading image: ${error}`);
    }
  }
};
</script>

<style>
/* Add Tailwind base styles */
@import "tailwindcss/base";
@import "tailwindcss/components";
@import "tailwindcss/utilities";
</style>
