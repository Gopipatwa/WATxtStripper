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
  min-height: 90vh;
  background: var(--bg-primary);
  color: var(--text-primary);
  transition: background-color 0.3s, color 0.3s;
  overflow-x: hidden;
  -webkit-tap-highlight-color: transparent;
}

.container {
  width: min(100%, 800px);
  margin: 0 auto;
  min-height: fit-content;
  padding: 0.75rem;
  display: flex;
  flex-direction: column;
}

main {
  flex: 1;
  display: flex;
  flex-direction: column;
  justify-content: center;
  gap: 1rem;
  max-width: 100%;
}

.toast {
  position: fixed;
  bottom: 1rem;
  left: 50%;
  transform: translateX(-50%);
  padding: 0.75rem 1.5rem;
  background: var(--accent-color);
  color: white;
  border-radius: 0.75rem;
  z-index: 100;
  box-shadow: var(--shadow-lg);
  animation: slideUp 0.3s ease;
  width: calc(100% - 2rem);
  max-width: 400px;
  text-align: center;
}

header {
  position: sticky;
  top: 0;
  z-index: 10;
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1.5rem;
  padding: 0.75rem 1rem;
  background: var(--glass-bg);
  backdrop-filter: blur(8px);
  -webkit-backdrop-filter: blur(8px);
  border-radius: 0.75rem;
}

h1 {
  font-size: clamp(1.25rem, 5vw, 1.875rem);
  font-weight: 600;
}

h2 {
  font-size: clamp(1rem, 4vw, 1.25rem);
  font-weight: 500;
  margin-bottom: 0.75rem;
  color: var(--text-secondary);
}

.theme-toggle {
  width: 44px;
  height: 44px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: none;
  border: none;
  font-size: 1.5rem;
  cursor: pointer;
  border-radius: 50%;
  transition: all 0.2s ease;
  touch-action: manipulation;
}

.text-input, .text-output {
  width: 100%;
  padding: 0.75rem;
  border: 1px solid var(--border-color);
  border-radius: 0.75rem;
  background: var(--bg-secondary);
  color: var(--text-primary);
  font-size: 1rem;
  line-height: 1.5;
  transition: all 0.2s ease;
  overflow: auto;
  resize: none;
  height: clamp(120px, 25vh, 200px);
  max-height: unset;
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
  font-size: 0.9375rem;
}

.users-section {
  margin: 0.75rem 0;
  padding: 1rem;
  border-radius: 0.75rem;
  overflow: hidden;
}

.user-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 0.75rem;
  padding: 0.5rem;
  max-height: 160px;
  overflow-y: auto;
}

.user-item {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.75rem;
  background: var(--bg-primary);
  border-radius: 0.75rem;
  border: 1px solid var(--border-color);
  transition: transform 0.2s ease;
  touch-action: manipulation;
}

.user-name {
  font-size: 0.9375rem;
  word-break: break-word;
  flex: 1;
  min-width: 0;
}

.emoji-input {
  width: 44px;
  height: 44px;
  padding: 0.25rem;
  text-align: center;
  font-size: 1.25rem;
  background: var(--bg-secondary);
  border: 1px solid var(--border-color);
  border-radius: 0.5rem;
  color: var(--text-primary);
  flex-shrink: 0;
}

.input-section, .output-section {
  padding: 1rem;
  background: var(--bg-secondary);
  border-radius: 0.75rem;
  margin: 0;
}

.actions {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(100px, 1fr));
  gap: 0.75rem;
  margin-top: 1rem;
}

button {
  height: 44px;
  padding: 0 1rem;
  border: none;
  border-radius: 0.75rem;
  color: white;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s ease;
  position: relative;
  overflow: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: var(--shadow-md);
  touch-action: manipulation;
  -webkit-tap-highlight-color: transparent;
}

button:active {
  transform: translateY(1px);
}

.actions button:first-child {
  background: linear-gradient(135deg, var(--copy-from), var(--copy-to));
}

.actions button:nth-child(2) {
  background: linear-gradient(135deg, var(--save-from), var(--save-to));
}

@media (max-width: 480px) {
  .container {
    padding: 0.5rem;
  }
  
  header {
    padding: 0.5rem;
    margin-bottom: 0.75rem;
    position: relative;
  }
  
  .input-section, .output-section {
    padding: 0.75rem;
  }
  
  .text-input, .text-output {
    font-size: 0.9375rem;
    padding: 0.625rem;
  }
  
  .user-grid {
    grid-template-columns: 1fr;
    max-height: 160px;
  }
  
  .actions {
    grid-template-columns: repeat(2, 1fr);
  }
  
  button {
    width: 100%;
    height: 44px;
    font-size: 1rem;
  }
}

@media (hover: hover) {
  .theme-toggle:hover {
    background: var(--bg-secondary);
    transform: rotate(15deg);
  }
  
  .user-item:hover {
    transform: translateY(-2px);
  }
  
  button:hover {
    transform: translateY(-2px);
    box-shadow: var(--shadow-lg);
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
  
  button:hover::before {
    opacity: 1;
  }
}

@supports not (backdrop-filter: blur(8px)) {
  header {
    background: var(--bg-primary);
  }
}
</style>
