<script setup>
import { ref, reactive, computed, onMounted, watch } from 'vue'

const MARKERS = [
  { value: '', label: 'без маркера' },
  { value: 'minus', label: 'минусующее' },
  { value: 'filter', label: 'фильтрующее' },
  { value: 'other', label: 'другое' },
]

const inputText = ref('')
const phrases = ref([])
const words = ref([])
const filteredPhraseIds = ref(null)
const selectedWord = ref(null)

const wordMarkers = reactive({})
const sortConfig = reactive({
  phrases: { field: null, asc: true },
  words: { field: null, asc: true },
})

const isDark = ref(false)
const uiStyle = ref('modern')
const customPrompt = ref('')
const isLoading = ref(false)

watch([customPrompt], () => {
  saveData()
})

const BASE_PROMPT = `Ты анализируешь список поисковых фраз для рекламной кампании.

Твоя задача — присвоить каждой фразе один из маркеров:

* "без маркера" — фраза релевантна и должна остаться
* "минусующее" — фраза явно нерелевантна, её нужно исключить
* "фильтрующее" — фраза потенциально нерелевантна или сомнительна
* "другое" — невозможно однозначно определить

---

# Контекст задачи

{context}

---

# Правила классификации

## 1. Минусующее

Относи сюда только если:

* фраза явно не связана с продуктом/услугой
* другой смысл (другая категория товара)
* информационный запрос, не имеющий коммерческой ценности
* бесплатные/скачать/своими руками, если это не цель бизнеса

👉 будь строгим: только очевидные случаи

---

## 2. Фильтрующее

* возможная нерелевантность
* неочевидное намерение
* может подходить, но есть риск нецелевого трафика

---

## 3. Другое

* недостаточно контекста
* неоднозначные формулировки
* может относиться к разным сценариям

---

## 4. Без маркера

* релевантно
* соответствует цели бизнеса
* потенциально приводит клиента

---

# Важные ограничения

* НЕ переусердствуй с минус-словами
* если сомневаешься → "фильтрующее" или "другое"
* по умолчанию оставляй "без маркера"

---

# Формат ответа

Верни результат в виде:

фраза | маркер

Без дополнительных комментариев.`

onMounted(() => {
  const savedTheme = localStorage.getItem('theme')
  if (savedTheme) {
    isDark.value = savedTheme === 'dark'
  } else {
    isDark.value = window.matchMedia('(prefers-color-scheme: dark)').matches
  }
  const savedStyle = localStorage.getItem('uiStyle')
  if (savedStyle) {
    uiStyle.value = savedStyle
  }
  applyTheme()
  loadSavedData()
})

function loadSavedData() {
  const savedPhrases = localStorage.getItem('phrases')
  const savedMarkers = localStorage.getItem('wordMarkers')
  if (savedPhrases) {
    phrases.value = JSON.parse(savedPhrases)
    updateWordsFromPhrases()
  }
  if (savedMarkers) {
    const markers = JSON.parse(savedMarkers)
    Object.assign(wordMarkers, markers)
  }
  const savedCustomPrompt = localStorage.getItem('customPrompt')
  if (savedCustomPrompt) {
    customPrompt.value = savedCustomPrompt
  } else if (!customPrompt.value) {
    customPrompt.value = `Бренд: 
Ниша: 
Продукт / услуга: 
Цель рекламы: 
Регион: 
Описание ЦА: 
Дополнительная информация:`
  }
}

function saveData() {
  localStorage.setItem('phrases', JSON.stringify(phrases.value))
  localStorage.setItem('wordMarkers', JSON.stringify(wordMarkers))
  localStorage.setItem('customPrompt', customPrompt.value)
}

function applyTheme() {
  document.documentElement.setAttribute('data-theme', isDark.value ? 'dark' : 'light')
  document.documentElement.setAttribute('data-ui-style', uiStyle.value)
  localStorage.setItem('theme', isDark.value ? 'dark' : 'light')
  localStorage.setItem('uiStyle', uiStyle.value)
}

function toggleTheme() {
  isDark.value = !isDark.value
  applyTheme()
}

