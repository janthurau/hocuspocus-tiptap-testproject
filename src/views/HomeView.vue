<script setup lang="ts">
import Note from "@/components/Note.vue";
import {HocuspocusProviderWebsocket, TiptapCollabProviderWebsocket} from '@hocuspocus/provider'
import {onMounted, ref} from "vue";

const status = ref('')
const websocket = new TiptapCollabProviderWebsocket({
  appId: 'xy9dj9e6',
  onStatus(data) {
    status.value = data.status
  }
})

type TNoteConfig = {
  name: string
}

const notes = ref<TNoteConfig[]>([
])

const add = () => {
  notes.value.push({name: `test${notes.value.length}`})
  localStorage.setItem('noteCount', notes.value.length.toString())
}

const remove = () => {
  notes.value.pop()
  localStorage.setItem('noteCount', notes.value.length.toString())
}

const reset = () => {
  localStorage.setItem('noteCount', "0")
  window.location.reload()
}

onMounted(() => {
  const notesToAdd = parseInt(localStorage.getItem('noteCount') ?? '1')

  for( let i = 0; i<notesToAdd ; i++){
    notes.value.push({name: `test${i}`})
  }
})

</script>

<template>
  <main>
    <div class="mb-10">
      <button
        class="border-2 bg-red-200"
        @click="add">Add new</button>

      <button
        class="border-2 bg-red-200"
        @click="remove">Remove last</button>

      <button
        class="border-2 bg-red-200"
        @click="reset">Reset</button>

      <p>{{status}}</p>
    </div>

    <div class="grid grid-cols-1 gap-3">
      <Note
        v-for="(note, key) in notes"
        class="my-5" :name="note.name" :key="key" :websocket="websocket"/>
    </div>
  </main>
</template>
