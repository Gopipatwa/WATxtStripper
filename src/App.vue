<script setup>
import { ref, computed, watch } from 'vue'

// State
const inputText = ref('')
const userEmojis = ref([]) // [{ name, emoji }]
const defaultEmojis = ['😊', '😉', '😎', '🤗', '😇', '😃', '😏', '😅', '😜', '🤩']
const isDarkMode = ref(window.matchMedia('(prefers-color-scheme: dark)').matches)

// Toggle dark mode
function toggleDarkMode() {
  isDarkMode.value = !isDarkMode.value
  document.documentElement.classList.toggle('dark', isDarkMode.value)
}

// Initialize dark mode
document.documentElement.classList.toggle('dark', isDarkMode.value)

// Regex for WhatsApp export: [16/06, 9:25 pm] Name:
const msgRegex = /^\[(\d{1,2}\/\d{1,2}),\s*([\d:]+\s*[ap]m)]\s([^:]+):/gmi

// Extract unique users from input
function extractUsers(text) {
  const users = new Set()
  let match
  while ((match = msgRegex.exec(text))) {
    users.add(match[3].trim())
  }
  return Array.from(users)
}

// Watch inputText and update userEmojis
watch(inputText, (val) => {
  const users = extractUsers(val)
  userEmojis.value = users.map((name, i) => {
    const found = userEmojis.value.find(u => u.name === name)
    return found || { name, emoji: defaultEmojis[i % defaultEmojis.length] }
  })
})

// Output text with names replaced by emoji
const outputText = computed(() => {
  let out = inputText.value
  userEmojis.value.forEach(u => {
    const pattern = new RegExp(`(\\[\\d{1,2}/\\d{1,2},\\s*[\\d:]+\\s*[ap]m]\\s*)${u.name.replace(/[.*+?^${}()|[\\]\\]/g, '\\$&')}:`, 'gmi')
    out = out.replace(pattern, `${u.emoji}:`)
  })
  return out
})

// Detect direction (RTL/LTR)
const isRTL = computed(() => /[\u0591-\u07FF\uFB1D-\uFDFD\uFE70-\uFEFC]/.test(outputText.value))

// Actions
const showToast = ref(false)
const toastMessage = ref('')

function showNotification(message) {
  toastMessage.value = message
  showToast.value = true
  setTimeout(() => {
    showToast.value = false
  }, 3000)
}

async function copyToClipboard() {
  try {
    await navigator.clipboard.writeText(outputText.value)
    showNotification('Copied to clipboard!')
  } catch (err) {
    showNotification('Failed to copy text')
  }
}

function saveToFile() {
  try {
    const blob = new Blob([outputText.value], { type: 'text/plain' })
    const a = document.createElement('a')
    a.href = URL.createObjectURL(blob)
    a.download = 'cleaned_chat.txt'
    a.click()
    URL.revokeObjectURL(a.href)
    showNotification('File saved successfully!')
  } catch (err) {
    showNotification('Failed to save file')
  }
}

</script>

<template>
  <div class="app" :class="{ dark: isDarkMode }">
    <div class="toast" v-if="showToast" role="alert">
      {{ toastMessage }}
    </div>
    <div class="container">
      <header class="shadow-sm">
        <h1>Whatsapp Text Stripper</h1>
        <button class="theme-toggle glass" @click="toggleDarkMode" :title="isDarkMode ? 'Switch to Light Mode' : 'Switch to Dark Mode'">
          {{ isDarkMode ? '🌙' : '☀️' }}
        </button>
      </header>

      <main>
        <div class="input-section shadow-md">
          <textarea
            v-model="inputText"
            placeholder="Paste WhatsApp chat here..."
            class="text-input"
            rows="8"
          ></textarea>
        </div>

        <div v-if="userEmojis.length" class="users-section shadow-md glass">
          <h2>Replace Users</h2>
          <div class="user-grid">
            <div v-for="u in userEmojis" :key="u.name" class="user-item shadow-sm">
              <span class="user-name">{{ u.name }}</span>
              <input v-model="u.emoji" maxlength="2" class="emoji-input" />
            </div>
          </div>
        </div>

        <div class="output-section shadow-lg">
          <h2>Output Preview</h2>
          <pre class="text-output" :dir="isRTL ? 'rtl' : 'ltr'">{{ outputText }}</pre>
          <div class="actions">
            <button class="shadow-sm transition-all" @click="copyToClipboard">Copy</button>
            <button class="shadow-sm transition-all" @click="saveToFile">Save</button>
          </div>
        </div>
      </main>
    </div>
  </div>
