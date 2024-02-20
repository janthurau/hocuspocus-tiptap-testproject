<script setup lang="ts">
import {useEditor, EditorContent, findParentNode, Extension} from '@tiptap/vue-3';
import StarterKit from '@tiptap/starter-kit'
import {
  HocuspocusProvider,
  HocuspocusProviderWebsocket,
  TiptapCollabProvider,
  type TCollabThread
} from '@hocuspocus/provider'
import {onMounted, onUnmounted, ref} from "vue";
import {TiptapTransformer} from "@hocuspocus/transformer";
import * as Y from 'yjs'
import Collaboration from "@tiptap/extension-collaboration";
import CollaborationCursor from "@tiptap/extension-collaboration-cursor";
import CodeBlock from '@tiptap/extension-code-block'

// add new prop title which is a number
const props = defineProps<{ name: string, websocket: HocuspocusProviderWebsocket }>()

const lastSaved = ref<Date | null>()
const status = ref('')
const synced = ref('')
const currentVersion = ref(0)

window.Y = Y

const provider = new TiptapCollabProvider({
  name: `threads-${props.name}`,
  broadcast: false,
  websocketProvider: props.websocket,
// token: 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.iNVzTzLphhDcpjuIMnHz2ZZpmmlHxFES-FtEp-JnOrE',
  // token: 'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE3MDE3MTI4MzIsIm5iZiI6MTcwMTcxMjgzMiwiZXhwIjoxNzAxNzk5MjMyLCJpc3MiOiJodHRwczovL2NvbGxhYi50aXB0YXAuZGV2IiwiYXVkIjoieHk5ZGo5ZTYifQ.GLKATyfk6zMySW7D1ACdqc4eS6iBq0yrlugs4A1xmKg',
  onStateless(data) {
    const payload = JSON.parse(data.payload)

    switch (payload.event) {
      case 'saved':
        lastSaved.value = new Date()
        break;
      case 'version.created':
        currentVersion.value = payload.version
        break;
      case 'version.preview':
        const tempYdoc = new Y.Doc()
        Y.applyUpdate(tempYdoc, Uint8Array.from(atob(payload.ydoc), c => c.charCodeAt(0)))

        console.log(payload.version)
        console.log(TiptapTransformer.fromYdoc(tempYdoc, 'default'))

        break;
    }
  },
  onClose(data) {
    console.log(`CLOSE`, data.event.code, data.event.reason)

    if (data.event.reason === 'Document deleted') {
      console.log('disconnecting')
      // provider.destroy()
    }
  },

  onMessage(data) {
    // console.log(data)
  },
  onStatus(data) {
    // console.log('status', data)
    status.value = data.status
  },
  onSynced(data) {
    console.log('synced')
    // called when the document is synced
    // terminate the editor once this is called
    synced.value = data.state ? "yes" : "no"
  },

  onClose(data) {
    console.log(data)
  }
})

provider.on('unsyncedChanges', (e) => {
  // console.log(e)
})

type TAuditHistoryVersion = {
  name?: string;
  version: number;
  date: number;
}

const versions = ref<TAuditHistoryVersion[]>([])
const versioningStatus = ref(false)

provider.watchVersions(e => {
  versions.value = provider.document.getArray<TAuditHistoryVersion>('__tiptapcollab__versions').toArray()
})
//
provider.document.getMap<number>('__tiptapcollab__config').observe(e => {
  versioningStatus.value = !!provider.document.getMap<number>('__tiptapcollab__config').get('autoVersioning')
})

if (!window.providers) window.providers = {}
window.providers[props.name] = provider

onUnmounted(() => provider.destroy())

onMounted(() => {

  Collaboration.configure({
    document: provider.document,
  })


  CollaborationCursor.configure({
    provider: provider,
    user: {
      name: "Andrew Arkhipdov",
      color: "#ff66b3",
    },
  })
})
const previousNode = ref()
const aiLoading = ref(false)

const AiBotExtension = Extension.create({
  name: 'aiBotExtension',
  priority: 1000,

  onSelectionUpdate(args) {
    return

    const currentNodePosition = args.editor.state.selection.$anchor.pos

    if (currentNodePosition < 0) {
      return
    }

    const currentNode = args.editor.state.selection.$anchor.parent
    try {
      const currentNodeId = currentNode.attrs.uid

      const previousNodeId = previousNode.value?.attrs.uid

      if (currentNodeId
        && previousNodeId
        && previousNodeId !== currentNodeId
        && previousNode.value.textContent
      ) {
        console.log(`sending node ID for AI action: ${previousNodeId}`)

        aiLoading.value = true
        provider.sendStateless(JSON.stringify({
          action: 'ai',
          nodeId: previousNodeId,
        }))
      }

    } catch (e) {
      console.error(e)
    }

    previousNode.value = currentNode
  }

})

