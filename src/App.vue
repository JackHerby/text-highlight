<script setup lang="ts">
import { computed } from 'vue'
import { onMounted, ref } from 'vue'

const stripHtmlAndNewlines = (str: string): string => {
  return str.replace(/<[^>]*>?/gm, '').replace(/\n/g, ' ')
}

interface IDocument {
  id: number
  text: string
}
const documents = ref<IDocument[]>([])
const fetchItems = async () => {
  try {
    const response = await fetch('/mock-data.json')
    if (!response.ok) {
      throw new Error('Error fetching data')
    }
    const data = await response.json()
    documents.value = data.map((doc: IDocument): IDocument => {
      return {
        id: doc.id,
        text: stripHtmlAndNewlines(doc.text),
      }
    })
  } catch (error) {
    console.error('Error fetching data', error)
  }
}
onMounted(() => fetchItems())

interface IHighlight {
  id: number | null
  startOffset: number
  endOffset: number
}
const highlights = ref<IHighlight[]>([])
onMounted(() => {
  const storedHighlights = localStorage.getItem('highlights')
  if (storedHighlights) {
    try {
      highlights.value = JSON.parse(storedHighlights)
    } catch (error) {
      console.error(error)
    }
  }
})

const selectedDocId = ref<number | null>(null)
const selectedTextStartOffset = ref(0)
const selectedTextEndOffset = ref(0)

const setSelectedText = (id: number) => {
  const selection = document.getSelection()
  console.log(selection)
  if (selection && selection.rangeCount > 0) {
    const range = selection.getRangeAt(0)
    if (range && range.startOffset !== range.endOffset) {
      selectedTextStartOffset.value = range.startOffset
      selectedTextEndOffset.value = range.endOffset
      selectedDocId.value = id
    }
  }
}
const saveHighlight = () => {
  highlights.value.push({
    id: selectedDocId.value,
    startOffset: selectedTextStartOffset.value,
    endOffset: selectedTextEndOffset.value,
  })
  localStorage.setItem('highlights', JSON.stringify(highlights.value))
  selectedTextStartOffset.value = 0
  selectedTextEndOffset.value = 0
  selectedDocId.value = null
}

const getHighlightedDocuments = computed(() => {
  return documents.value.map((doc) => {
    const docHighlights = highlights.value.filter((highlight) => highlight.id === doc.id)

    let docText = doc.text
    const range = document.createRange()
    const tempDiv = document.createElement('div')
    tempDiv.innerHTML = doc.text
    const textNode = tempDiv.firstChild
    for (const highlight of docHighlights) {
      if (textNode) {
        range.setStart(textNode, highlight.startOffset)
        range.setEnd(textNode, highlight.endOffset)
        const markElement = document.createElement('mark')
        markElement.className = 'highlight'
        range.surroundContents(markElement)
        docText = tempDiv.innerHTML
      }
    }

    return {
      ...doc,
      text: docText,
    }
  })
})

const unmarkAll = () => {
  highlights.value = []
  localStorage.removeItem('highlights')
  fetchItems()
}
</script>

<template>
  <header class="header">
    <button class="btn" @click="saveHighlight">Mark</button>
    <button class="btn" @click="unmarkAll">Unmark All</button>
  </header>

  <main>
    <p
      class="doc"
      v-for="doc in getHighlightedDocuments"
      :key="doc.id"
      v-html="doc.text"
      @mouseup="setSelectedText(doc.id)"
    ></p>
  </main>
</template>

<style scoped>
.header {
  position: sticky;
  top: 0;
  left: 0;
  width: 100%;
  display: flex;
  justify-content: center;
  padding: 1rem;
  gap: 1rem;
}

.btn {
  padding: 0.5rem 1.5rem;
  color: var(--vt-c-white);
  background-color: var(--vt-c-indigo);
  border: 1px solid var(--vt-c-white);
  border-radius: 0.5rem;
}

.doc {
  padding: 1rem 0;
}

.highlight {
  background-color: yellow;
}
</style>