</template>

<style scoped>
.app {
  min-height: 100vh;
  background: var(--bg-primary);
  color: var(--text-primary);
  transition: background-color 0.3s, color 0.3s;
  overflow-x: hidden;
}

.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 2rem;
}

header {
  position: sticky;
  top: 0;
  z-index: 10;
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 2rem;
  padding: 1rem 2rem;
  background: var(--glass-bg);
  backdrop-filter: blur(8px);
  -webkit-backdrop-filter: blur(8px);
  border-radius: 1rem;
}

h1 {
  font-size: 1.875rem;
  font-weight: 600;
}

h2 {
  font-size: 1.25rem;
  font-weight: 500;
  margin-bottom: 1rem;
  color: var(--text-secondary);
}

.theme-toggle {
  background: none;
  border: none;
  font-size: 1.5rem;
  cursor: pointer;
  padding: 0.75rem;
  border-radius: 0.75rem;
  transition: all 0.2s ease;
}

.theme-toggle:hover {
  background: var(--bg-secondary);
  transform: rotate(15deg);
}

.text-input, .text-output {
  width: 100%;
  padding: 1rem;
  border: 1px solid var(--border-color);
  border-radius: 0.75rem;
  background: var(--bg-secondary);
  color: var(--text-primary);
  font-size: 1rem;
  line-height: 1.6;
  transition: all 0.2s ease;
  overflow: auto;
  resize: none;
  height: auto;
  min-height: 200px;
  max-height: 50vh;
}

.text-input:focus {
  outline: none;
  border-color: var(--accent-color);
}

.text-output {
  white-space: pre-wrap;
  text-align: left;
  margin: 0;
  background: var(--bg-primary);
}

.users-section {
  margin: 2rem 0;
  padding: 1.5rem;
  border-radius: 1rem;
  overflow: hidden;
}

.user-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
  gap: 1rem;
  padding: 0.5rem;
  max-height: 300px;
  overflow-y: auto;
}

.user-item {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding: 0.75rem;
  background: var(--bg-primary);
  border-radius: 0.75rem;
  border: 1px solid var(--border-color);
  transition: transform 0.2s ease;
}

.user-item:hover {
  transform: translateY(-2px);
}

.emoji-input {
  width: 3rem;
  padding: 0.25rem;
  text-align: center;
  font-size: 1.125rem;
  background: var(--bg-secondary);
  border: 1px solid var(--border-color);
  border-radius: 0.5rem;
  color: var(--text-primary);
}

.input-section, .output-section {
  padding: 1.5rem;
  background: var(--bg-secondary);
  border-radius: 1rem;
  margin-bottom: 2rem;
}

.actions {
  display: flex;
  gap: 1rem;
  margin-top: 1.5rem;
}

button {
  padding: 0.75rem 1.5rem;
  border: none;
  border-radius: 0.75rem;
  color: white;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s ease;
  position: relative;
  overflow: hidden;
  min-width: 120px;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: var(--shadow-md);
}

button::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: linear-gradient(rgba(255,255,255,0.1), rgba(255,255,255,0));
  opacity: 0;
  transition: opacity 0.2s ease;
}

button:hover {
  transform: translateY(-2px);
  box-shadow: var(--shadow-lg);
}

button:hover::before {
  opacity: 1;
}

button:active {
  transform: translateY(1px);
  box-shadow: var(--shadow-sm);
}

.actions button:first-child {
  background: linear-gradient(135deg, var(--copy-from), var(--copy-to));
}

.actions button:nth-child(2) {
  background: linear-gradient(135deg, var(--save-from), var(--save-to));
}

.toast {
  position: fixed;
  bottom: 2rem;
  left: 50%;
  transform: translateX(-50%);
  padding: 0.75rem 1.5rem;
  background: var(--accent-color);
  color: white;
  border-radius: 0.75rem;
  z-index: 100;
  box-shadow: var(--shadow-lg);
  animation: slideUp 0.3s ease;
}

@keyframes slideUp {
  from {
    transform: translate(-50%, 100%);
    opacity: 0;
  }
  to {
    transform: translate(-50%, 0);
    opacity: 1;
  }
}

@media (max-width: 768px) {
  .container {
    padding: 1rem;
  }
  
  header {
    padding: 0.75rem 1rem;
  }
  
  .user-grid {
    grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  }
}
</style>