function toggleUiStyle() {
  uiStyle.value = uiStyle.value === 'modern' ? 'retro' : 'modern'
  applyTheme()
}

function clearProject() {
  phrases.value = []
  words.value = []
  filteredPhraseIds.value = null
  selectedWord.value = null
  selectedWords.value = []
  for (const key in wordMarkers) {
    delete wordMarkers[key]
  }
  localStorage.removeItem('phrases')
  localStorage.removeItem('wordMarkers')
  localStorage.removeItem('customPrompt')
}

const selectedWords = ref([])
const markerFilter = ref('')
const lastSelectedIndex = ref(-1)

function toggleWordSelection(word, event) {
  const wordsList = sortedWords.value.map(w => w.word)
  const idx = wordsList.indexOf(word)
  
  if (event.ctrlKey || event.metaKey) {
    const selIdx = selectedWords.value.indexOf(word)
    if (selIdx > -1) {
      selectedWords.value.splice(selIdx, 1)
    } else {
      selectedWords.value.push(word)
    }
    lastSelectedIndex.value = idx
  } else if (event.shiftKey && lastSelectedIndex.value >= 0) {
    selectedWords.value = []
    const start = Math.min(lastSelectedIndex.value, idx)
    const end = Math.max(lastSelectedIndex.value, idx)
    const range = wordsList.slice(start, end + 1)
    for (const w of range) {
      selectedWords.value.push(w)
    }
  } else {
    selectedWords.value = [word]
    lastSelectedIndex.value = idx
  }
  applyFilterBySelectedWords()
}

function applyFilterBySelectedWords() {
  if (!selectedWords.value.length) {
    filteredPhraseIds.value = null
    return
  }
  const phraseIds = new Set()
  for (const phrase of phrases.value) {
    const phraseLower = phrase.text.toLowerCase()
    for (const word of selectedWords.value) {
      if (phraseLower.includes(word)) {
        phraseIds.add(phrase.id)
        break
      }
    }
  }
  filteredPhraseIds.value = phraseIds
}

function copySelectedWords() {
  if (!selectedWords.value.length) return
  const text = selectedWords.value.join('\n')
  navigator.clipboard.writeText(text)
}

function applyMarkerFilter() {
  selectedWords.value = []
  applyFilterBySelectedWords()
}

function setMarkerForSelected(marker) {
  for (const word of selectedWords.value) {
    wordMarkers[word] = marker
  }
  updateWordsFromPhrases()
  saveData()
  selectedWords.value = []
  applyFilterBySelectedWords()
}

async function processWithAI() {
  const apiKey = import.meta.env.VITE_OPENROUTER_API_KEY
  const model = import.meta.env.VITE_OPENROUTER_MODEL || 'glm/glm-4-air-free'
  
  if (!apiKey || apiKey === 'your_api_key_here') {
    alert('Настройте API ключ в .env файле')
    return
  }
  
  if (!phrases.value.length) {
    alert('Нет фраз для обработки')
    return
  }
  
  isLoading.value = true
  
  const context = customPrompt.value || 'Контекст не указан'
  let prompt = BASE_PROMPT.replace('{context}', context)
  prompt += '\n\n# Фразы для анализа\n\n'
  prompt += phrases.value.map(p => p.text).join('\n')
  
  try {
    const controller = new AbortController()
    const timeoutId = setTimeout(() => controller.abort(), 60000)
    
    const response = await fetch('https://openrouter.ai/api/v1/chat/completions', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${apiKey}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        model: model,
        messages: [{ role: 'user', content: prompt }],
        max_tokens: 2000
      }),
      signal: controller.signal
    })
    
    clearTimeout(timeoutId)
    
    if (!response.ok) {
      const error = await response.json()
      throw new Error(error.error?.message || 'API error')
    }
    
    const data = await response.json()
    const content = data.choices?.[0]?.message?.content || ''
    
    const lines = content.split('\n')
    for (const line of lines) {
      const match = line.match(/^(.+?)\s*\|\s*(.+)$/)
      if (match) {
        const phraseText = match[1].trim()
        const markerText = match[2].trim().toLowerCase()
        
        let marker = ''
        if (markerText.includes('минус')) marker = 'minus'
        else if (markerText.includes('фильтр')) marker = 'filter'
        else if (markerText.includes('другое')) marker = 'other'
        else if (markerText.includes('без маркера') || !markerText) marker = ''
        
        if (marker) {
          const wordsArray = phraseText.toLowerCase().split(/\s+/)
          for (const word of wordsArray) {
            const cleaned = word.replace(/[^а-яёa-z0-9]/g, '')
            if (cleaned) {
              wordMarkers[cleaned] = marker
            }
          }
        }
      }
    }
    
    updateWordsFromPhrases()
    saveData()
    clearFilter()
  } catch (error) {
    console.error(error)
    alert('Ошибка: ' + error.message)
  } finally {
    isLoading.value = false
  }
}