const editor = useEditor({
  extensions: [
    StarterKit.configure({
      history: false,
      codeBlock: false
    }),
    Collaboration.configure({
      document: provider.document,
    }),
    CodeBlock.configure({
      HTMLAttributes: {
        class: 'bg-gray b-2 b-red',
      },
    }),
    CollaborationCursor.configure({
      provider: provider,
      user: {
        name: "Andrew Arkhipov",
        color: "#ff66b3",
      },
    }),

    // UniqueId.configure({
    //   types: ['paragraph'],
    //   attributeName: 'uid',
    // }),
    // AiBotExtension.configure(),
  ],
  onCreate(args) {
    // applyDevTools(args.editor.view);
  },
  editorProps: {
    attributes: {
      class: 'border-2 border-black p-4 rounded-md h-96'
    },
  },
  autofocus: 'start',
})
window.editors ??= {}
window.editors[props.name] = editor

const previewVersion = (version: number) => {
  console.log(`fetching preview for ${version}`)

  provider.sendStateless(JSON.stringify({action: 'version.preview', version}))
}

const revertTo = (version: number) => {
  console.log(`reverting to ${version}`)

  // provider.revertToVersion(version)
  return provider.sendStateless(JSON.stringify({action: 'document.revert', version}))

}

const toggleVersioning = () => {
  console.log('updating')

  switch (provider.isAutoVersioning()) {
    case true:
      provider.disableAutoVersioning()
      break
    case false:
      provider.enableAutoVersioning()
      break
  }

}

const triggerVersioning = () => {
  console.log('triggering version')
  provider.createVersion()
}

provider.on('stateless', (payload) => {

  try {
    const jsonPayload = JSON.parse(payload.payload)

    if (jsonPayload.event === 'ai.done') {
      aiLoading.value = false
    }
  } catch (e) {
    console.error(e)
  }


})

const threads = ref<TCollabThread[]>()

provider.watchThreads((a, b) => {
  threads.value = provider.getThreads()
})

onMounted(() => {
  threads.value = provider.getThreads()
})

</script>

<template>
  <div>

    <p>Status: {{ status }}</p><br/>

    <div class="grid grid-cols-1 gap-2">
      <button @click.prevent="toggleVersioning" class="border bg-gray-200 p-1.5">Toggle Versioning (Current status:
        {{ versioningStatus }})
      </button>
      <button @click.prevent="triggerVersioning" class="border bg-gray-200 p-1.5">Manually create version</button>
      <button @click.prevent="provider.sendStateless(JSON.stringify({
          action: 'ai',
        }))" class="border bg-gray-200 p-1.5">SSE
      </button>


      <button @click.prevent="provider.createThread({data: {rrr: 'fff'}})" class="border bg-gray-200 p-1.5">Create
        Thread
      </button>
      <button @click.prevent="provider.updateThread(provider.getThreads()[0].id, {data: {k: new Date().toISOString()}})"
              class="border bg-gray-200 p-1.5">Update Thread
      </button>
      <button @click.prevent="provider.deleteThread(provider.getThreads()[0].id)" class="border bg-gray-200 p-1.5">
        Delete Threads
      </button>

      <button @click.prevent="provider.addComment(provider.getThreads()[0].id, { data: { commentText: 'JA MOIN'}})" class="border bg-gray-200 p-1.5">Add
        comment
      </button>
      <button @click.prevent="provider.updateComment(provider.getThreads()[0].id, provider.getThreads()[0].comments[0].id, {data: {update: (new Date()).toISOString()}})" class="border bg-gray-200 p-1.5">
        Update comment
      </button>
      <button @click.prevent="provider.deleteComment(provider.getThreads()[0].id, provider.getThreads()[0].comments[0].id)" class="border bg-gray-200 p-1.5">
        Delete comment
      </button>
    </div>


    <div>

      <div
        v-for="thread in threads" :key="thread.id">
        Thread: {{ thread.id }} ///
        {{ thread.updatedAt }} --- {{ JSON.stringify(thread.data) }}

        <div class="ml-5">

          <div
            v-for="comment in thread.comments" :key="comment.id">
            Comment: {{ comment.id }} // {{ comment.updatedAt }}
          </div>

        </div>


      </div>


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

    {{ props.name }}<br/>

    AI Loading: {{ aiLoading }}

    <editor-content :editor="editor"/>
    <!--      <textarea class="text-4xl bg-yellow-300 w-64 h-64 overflow-hidden p-2 shadow-lg"></textarea>-->
  </div>
</template>

<style>
.tiptap pre {
  background: #0D0D0D;
  color: #fff;
  font-family: JetBrainsMono, monospace;
  padding: 0.75rem 1rem;
  border-radius: 0.5rem;
}
</style>
