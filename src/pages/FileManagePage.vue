<script setup lang="ts">
import { onBeforeMount, ref, type Ref } from 'vue';
import { formatBytes } from '../utils/utils';
import { DeleteFile, ListFiles } from '../api';
import type { _Object } from '@aws-sdk/client-s3';
import { useRoute } from 'vue-router';
import { useTimeoutFn } from '@vueuse/core';
import { NTooltip, useMessage } from 'naive-ui';

const { localStorage } = window;
const route = useRoute();

let uploadedFiles: Ref<_Object[]> = ref([]);
const isDeleteEnabled = ref(false);
const hoverStartTime = ref<number | null>(null);
const hoveredItemKey = ref<string | null>(null);
const showTooltip = ref(false);

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
    if (!key || !isDeleteEnabled.value) {
        return;
    }
    await DeleteFile(key);
    await refreshFiles();
};

const startHoverTimer = (key: string) => {
    if (isDeleteEnabled.value) return;
    hoveredItemKey.value = key;
    hoverStartTime.value = Date.now();
    showTooltip.value = true;
    useTimeoutFn(() => {
        if (hoveredItemKey.value === key && Date.now() - (hoverStartTime.value || 0) >= 3000) {
            isDeleteEnabled.value = true;
            showTooltip.value = false;
        }
    }, 3000);
};

const clearHoverTimer = () => {
    if (isDeleteEnabled.value) return;
    hoveredItemKey.value = null;
    hoverStartTime.value = null;
    showTooltip.value = false;
};
</script>

<template>
    <div class="flex flex-col items-center mt-5">
        <h1 class="text-lg">{{ route.query.mode === 'clip' ? $t("page_title.clipmanage") : $t("page_title.filemanage") }}</h1>
        <div class="px-4 py-4 max-w-screen-md w-4/5">
            <div v-for="file in uploadedFiles
                .filter(file => route.query.mode !== 'clip' || file.Key?.startsWith('clip_'))
                .sort((a, b) => {
                    const aIsClip = a.Key?.startsWith('clip_') ? 1 : 0;
                    const bIsClip = b.Key?.startsWith('clip_') ? 1 : 0;
                    return aIsClip - bIsClip;
                })" :key="file.Key"
                class="w-full flex flex-row items-center mt-4 rounded border-1 border-gray-300 px-2 py-1 relative">
                <div class="w-10 h-10 i-mdi-file-document-outline" :class="{'text-green-500': file.Key?.startsWith('clip_')}"></div>
                <div class="flex flex-col">
                    <a class="text-lg font-semibold" :class="{'text-green-500': file.Key?.startsWith('clip_')}"
                        :href="`/${file.Key}`" 
                        target="_blank">
                        {{ file.Key ? decodeURIComponent(file.Key) : '' }}
                    </a>
                    <div class="text-sm text-gray">{{ formatBytes(file.Size ?? 0) }} · {{ file.LastModified ? new Date(file.LastModified).toLocaleString() : '' }}</div>
                </div>
                <div class="ml-auto flex gap-2">
                    <div v-if="route.query.mode === 'clip'" class="w-6 h-6 i-mdi-pencil-outline cursor-pointer"
                        @click="() => { 
                            localStorage.setItem('openFile', JSON.stringify({
                                filename: file.Key || '',
                                contentUrl: `/${file.Key}`,
                                timestamp: file.LastModified
                            }));
                            $router.push(`/clip?open=${file.Key}`) 
                        }"></div>
                    <n-tooltip placement="right" trigger="hover" :show="showTooltip && hoveredItemKey === file.Key">
                        <template #trigger>
                            <div class="w-6 h-6 i-mdi-trash-can-outline"
                                :class="{ 'cursor-pointer': isDeleteEnabled, 'cursor-not-allowed': !isDeleteEnabled, 'opacity-50': !isDeleteEnabled }"
                                @mouseenter="startHoverTimer(file.Key || '')"
                                @mouseleave="clearHoverTimer()"
                                @click="onDeleteFileClick(file.Key)">
                            </div>
                        </template>
                        停留3秒后可用
                    </n-tooltip>
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

:root {
    --n-tooltip-color: rgba(0, 0, 0, 0.85);
}

.n-tooltip {
    z-index: 1000;
}
</style>