function updateWordsFromPhrases() {
  const wordMap = new Map()
  for (const phrase of phrases.value) {
    const wordsArray = phrase.text.toLowerCase().split(/\s+/)
    for (const word of wordsArray) {
      const cleaned = word.replace(/[^а-яёa-z0-9]/g, '')
      if (cleaned) {
        wordMap.set(cleaned, (wordMap.get(cleaned) || 0) + 1)
      }
    }
  }
  words.value = Array.from(wordMap.entries())
    .map(([word, freq]) => ({ 
      word, 
      freq, 
      marker: wordMarkers[word] || '' 
    }))
    .sort((a, b) => b.freq - a.freq)
}

function splitPhrases() {
  const lines = inputText.value.split('\n').filter(l => l.trim())
  const newId = phrases.value.length > 0 
    ? Math.max(...phrases.value.map(p => p.id)) + 1 
    : 0
  const phraseList = lines.map((text, i) => ({ 
    id: newId + i, 
    text: text.trim(),
    wordCount: text.trim().split(/\s+/).length
  }))
  phrases.value = [...phrases.value, ...phraseList]
  updateWordsFromPhrases()
  
  filteredPhraseIds.value = null
  selectedWord.value = null
  saveData()
  inputText.value = ''
}

function sortPhrases(field) {
  if (sortConfig.phrases.field === field) {
    sortConfig.phrases.asc = !sortConfig.phrases.asc
  } else {
    sortConfig.phrases.field = field
    sortConfig.phrases.asc = true
  }
}

function sortWords(field) {
  if (sortConfig.words.field === field) {
    sortConfig.words.asc = !sortConfig.words.asc
  } else {
    sortConfig.words.field = field
    sortConfig.words.asc = true
  }
}

const sortedPhrases = computed(() => {
  if (!phrases.value.length) return []
  let result = phrases.value
  if (filteredPhraseIds.value !== null) {
    result = result.filter(p => filteredPhraseIds.value.has(p.id))
  }
  const { field, asc } = sortConfig.phrases
  if (!field) return result
  return [...result].sort((a, b) => {
    const av = a[field]
    const bv = b[field]
    return asc ? (av > bv ? 1 : -1) : (av < bv ? 1 : -1)
  })
})

const sortedWords = computed(() => {
  if (!words.value.length) return []
  const { field, asc } = sortConfig.words
  let result = words.value.map(w => ({ 
    ...w, 
    marker: wordMarkers[w.word] || '',
    markerOrder: getMarkerOrder(wordMarkers[w.word] || '')
  }))
  
  if (markerFilter.value) {
    result = result.filter(w => w.marker === markerFilter.value)
  }
  
  if (!field) return result
  return [...result].sort((a, b) => {
    let av = a[field]
    let bv = b[field]
    if (field === 'marker') {
      av = a.markerOrder
      bv = b.markerOrder
    }
    return asc ? (av > bv ? 1 : -1) : (av < bv ? 1 : -1)
  })
})

function getMarkerOrder(marker) {
  const idx = MARKERS.findIndex(m => m.value === marker)
  return idx === -1 ? MARKERS.length : idx
}

