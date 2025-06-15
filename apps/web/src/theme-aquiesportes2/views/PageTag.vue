<template>
    <div class="w-full relative bg-gray-100">
        <div class="lg:max-w-[1200px] md:max-w-3xl mx-auto px-4 py-6">
            <div v-if="!data.tag" class="bg-white rounded-lg p-6">
                <div class="text-center">
                    <h1 class="text-2xl font-bold text-neutral-800 mb-4">Tag não encontrada</h1>
                    <p class="text-neutral-600">A tag que você está procurando não existe ou está indisponível.</p>
                </div>
            </div>

            <div v-else>
                <header class="pb-3 w-full">
                    <div class="border-2 border-[#001E62] py-2 px-4 rounded-md w-full bg-gradient-to-r from-[#001E62] to-[#0056b3] text-white">
                        <div class="flex flex-col">
                            <h1 class="text-2xl font-bold text-left">Tag: {{ data.tag.name }}</h1>
                            <div class="text-xs mt-0.5">
                                {{ data.tag.postCount }} posts
                            </div>
                            <p v-if="data.tag.description" class="text-gray-200 mt-1 text-sm">{{ data.tag.description }}</p>
                        </div>
                    </div>
                </header>

                <!-- Initial loading state -->
                <div v-if="loading && posts.length === 0" class="flex justify-center items-center py-20 bg-white rounded-lg">
                    <div class="animate-spin rounded-full h-12 w-12 border-t-2 border-b-2 border-[#001E62]"></div>
                </div>

                <!-- Posts List -->
                <div v-else-if="posts.length > 0" class="space-y-4">
                    <!-- Grid de posts -->
                    <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
                        <div v-for="post in posts" :key="post.id" class="bg-white rounded-lg shadow-md overflow-hidden border border-gray-300 transition-transform hover:-translate-y-1 duration-300">
                            <a :href="`/post/${post.slug}`" class="block" aria-label="Ler mais sobre este post">
                                <div v-if="post.featureImage" class="relative aspect-video overflow-hidden">
                                    <OptimizedImage
                                        :src="post.featureImage"
                                        :alt="post.featureImageAlt || post.title"
                                        class="w-full h-full object-cover"
                                        loading="lazy"
                                        width="400"
                                        height="225"
                                        :hover="true"
                                    />
                                    <!-- Tag badge no canto inferior esquerdo em amarelo -->
                                    <div class="absolute left-3 bottom-3 bg-[#ffcc00] text-[#001E62] text-xs font-semibold px-3 py-1 rounded-full">
                                        {{ data.tag.name }}
                                    </div>
                                    
                                    <!-- Botão de play, se for conteúdo de vídeo (exemplo) -->
                                    <div v-if="post.title.toLowerCase().includes('vídeo') || post.title.toLowerCase().includes('video')" 
                                         class="absolute right-3 bottom-3 bg-black/30 text-white text-xs px-2 py-1 rounded flex items-center">
                                        <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4 mr-1" viewBox="0 0 20 20" fill="currentColor">
                                            <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zM9.555 7.168A1 1 0 008 8v4a1 1 0 001.555.832l3-2a1 1 0 000-1.664l-3-2z" clip-rule="evenodd" />
                                        </svg>
                                        4:19
                                    </div>
                                </div>
                                <div class="p-4">
                                    <div class="flex items-center mb-2 text-xs text-neutral-500">
                                        <span>{{ formatDate(post.publishedAt || post.updatedAt) }}</span>
                                    </div>
                                    <h3 class="font-bold text-neutral-900">{{ post.title }}</h3>
                                </div>
                            </a>
                        </div>
                    </div>
                </div>

                <!-- No posts state -->
                <div v-else-if="!loading && posts.length === 0" class="text-center py-16 bg-white rounded-lg">
                    <h2 class="text-2xl font-bold mb-2 text-neutral-800">Nenhum post encontrado com esta tag</h2>
                    <p class="text-neutral-600">Volte mais tarde para novos conteúdos!</p>
                </div>

                <!-- Loading more indicator -->
                <div v-if="loadingMore" class="mt-8 flex justify-center items-center py-6">
                    <div class="animate-spin rounded-full h-8 w-8 border-t-2 border-b-2 border-[#001E62]"></div>
                </div>

                <!-- Intersection observer target -->
                <div ref="observerTarget" class="h-4 w-full"></div>
            </div>
        </div>
    </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted, watch } from 'vue';
import { useRoute } from 'vue-router';
import { useHead } from '@unhead/vue';
import { vue3 } from '@cmmv/blog/client';
import OptimizedImage from '../../components/OptimizedImage.vue';
import { useSettingsStore } from '../../store/settings';

import {
    formatDate, stripHtml
} from '../../composables/useUtils';

const settingsStore = useSettingsStore();
const blogAPI = vue3.useBlog();
const route = useRoute();

const data = ref<any>(await blogAPI.tags.getPostsBySlug(route.params.slug as string));
const posts = ref<any[]>(data.value.posts || []);
const settings = ref<any>(settingsStore.getSettings);
const loading = ref(false);
const loadingMore = ref(false);
const hasMorePosts = ref(true);
const currentPage = ref(0);
const observerTarget = ref<HTMLElement | null>(null);
const observer = ref<IntersectionObserver | null>(null);

const pageUrl = computed(() => {
    // @ts-ignore
    return `${import.meta.env.VITE_WEBSITE_URL}/tag/${data.value?.tag?.slug || ''}`
})

const headData = ref({
    title: data.value?.tag?.name + ' - ' + settings.value['blog.title'],
    meta: [
        { name: 'description', content: data.value?.tag?.description },
        { name: 'keywords', content: settings.value['blog.keywords'] },
        { property: 'og:type', content: 'website' },
        { property: 'og:title', content: data.value?.tag?.name + ' - ' + settings.value['blog.title'] },
        { property: 'og:description', content: data.value?.tag?.description },
        { property: 'og:image', content: settings.value['blog.logo'] },
        { property: 'og:url', content: pageUrl.value }
    ],
    link: [
        { rel: 'canonical', href: pageUrl.value }
    ]
})

useHead(headData);

const loadMorePosts = async () => {
    if (loadingMore.value || !hasMorePosts.value) return;

    try {
        loadingMore.value = true;
        currentPage.value++;

        const response = await blogAPI.tags.getPostsBySlug(route.params.slug as string, posts.value.length);

        if (response && response.posts && response.posts.length > 0) {
            posts.value = [...posts.value, ...response.posts];
            hasMorePosts.value = posts.value.length < (response.total || 0);
        } else {
            hasMorePosts.value = false;
        }
    } catch (err) {
        console.error('Failed to load more posts:', err);
    } finally {
        loadingMore.value = false;
    }
};

const setupIntersectionObserver = () => {
    observer.value = new IntersectionObserver(
        (entries) => {
            const [entry] = entries;
            if (entry.isIntersecting && hasMorePosts.value && !loadingMore.value) {
                loadMorePosts();
            }
        },
        { threshold: 0.1 }
    );

    if (observerTarget.value) {
        observer.value.observe(observerTarget.value);
    }
};

onMounted(async () => {
    loading.value = false;
    setupIntersectionObserver();
});

onUnmounted(() => {
    if (observer.value && observerTarget.value) {
        observer.value.unobserve(observerTarget.value);
        observer.value.disconnect();
    }
});
</script>

<style scoped>
.article-container {
    max-width: 100%;
    margin: 0 auto;
}
</style>
