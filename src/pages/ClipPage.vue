import { nextTick } from 'vue';

<script setup lang="ts">
import { nextTick, onMounted, onBeforeUnmount, ref, watch, watchEffect } from "vue";
import { minimalSetup } from "codemirror"
import { EditorState } from "@codemirror/state"
import { EditorView, lineNumbers, highlightSpecialChars, drawSelection, dropCursor } from "@codemirror/view"
import { useRoute } from "vue-router";
import useClipStore from "../store/clip";
import { PutFile } from "../api";
import { getRandomFilename } from "../utils/utils";

const route = useRoute();
const code = ref("");
const modified = ref(false);
const editorElement = ref();
const lastModified = ref("");
let editor: EditorView;

const isLoading = ref(false);
const loadError = ref('');
const isFirstLoad = ref(true);
const isLoadedSuccessfully = ref(false);

// 获取文件内容
const fetchFileContent = async (filename: string) => {
  try {
    isLoading.value = true;
    loadError.value = '';
    const content = await fetch(`/${filename}`).then(res => res.text());
    const timestamp = new Date().toISOString();
    lastModified.value = new Date(timestamp).toLocaleString();
    isLoadedSuccessfully.value = true;
    return { content, timestamp };
  } catch (error) {
    console.error('Failed to fetch file:', error);
    loadError.value = '文件加载失败';
    throw error;
  } finally {
    isLoading.value = false;
  }
};

// 保存到localStorage
const saveToLocalStorage = (filename: string, content: string) => {
  const timestamp = new Date().toISOString();
  localStorage.setItem('latestClip', JSON.stringify({ filename, content, timestamp }));
  lastModified.value = new Date(timestamp).toLocaleString();
};

let startState = EditorState.create({
  doc: "",
  extensions: [
    minimalSetup,
    lineNumbers(),
    highlightSpecialChars(),
    drawSelection(),
    dropCursor(),
    EditorView.updateListener.of((update) => {
      code.value = update.state.doc.toString();
      if (update.docChanged) {
        modified.value = true;
        if (!isFirstLoad.value && autoSaveEnabled.value && code.value !== lastSavedCode.value) {
          onAutoSaveBtnClick();
          lastSavedCode.value = code.value;
        }
        isFirstLoad.value = false;
      }
    }),
  ]
});

onMounted(async () => {
  editor = new EditorView({
    state: startState,
    parent: editorElement.value,
  });
  
  // 处理open参数
  if (route.query.open) {
    isLoadedSuccessfully.value = false;
    const filenameToOpen = route.query.open as string;
    filename.value = filenameToOpen;
    try {
      const { content } = await fetchFileContent(filenameToOpen);
      editor.dispatch({
        changes: { from: 0, to: editor.state.doc.length, insert: content }
      });
      isLoadedSuccessfully.value = true;
    } catch {
      saveFilter();
      editor.dispatch({
        changes: { from: 0, to: editor.state.doc.length, insert: '获取剪贴板内容失败，请刷新重试' }
      });
      lastModified.value = '';
    }
  }
  // 如果有local参数，尝试从localStorage加载
  else if (route.query.local) {
    isLoadedSuccessfully.value = false;
    const latestClip = localStorage.getItem('latestClip');
    if (latestClip) {
      try {
      const { filename: f } = JSON.parse(latestClip);
      const { content } = await fetchFileContent(f);
      filename.value = f;
      editor.dispatch({
        changes: { from: 0, to: editor.state.doc.length, insert: content }
      });
      lastModified.value = new Date().toLocaleString();
      isLoadedSuccessfully.value = true;
    }catch{
      saveFilter();
      editor.dispatch({
        changes: { from: 0, to: editor.state.doc.length, insert: '获取剪贴板内容失败，请刷新重试' }
      });
      lastModified.value = '';
    }
    }
  }else{
    // mode=open 新增剪贴板模式
    isLoadedSuccessfully.value = true;
  }
  
  editor.requestMeasure({
    read: () => {
      editor.focus();
    }
  });
});

let filename = ref(getRandomFilename());

let refreshRandomFileName = () => {
  filename.value = getRandomFilename();
};

const clipStore = useClipStore();
const saveStatusText = ref('');
const autoSaveEnabled = ref(true);
let lastSavedCode = ref('');
let saveFilter = () =>{
  if (!isLoadedSuccessfully.value) {
      saveStatusText.value = '加载剪贴板失败，请刷新网页';
      setTimeout(() => {
        nextTick(() => {
          saveStatusText.value = '';
        });
      }, 3000);
    return true;
  }
  return false
}
let onSaveBtnClick = async () => {
  if(saveFilter()) return;
  saveStatusText.value = '';
  try {
    await PutFile(filename.value, code.value, clipStore.visibility, "text");
    modified.value = false;
    saveToLocalStorage(filename.value, code.value);
    if (route.query.open) {
      const timestamp = new Date().toISOString();
      localStorage.setItem('openClip', JSON.stringify({
        filename: filename.value,
        content: code.value,
        timestamp
      }));
    }
    saveStatusText.value = '手动保存成功';
  } catch {
    saveStatusText.value = '保存剪贴板内容失败，请重试' 
  }

  
  setTimeout(() => {
    nextTick(() => {
      saveStatusText.value = '';
    });
  }, 3000);
};

