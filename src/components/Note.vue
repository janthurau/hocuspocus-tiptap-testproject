<script setup lang="ts">
import { useEditor, EditorContent } from '@tiptap/vue-3'
import StarterKit from '@tiptap/starter-kit'
import {Collaboration} from "@tiptap/extension-collaboration";
import {HocuspocusProvider, HocuspocusProviderWebsocket} from '@hocuspocus/provider'
import {onUnmounted, ref} from "vue";
import {CollaborationCursor} from "@tiptap/extension-collaboration-cursor";

const props = defineProps<{name: string, websocket: HocuspocusProviderWebsocket}>()

const lastSaved = ref<Date | null>()
const status = ref('')
const synced = ref('')

const provider = new HocuspocusProvider({
  name: props.name,
  websocketProvider: props.websocket,
  onStateless({payload}){
    switch(payload){
      case 'saved':
        lastSaved.value = new Date()
        break;
    }
  },
  onStatus(data) {
    status.value = data.status
  },
  onSynced(data) {
    synced.value = data.state ? "yes" : "no"
  },
})

onUnmounted(() => provider.destroy())

const editor = useEditor({
  extensions: [
    StarterKit.configure({
      history: false
    }),
    Collaboration.configure({
      document: provider.document
    }),
    CollaborationCursor.configure({
      provider: provider,
      user: {
        name: 'Cyndi Lauper',
        color: '#f783ac',
      },
    }),
  ],
  editorProps: {
    attributes: {
      class: 'text-4xl bg-yellow-300 w-64 h-64 overflow-hidden p-2 shadow-lg',
    },
  },
})

</script>

<template>
  <div>

    <p>Last saved: {{lastSaved?.toISOString()}}</p>
    <p>Status: {{status}}</p>
    <p>Synced: {{synced}}</p>

    <button
      class="border-2 bg-red-200"
      @click="provider.startSync()">Start sync</button>

    <editor-content :editor="editor" style="font-family: 'Reenie Beanie'" />
  </div>
</template>