function filterByWord(word) {
  selectedWords.value = []
  selectedWord.value = word
  const phraseIds = new Set()
  for (const phrase of phrases.value) {
    if (phrase.text.toLowerCase().includes(word)) {
      phraseIds.add(phrase.id)
    }
  }
  filteredPhraseIds.value = phraseIds
}

function clearFilter() {
  filteredPhraseIds.value = null
  selectedWord.value = null
  selectedWords.value = []
}

function setMarker(word, marker) {
  wordMarkers[word] = marker
  const idx = words.value.findIndex(w => w.word === word)
  if (idx !== -1) {
    words.value[idx].marker = marker
  }
}
</script>

<template>
  <div class="app">
    <header class="header">
      <div class="logo">
        <span class="logo-icon">📊</span>
        <div class="logo-text">
          <h1>WordParser</h1>
          <span class="logo-subtitle">Сервис для анализа ключевых фраз</span>
        </div>
      </div>
      <div class="header-buttons">
        <button class="style-toggle" @click="toggleUiStyle" :title="uiStyle === 'modern' ? 'Ретро стиль' : 'Современный стиль'">
          {{ uiStyle === 'modern' ? '◐' : '◎' }}
        </button>
        <button class="theme-toggle" @click="toggleTheme" :title="isDark ? 'Светлая тема' : 'Тёмная тема'">
          {{ isDark ? '☀️' : '🌙' }}
        </button>
      </div>
    </header>

    <div class="input-section">
      <div class="input-blocks">
        <div class="input-block">
          <label>Контекст для AI</label>
          <textarea
            v-model="inputText"
            placeholder="Вставьте фразы из Яндекс Директ или Google Ads (одна фраза на строку)"
            rows="6"
          ></textarea>
          <div class="buttons">
            <button class="btn primary" @click="splitPhrases">Разбить на слова</button>
            <button class="btn" @click="clearProject" v-if="phrases.length">Очистить проект</button>
          </div>
        </div>
        <div class="input-block">
          <label>Контекст для AI</label>
          <textarea
            v-model="customPrompt"
            rows="8"
          ></textarea>
          <div class="buttons">
            <button class="btn primary" @click="processWithAI" :disabled="isLoading || !phrases.length">
              {{ isLoading ? 'Отправка...' : 'Отправить в AI' }}
            </button>
          </div>
        </div>
      </div>
    </div>

    <div class="tables-container">
      <div class="table-section">
        <div class="table-header">
          <h2>Фразы</h2>
          <span v-if="selectedWord || selectedWords.length" class="filter-badge" @click="clearFilter">
            Фильтр: {{ selectedWords.length > 1 ? selectedWords.length + ' слов' : selectedWord }} ×
          </span>
        </div>
        <div class="table-wrapper">
          <table>
            <thead>
              <tr>
                <th @click="sortPhrases('id')" :class="{ active: sortConfig.phrases.field === 'id', asc: sortConfig.phrases.asc }">
                  # {{ sortConfig.phrases.field === 'id' ? (sortConfig.phrases.asc ? '↑' : '↓') : '' }}
                </th>
                <th @click="sortPhrases('wordCount')" :class="{ active: sortConfig.phrases.field === 'wordCount', asc: sortConfig.phrases.asc }">
                  Слов {{ sortConfig.phrases.field === 'wordCount' ? (sortConfig.phrases.asc ? '↑' : '↓') : '' }}
                </th>
                <th @click="sortPhrases('text')" :class="{ active: sortConfig.phrases.field === 'text', asc: sortConfig.phrases.asc }">
                  Фраза {{ sortConfig.phrases.field === 'text' ? (sortConfig.phrases.asc ? '↑' : '↓') : '' }}
                </th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="phrase in sortedPhrases" :key="phrase.id">
                <td>{{ phrase.id + 1 }}</td>
                <td class="center">{{ phrase.wordCount }}</td>
                <td>{{ phrase.text }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <div class="table-section">
        <div class="table-header">
          <h2>Слова</h2>
          <div class="selection-controls">
            <span v-if="selectedWords.length">{{ selectedWords.length > 5 ? selectedWords.slice(0,5).join(', ') + '...' : selectedWords.join(', ') }}</span>
            <span v-else>Фильтр:</span>
            <select v-model="markerFilter" @change="applyMarkerFilter">
              <option value="">Все</option>
              <option v-for="m in MARKERS" :key="m.value" :value="m.value">{{ m.label }}</option>
            </select>
            <button v-if="selectedWords.length" class="btn-small" @click="copySelectedWords">Копировать</button>
            <select v-if="selectedWords.length" @change="setMarkerForSelected($event.target.value); $event.target.value=''">
              <option value="">Применить маркер</option>
              <option v-for="m in MARKERS" :key="m.value" :value="m.value">{{ m.label }}</option>
            </select>
          </div>
        </div>
        <div class="table-wrapper">
          <table>
            <thead>
              <tr>
                <th @click="sortWords('word')" :class="{ active: sortConfig.words.field === 'word', asc: sortConfig.words.asc }">
                  Слово {{ sortConfig.words.field === 'word' ? (sortConfig.words.asc ? '↑' : '↓') : '' }}
                </th>
                <th @click="sortWords('freq')" :class="{ active: sortConfig.words.field === 'freq', asc: sortConfig.words.asc }">
                  Частотность {{ sortConfig.words.field === 'freq' ? (sortConfig.words.asc ? '↑' : '↓') : '' }}
                </th>
                <th @click="sortWords('marker')" :class="{ active: sortConfig.words.field === 'marker', asc: sortConfig.words.asc }">
                  Маркер {{ sortConfig.words.field === 'marker' ? (sortConfig.words.asc ? '↑' : '↓') : '' }}
                </th>
              </tr>
            </thead>
            <tbody tabindex="0">
              <tr v-for="word in sortedWords" :key="word.word" @click="filterByWord(word.word)" @click.ctrl="toggleWordSelection(word.word, $event)" @click.meta="toggleWordSelection(word.word, $event)" @click.shift="toggleWordSelection(word.word, $event)" :class="{ clickable: true, selected: selectedWords.includes(word.word) }">
                <td class="word-cell">{{ word.word }}</td>
                <td class="readonly">{{ word.freq }}</td>
                <td @click.stop class="readonly">
                  <select :value="wordMarkers[word.word] || ''" @change="setMarker(word.word, $event.target.value)">
                    <option v-for="m in MARKERS" :key="m.value" :value="m.value">{{ m.label }}</option>
                  </select>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.app {
  display: flex;
  flex-direction: column;
  height: 100vh;
  padding: 16px;
  box-sizing: border-box;
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
}

.logo {
  display: flex;
  align-items: center;
  gap: 12px;
}

.logo-icon {
  font-size: 32px;
}

.logo-text h1 {
  font-size: 24px;
  font-weight: 700;
  margin: 0;
  letter-spacing: -0.02em;
}

.logo-subtitle {
  font-size: 12px;
  opacity: 0.6;
}

.header-buttons {
  display: flex;
  gap: 8px;
}

.style-toggle,
.theme-toggle {
  background: var(--hover-bg);
  border: none;
  border-radius: var(--radius);
  padding: 10px 14px;
  font-size: 20px;
  cursor: pointer;
  transition: all 0.2s ease;
  -webkit-tap-highlight-color: transparent;
}

.style-toggle:hover,
.theme-toggle:hover {
  background: var(--th-hover);
}

.input-section {
  margin-bottom: 16px;
}

.input-section .input-blocks {
  display: flex;
  gap: 16px;
  height: 200px;
}

.input-section .input-block {
  flex: 1;
  display: flex;
  flex-direction: column;
}

.input-section .input-block label {
  font-weight: 600;
  margin-bottom: 8px;
  font-size: 14px;
}

.context-form {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.input-section textarea {
  flex: 1;
  width: 100%;
  padding: 12px;
  border: 2px solid var(--border-color);
  border-radius: var(--radius);
  font-size: 14px;
  resize: none;
  box-sizing: border-box;
  background: var(--input-bg);
  color: var(--text-color);
  font-family: inherit;
  -webkit-tap-highlight-color: transparent;
  box-shadow: var(--shadow);
}

.input-section textarea:focus {
  outline: none;
  border-color: var(--primary-bg);
}

.btn {
  margin-top: 10px;
  margin-right: 10px;
  padding: 8px 16px;
  border: 2px solid var(--border-color);
  border-radius: var(--radius);
  font-size: 14px;
  font-weight: normal;
  cursor: pointer;
  transition: none;
  display: inline-block;
  background: var(--input-bg);
  color: var(--text-color);
  box-shadow: var(--shadow);
}

.btn:hover {
  transform: none;
}

.btn:active {
  transform: none;
}

.btn.primary {
  background: var(--primary-bg);
  color: var(--primary-text);
  border-color: #404040;
}

.btn.primary:hover {
  background: var(--primary-hover);
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn:disabled:hover {
  transform: none;
}

.tables-container {
  display: flex;
  gap: 16px;
  flex: 1;
  min-height: 0;
}

.table-section {
  flex: 1;
  display: flex;
  flex-direction: column;
  min-width: 0;
}

.table-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
}

.table-header h2 {
  font-size: 20px;
  font-weight: 600;
  margin: 0;
  letter-spacing: -0.01em;
}

.selection-controls {
  display: flex;
  align-items: center;
  gap: 10px;
}

.selection-controls span {
  font-size: 13px;
  font-weight: 500;
}

.selection-controls select {
  padding: 4px 8px;
  font-size: 13px;
}

.btn-small {
  padding: 4px 10px;
  font-size: 12px;
  border: 1px solid var(--border-color);
  border-radius: var(--radius);
  background: var(--input-bg);
  color: var(--text-color);
  cursor: pointer;
}

.filter-badge {
  background: var(--badge-bg);
  color: var(--badge-text);
  padding: 6px 12px;
  border-radius: 20px;
  font-size: 13px;
  font-weight: 500;
  cursor: pointer;
}

.filter-badge:hover {
  opacity: 0.85;
}

.table-wrapper {
  flex: 1;
  overflow: auto;
  border: 2px solid var(--border-color);
  border-radius: var(--radius);
  background: var(--input-bg);
  box-shadow: var(--shadow);
}

table {
  width: 100%;
  border-collapse: collapse;
  font-size: 14px;
  border: 1px solid var(--border-color);
}

th, td {
  padding: 8px 10px;
  text-align: left;
  border: 1px solid var(--border-color);
}

th {
  background: var(--th-bg);
  font-weight: bold;
  font-size: 13px;
  text-transform: none;
  letter-spacing: 0;
  color: var(--text-color);
  opacity: 1;
  cursor: pointer;
  user-select: none;
  position: sticky;
  top: 0;
  z-index: 1;
  border: 2px outset var(--border-color);
}

th:hover {
  background: var(--th-hover);
}

th.active {
  background: var(--th-active);
  border: 2px inset var(--border-color);
}

th.active.asc::after {
  content: ' ↑';
}

th.active:not(.asc)::after {
  content: ' ↓';
}

th:not(.active)::after {
  content: '';
}

tr.clickable:hover {
  background: var(--hover-bg);
  cursor: pointer;
}

tr.clickable td:first-child {
  color: var(--link-color);
  font-weight: 500;
}

td.center {
  text-align: center;
  width: 50px;
}

tr.selected {
  background: var(--th-active) !important;
}

tr.selected td.word-cell {
  font-weight: bold;
}

td.word-cell {
  user-select: text;
  cursor: pointer;
}

td.readonly {
  user-select: none;
  -webkit-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
  pointer-events: none;
}

td.readonly select {
  pointer-events: auto;
}

th {
  user-select: none;
  -webkit-user-select: none;
}

td.readonly select {
  pointer-events: auto;
  pointer-events: revert;
}

th {
  user-select: none;
}

select {
  padding: 4px 8px;
  border: 2px outset var(--border-color);
  border-radius: var(--radius);
  background: var(--input-bg);
  color: var(--text-color);
  font-size: 13px;
  font-family: inherit;
  cursor: pointer;
  -webkit-tap-highlight-color: transparent;
}

select:focus {
  outline: none;
  border-color: var(--primary-bg);
}
</style>