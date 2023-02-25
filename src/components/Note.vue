<script setup lang="ts">
import { useEditor, EditorContent } from '@tiptap/vue-3'
import StarterKit from '@tiptap/starter-kit'

const editor = useEditor({
  content: '<p>I’m running Tiptap with Vue.js. 🎉</p>',
  extensions: [
    StarterKit,
  ],
  editorProps: {
    attributes: {
      class: 'border-2 border-yellow-500 bg-yellow-300',
    },
  },
})

</script>

<template>
  <editor-content :editor="editor" />
</template>