let onAutoSaveBtnClick = async () => {
  if(saveFilter()) return;
  try {
    saveStatusText.value = '';
    await PutFile(filename.value, code.value, clipStore.visibility, "text");
    modified.value = false;
    saveToLocalStorage(filename.value, code.value);
    if (route.query.open) {
      const timestamp = new Date().toISOString();
      localStorage.setItem('openClip', JSON.stringify({
        filename: filename.value,
        content: code.value,
        timestamp
      }));
    }
    saveStatusText.value = '自动保存成功';
  } catch {
    saveStatusText.value = '自动保存剪贴板内容失败...';
    setTimeout(() => {
      saveStatusText.value = '';
      console.log(saveStatusText)
    }, 3000);
  }
};

let saveContentKeydown = (e: KeyboardEvent) => {
  if ((e.ctrlKey && e.key === "s") || (e.metaKey && e.key === "s")) {
    e.preventDefault();
    onSaveBtnClick();
  }
};

let onPasteFile = async (e: ClipboardEvent) => {
  if (!e.clipboardData?.files.length) {
    return;
  }
  const file = e.clipboardData.files[0];
  const text = await file.text();
  const cursor = editor.state.selection.main.head;
  editor.dispatch({
    changes: { from: cursor, insert: text },
  });
};

onMounted(() => {
  window.addEventListener("keydown", saveContentKeydown);
  document.addEventListener("paste", onPasteFile);
});

onBeforeUnmount(() => {
  window.removeEventListener("keydown", saveContentKeydown);
  document.removeEventListener("paste", onPasteFile);
});
</script>

<template>
  <div class="flex flex-col items-center">
    <div class="text-area flex flex-col mt-4">
      <div class="header p-2 flex flex-row items-center">
        <input class="filename-input monospace" type="text" v-model="filename" :placeholder="$t('common.filename')" />
        <button @click="refreshRandomFileName" class="i-mdi-refresh ml-1 w-5 h-5"></button>
        <div :class="modified ? 'unsave-attention' : 'save-attention'"></div>
        <button 
          class="local-btn ml-2 px-4 py-1 rounded text-sm text-white bg-blue-500 hover:bg-blue-600"
          @click="$router.push('/clip?local=true'); $router.go(0)"
        >
          上次改过的文本
        </button>
      </div>
      <div ref="editorElement" :class="{'opacity-50': isLoading, 'pointer-events-none': isLoading}"></div>
      <div v-if="loadError" class="text-red-500 text-sm px-2">{{ loadError }}</div>
      <div class="footer p-2 flex items-center">
        <select class="public-select" v-model="clipStore.visibility">
          <option value="private">{{ $t('common.private') }}</option>
          <option value="public">{{ $t('common.public') }}</option>
        </select>
        <div class="text-sm text-gray-500 ml-4" v-if="route.query.local">
          本次加载项为本地记载的最近更新过的剪贴板
        </div>
        <div class="text-sm text-gray-500 ml-4" v-if="lastModified">
          最后修改: {{ lastModified }}
        </div>

        <div style="margin-left: auto;"></div>
        <div class="save-status" :class="{'error': saveStatusText.includes('失败')}" v-if="saveStatusText">{{ saveStatusText }}</div>
        <div class="flex items-center ml-4">
          <input type="checkbox" id="auto-save" v-model="autoSaveEnabled" class="mr-2" />
          <label for="auto-save" class="text-sm">变动时自动保存</label>
        </div>
        <button class="save-btn ml-2" @click="onSaveBtnClick">{{ $t('common.save') }}</button>
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

.pannel {
  --uno: my-6 px-4 py-4 max-w-screen-md w-4/5 rounded shadow-md;
}

.tips-pannel {
  background-color: #d1e7dd;
}

.text-area {
  --uno: rounded max-w-screen-md w-4/5 border-1 border-gray-300;
  background-color: white;
}

.text-area .header {
  background-color: #f5f5f5;
}

.text-area .footer {
  --uno: flex flex-row items-center;
  background-color: #f5f5f5;
  padding: 0.5rem;
}

.text-area .footer .public-select {
  --uno: border-1 rounded px-6 py-1.5 text-sm;
  border-color: #d1d1d1;
  outline-color: #0969da;
}

.text-area .footer .save-status {
  --uno: rounded px-6 py-1.5 text-sm ml-2 text-white;
  background-color: #1f883d;
  animation: fadeOut 3s ease-in-out forwards;
  margin-left: auto;
  transition: background-color 0.3s;
}

.text-area .footer .save-status.error {
  background-color: #dc3545;
}

@keyframes fadeOut {
  0% { opacity: 1; }
  100% { opacity: 0; }
}

.text-area .footer .save-btn {
  --uno: rounded px-2 py-0.5 text-sm text-white;
  background-color: #1f883d;
  min-width: auto;
  white-space: nowrap;
}

.text-area .footer .save-btn:hover {
  background-color: #1a7f37;
}

.text-area .header .filename-input {
  --uno: border-1 rounded px-3 py-2 text-sm w-60;
  border-color: #d1d1d1;
  outline-color: #0969da;
}

.cm-editor {
  height: 400px;
  border-top: 1px solid #ddd;
  border-bottom: 1px solid #ddd;
}

.cm-editor.cm-focused {
  outline: none;
}

.cm-gutter.cm-lineNumbers {
  background-color: white;
}

.cm-gutters {
  border: none !important;
}

.cm-selectionBackground {
  background-color: #54aeff66 !important;
}

.unsave-attention {
  --uno: i-mdi-circle-small w-8 h-8 ml-auto;
  color: #9a6700 !important;
}

.save-attention {
  --uno: i-mdi-circle-small w-8 h-8 ml-auto;
  color: #1f883d !important;
}
</style>
