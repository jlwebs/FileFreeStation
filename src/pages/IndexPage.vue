<script setup lang="ts">
import { ref, onMounted, type Ref } from 'vue';
import { useI18n } from "vue-i18n";
import useI18nStore from "../store/i18n";
import { useRouter } from "vue-router";

const router = useRouter();

const i18nStore = useI18nStore();
let updateLocale = (locale: string) => {
  i18nStore.setLocale(locale);
};

const { locale } = useI18n();
if (i18nStore.locale !== "") {
  locale.value = i18nStore.locale;
}

let onClipAreaClick = () => {
  router.push({ path: '/clip', query: { mode: 'clip' } });
};

let onUploadClick = () => {
  router.push("/file");
};

const tooltipFile: Ref<HTMLElement | null> = ref(null);
const tooltipClip: Ref<HTMLElement | null> = ref(null);

onMounted(() => {
  setTimeout(() => {
    if (tooltipFile.value) {
      tooltipFile.value.classList.add('fade-out');
    }
    if (tooltipClip.value) {
      tooltipClip.value.classList.add('fade-out');
    }
  }, 3000);
});
</script>

<template>
  <div class="flex flex-col items-center">
    <div class="pannel file-pannel">
      <div class="text-2xl flex flex-row items-center">
        <router-link to="/filemanage" class="link-hint">{{ $t("index.file_channel_title") +"👈"}}</router-link>
      </div>
      <div class="upload-area" @click="onUploadClick"></div>
      <div class="tooltip" ref="tooltipFile">点击开始上传{{ $t("index.file_channel_title") }}</div>
    </div>
    <div class="pannel clip-pannel">
      <div class="text-2xl flex flex-row items-center">
        <router-link :to="{ path: '/filemanage', query: { mode: 'clip' }}" class="link-hint">{{ $t("index.clip_channel_title")+"👈" }}</router-link>
        <button 
          class="local-btn ml-auto px-4 py-1 rounded text-sm text-white bg-blue-500 hover:bg-blue-600"
          @click="$router.push('/clip?local=true')"
        >
          上次改过的文本
        </button>
      </div>
      <div class="clip-area" @click="onClipAreaClick"></div>
      <div class="tooltip" ref="tooltipClip">点击开始上传{{ $t("index.clip_channel_title") }}</div>
    </div>
    <!-- <div class="pannel tips-pannel">{{ $t("index.tips_content") }}</div> -->
    <select v-model="$i18n.locale" class="locale-changer" @change="updateLocale($i18n.locale)">
      <option v-for="locale in $i18n.availableLocales" :key="`locale-${locale}`" :value="locale">{{ locale }}</option>
    </select>
  </div>
</template>

<style scoped>
.pannel {
  --uno: my-6 px-4 py-4 max-w-screen-md w-4/5 rounded shadow-md;
  position: relative;
  margin-right: 15px;
}

.file-pannel {
  background-color: #bee5eb;
}

.clip-pannel {
  background-color: #F27E93;
}

.tips-pannel {
  background-color: #d1e7dd;
}

.upload-area {
  --uno: border-dashed border-1 border-gray-400 h-30 mt-2 cursor-pointer;
  background: url(../assets/upload.svg) center center no-repeat;
  background-color: #f8f9fa;
}

.clip-area {
  display: block;
  width: 100%;
  --uno: border-dashed border-1 border-gray-400 h-30 mt-2 cursor-pointer;
  background: url(../assets/clipboard.svg) center center no-repeat;
  background-color: #f8f9fa;
}

.link-hint {
  text-decoration: underline;
  text-decoration-style: dashed;
  text-decoration-color: #a0aec0;
  text-underline-offset: 0.2em;
}

.link-hint:hover {
  text-decoration-color: black;
}

.locale-changer {
  border: grey 1px solid;
  border-radius: 6px;
  padding: 2px 4px;
}

.tooltip {
  width:50px;
  position: absolute;
  top: 50%;
  right: 13%;
  transform: translateY(-50%);
  background-color: rgba(0, 123, 255, 0.8);
  color: white;
  padding: 5px 10px;
  border-radius: 5px;
  margin-right: -160px; /* Adjust as needed */
  z-index: 10;
  opacity: 1;
  transition: opacity 3s ease-in-out;
  white-space: normal;
  word-break: break-word;
}

.tooltip.fade-out {
  opacity: 0;
}

.tooltip::after {
  content: "";
  position: absolute;
  top: 50%;
  left: -10px;
  margin-top: -5px;
  border-width: 5px;
  border-style: solid;
  border-color: transparent #007bff transparent transparent;
}
</style>

<style>
html,
body,
#app {
  margin: 0;
  padding: 0;
  background-color: #f8f9fa;
}
</style>
