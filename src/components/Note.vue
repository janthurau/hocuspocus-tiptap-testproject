<script setup lang="ts">
import {useEditor, EditorContent} from '@tiptap/vue-3'
import StarterKit from '@tiptap/starter-kit'
import {Collaboration} from "@tiptap/extension-collaboration";
import Image from '@tiptap/extension-image'
import {HocuspocusProvider, HocuspocusProviderWebsocket} from '@hocuspocus/provider'
import {onUnmounted, ref} from "vue";
import {CollaborationCursor} from "@tiptap/extension-collaboration-cursor";
import {TiptapTransformer} from "@hocuspocus/transformer";

// add new prop title which is a number
const props = defineProps<{ name: string, websocket: HocuspocusProviderWebsocket }>()

const lastSaved = ref<Date | null>()
const status = ref('')
const synced = ref('')
const currentVersion = ref(0)

// create a method that returns a random number between variables min and max that are passed to the method

const provider = new HocuspocusProvider({
  name: `test-${props.name}`,
  websocketProvider: props.websocket,
  token: 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.EMfozF498NIwz9weJPF3d_AVSRYS22HE39T8E4qfLI8',
  onStateless(data) {
    const payload = JSON.parse(data.payload)

    switch (payload.event) {
      case 'saved':
        lastSaved.value = new Date()
        break;
      case 'version.created':
        currentVersion.value = payload.version
        break;
    }
  },
  onStatus(data) {
    console.log('status', data)
    status.value = data.status
  },
  onSynced(data) {
    // called when the document is synced
    // terminate the editor once this is called
    synced.value = data.state ? "yes" : "no"
  },
})

type TAuditHistoryVersion = {
  name?: string;
  version: number;
  date: number;
}


const versions = ref<TAuditHistoryVersion[]>([])
const versioningStatus = ref(false)

provider.document.getArray<TAuditHistoryVersion>('__tiptapcollab__versions').observe(e => {
  versions.value = provider.document.getArray<TAuditHistoryVersion>('__tiptapcollab__versions').toArray()
})

provider.document.getMap<number>('__tiptapcollab__config').observe(e => {
  versioningStatus.value = !!provider.document.getMap<number>('__tiptapcollab__config').get('autoVersioning')
})

if (!window.providers) window.providers = {}
window.providers[props.name] = provider

onUnmounted(() => provider.destroy())

const editor = useEditor({
  extensions: [
    StarterKit.configure({
      history: false
    }),
    Collaboration.configure({
      document: provider.document,
    }),
    Image.configure({
      inline: true,
    })    // CollaborationCursor.configure({
    //   provider: provider,
    //   user: {
    //     name: 'Cyndi Lauper',
    //     color: '#f783ac',
    //   },
    // }),
  ],
  editorProps: {
    attributes: {
      class: 'text-4xl bg-yellow-300 w-64 h-64 overflow-hidden p-2 shadow-lg'
    },
  },
})

const previewVersion = (version: number) => {
  console.log(`fetching preview for ${version}`)
}

const revertTo = (version: number) => {
  console.log(`reverting to ${version}`)

  provider.sendStateless(JSON.stringify({action: 'revertTo', version}))
}

const toggleVersioning = () => {
  console.log('updating')
  provider.document.getMap<number>('__tiptapcollab__config').set('autoVersioning', versioningStatus.value ? 0 : 1)
}

const triggerVersioning = () => {
  console.log('triggering version')
  provider.sendStateless(JSON.stringify({action: 'version.create', name: 'my custom name'}))
}


</script>

<template>
  <div>

    <p>Status: {{ status }}</p><br/>

    <div class="grid grid-cols-1 gap-2">
      <button @click.prevent="toggleVersioning" class="border bg-gray-200 p-1.5">Toggle Versioning (Current status:
        {{ versioningStatus }})
      </button>
      <button @click.prevent="triggerVersioning" class="border bg-gray-200 p-1.5">Manually create version</button>
    </div>

    <p class="mt-2">Last saved: <span v-if="versions.length > 0">{{
        (new Date(versions[versions.length - 1].date)).toISOString()
      }}</span></p>
    <p>Earliest version date: <span v-if="versions.length > 0">{{
        (new Date(versions[0].date)).toISOString()
      }}</span></p>
    <p>Latest version #: {{ versions.length }}</p>

    <div class="my-4">

      <h2>Versions:</h2>
      <div class="grid grid-cols-4 gap-2">

        <template v-for="(version, key) in versions" :key="key">

        <span>
          {{ version.version }}
        </span>

        <span>
          [{{ (new Date(version.date)).toISOString() }}]
        </span>

        <span>
          {{ version.name }}
        </span>

        <span>
          <a @click.prevent="previewVersion(version.version)" href="" class="cursor-pointer font-bold">preview</a>
          or
          <a @click.prevent="revertTo(version.version)" href="" class="cursor-pointer font-bold">revertTo</a>
        </span>
        </template>
      </div>
    </div>

    <button
      v-if="false"
      class="border-2 bg-red-200"
      @click="provider.sendStateless(JSON.stringify({action: 'undo'}))">Undo
    </button>

    <button
      v-if="false"
      class="border-2 bg-red-200 ml-2"
      @click="provider.sendStateless(JSON.stringify({action: 'redo'}))">Redo
    </button>

    <button
      v-if="false"
      class="border-2 bg-red-200 ml-2"
      @click="provider.sendStateless(JSON.stringify({action: 'printFinalStack'}))">printFinalStack
    </button>

    <button
      v-if="false"
      class="border-2 bg-red-200 ml-2"
      @click="provider.sendStateless(JSON.stringify({action: 'printLastUpdate'}))">printLastUpdate
    </button>

    <editor-content :editor="editor" style="font-family: 'Reenie Beanie'"/>
    <!--      <textarea class="text-4xl bg-yellow-300 w-64 h-64 overflow-hidden p-2 shadow-lg"></textarea>-->
  </div>
</template>
