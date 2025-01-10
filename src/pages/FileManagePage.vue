<script setup lang="ts">
import { onBeforeMount, ref, type Ref } from 'vue';
import { formatBytes } from '../utils/utils';
import { DeleteFile, ListFiles } from '../api';
import type { _Object } from '@aws-sdk/client-s3';

const { localStorage } = window;

let uploadedFiles: Ref<_Object[]> = ref([]);

const refreshFiles = async () => {
    const res = await ListFiles();
    if (res.Contents) {
        uploadedFiles.value = res.Contents.map(file => ({
            ...file,
            Key: file.Key ? decodeURIComponent(file.Key) : ''
        }));
    }
};

onBeforeMount(async () => {
    await refreshFiles();
});

const onDeleteFileClick = async (key?: string) => {
    if (!key) {
        return;
    }
    await DeleteFile(key);
    await refreshFiles();
};
</script>

<template>
    <div class="flex flex-col items-center mt-5">
        <h1 class="text-lg">{{ $t("page_title.filemanage") }}</h1>
        <div class="px-4 py-4 max-w-screen-md w-4/5">
            <div v-for="file in uploadedFiles.sort((a, b) => {
                const aIsClip = a.Key?.startsWith('clip_') ? 1 : 0;
                const bIsClip = b.Key?.startsWith('clip_') ? 1 : 0;
                return aIsClip - bIsClip;
            })" :key="file.Key"
                class="w-full flex flex-row items-center mt-4 rounded border-1 border-gray-300 px-2 py-1">
                <div class="w-10 h-10 i-mdi-file-document-outline" :class="{'text-green-500': file.Key?.startsWith('clip_')}"></div>
                <div class="flex flex-col">
                    <a class="text-lg font-semibold" :href="`/${file.Key}`" target="_blank" :class="{'text-green-500': file.Key?.startsWith('clip_')}">{{ file.Key ? decodeURIComponent(file.Key) : '' }}</a>
                    <div class="text-sm text-gray">{{ formatBytes(file.Size ?? 0) }} · {{ file.LastModified ? new Date(file.LastModified).toLocaleString() : '' }}</div>
                </div>
                <div class="ml-auto flex gap-2">
                  <div class="w-6 h-6 i-mdi-pencil-outline cursor-pointer"
                      @click="() => { 
                        localStorage.setItem('openFile', JSON.stringify({
                          filename: file.Key || '',
                          contentUrl: `/${file.Key}`,
                          timestamp: file.LastModified
                        }));
                        $router.push(`/clip?open=${file.Key}`) 
                      }"></div>
                  <div class="w-6 h-6 i-mdi-trash-can-outline cursor-pointer"
                      @click="onDeleteFileClick(file.Key)"></div>
                </div>
            </div>
        </div>
    </div>
</template>

<style>
html,
body,
#app {
    margin: 0;
    padding: 0;
    background-color: #f8f9fa;
}
</style>
