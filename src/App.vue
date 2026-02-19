<script setup lang="ts">
import { ref, watch, onMounted, onUnmounted } from 'vue'  // ← ДОБАВЛЕН ИМПОРТ
import CodeEditor from './components/CodeEditor.vue'
import P5Canvas from './components/P5Canvas.vue'
import ConsoleOutput from './components/ConsoleOutput.vue'
import { saveAs } from 'file-saver'

const code = ref(`function setup() {
  createCanvas(400, 400);
}

function draw() {
  background(220);
  ellipse(mouseX, mouseY, 50, 50);
  if (frameCount % 60 === 0) {
    console.log("Прошло 60 кадров");
  }
}`)

// Сохраняем оригинальный код для сброса
const originalCode = ref(code.value)

const messages = ref<string[]>([])
const canvasRef = ref<InstanceType<typeof P5Canvas> | null>(null)
const fileInput = ref<HTMLInputElement | null>(null)

// Параметры шрифта
const fontSize = ref(15)
const fontFamily = ref('Consolas, Monaco, monospace')
const availableFonts = [
  { name: 'Consolas', value: 'Consolas, Monaco, monospace' },
  { name: 'Monaco', value: 'Monaco, Consolas, monospace' },
  { name: 'Courier New', value: '"Courier New", Courier, monospace' },
  { name: 'Fira Code', value: '"Fira Code", Consolas, monospace' },
  { name: 'JetBrains Mono', value: '"JetBrains Mono", Consolas, monospace' },
  { name: 'Source Code Pro', value: '"Source Code Pro", Consolas, monospace' },
  { name: 'Ubuntu Mono', value: '"Ubuntu Mono", Consolas, monospace' },
  { name: 'Menlo', value: 'Menlo, Consolas, monospace' }
]

// Параметры темы
type Theme = 'dark' | 'light'
const theme = ref<Theme>('dark')

// История изменений для Undo/Redo
const history = ref<string[]>([])
const historyIndex = ref(-1)

function addMessage(msg: string) {
  messages.value.push(msg)
  // Автоскролл консоли вниз
  setTimeout(() => {
    const console = document.querySelector('.console')
    if (console) {
      console.scrollTop = console.scrollHeight
    }
  }, 100)
}

function clearConsole() {
  messages.value = []
  addMessage('🧹 Консоль очищена')
}

function startSketch() {
  addMessage('🚀 Запуск скетча...')
  try {
    canvasRef.value?.start(code.value)
  } catch (e: any) {
    addMessage(`❌ Ошибка запуска: ${e.message}`)
  }
}

function stopSketch() {
  addMessage('⏹️ Остановка скетча...')
  canvasRef.value?.stop()
}

function saveSketch() {
  const blob = new Blob([code.value], { type: 'text/javascript;charset=utf-8' })
  saveAs(blob, `sketch_${new Date().toISOString().slice(0,19).replace(/:/g, '-')}.js`)
  addMessage('💾 Скетч сохранён')
}

function loadSketch() {
  fileInput.value?.click()
}

function handleFileUpload(event: Event) {
  const target = event.target as HTMLInputElement
  if (target.files && target.files[0]) {
    const reader = new FileReader()
    reader.onload = (e) => {
      saveToHistory()
      code.value = e.target?.result as string
      addMessage(`📂 Скетч загружен: ${target.files?.[0].name}`)
    }
    reader.readAsText(target.files[0])
  }
}

function increaseFontSize() {
  fontSize.value = Math.min(32, fontSize.value + 1)
}

function decreaseFontSize() {
  fontSize.value = Math.max(8, fontSize.value - 1)
}

function toggleTheme() {
  theme.value = theme.value === 'dark' ? 'light' : 'dark'
  addMessage(`🎨 Тема изменена на ${theme.value === 'dark' ? 'тёмную' : 'светлую'}`)
}

// Новые функции
function clearEditor() {
  if (confirm('Очистить редактор кода? Все изменения будут потеряны.')) {
    saveToHistory()
    code.value = ''
    addMessage('🧹 Редактор очищен')
  }
}

function resetToExample() {
  if (confirm('Восстановить пример кода?')) {
    saveToHistory()
    code.value = originalCode.value
    addMessage('🔄 Восстановлен пример кода')
  }
}

function copyToClipboard() {
  navigator.clipboard.writeText(code.value).then(() => {
    addMessage('📋 Код скопирован в буфер обмена')
  }).catch(() => {
    addMessage('❌ Не удалось скопировать код')
  })
}

function formatCode() {
  // Простое форматирование кода (базовое)
  try {
    saveToHistory()
    let formatted = code.value
      .replace(/\s*{\s*/g, ' {\n  ')
      .replace(/\s*}\s*/g, '\n}\n\n')
      .replace(/;\s*/g, ';\n  ')
      .replace(/\n\s*\n\s*\n/g, '\n\n')
      .replace(/{\n  \n/g, '{\n')
    
    code.value = formatted
    addMessage('✨ Код отформатирован')
  } catch (e) {
    addMessage('❌ Ошибка форматирования')
  }
}

function saveToHistory() {
  // Сохраняем текущее состояние для Undo
  if (code.value !== history.value[historyIndex.value]) {
    history.value = history.value.slice(0, historyIndex.value + 1)
    history.value.push(code.value)
    historyIndex.value = history.value.length - 1
  }
}

function undo() {
  if (historyIndex.value > 0) {
    historyIndex.value--
    code.value = history.value[historyIndex.value]
    addMessage('↩️ Отмена действия')
  } else {
    addMessage('⚠️ Нечего отменять')
  }
}

function redo() {
  if (historyIndex.value < history.value.length - 1) {
    historyIndex.value++
    code.value = history.value[historyIndex.value]
    addMessage('↪️ Возврат действия')
  } else {
    addMessage('⚠️ Нечего возвращать')
  }
}

function downloadScreenshot() {
  addMessage('📸 Генерация скриншота...')
  // Эта функция будет делать скриншот холста через p5.js
  // canvasRef.value?.screenshot()
}

function showShortcuts() {
  addMessage('⌨️ Горячие клавиши: Ctrl+Enter - запуск, Ctrl+S - сохранить, Ctrl+Z - отмена, Ctrl+Y - повтор')
}

// Обработчик клавиш для всего приложения
onMounted(() => {
  window.addEventListener('keydown', handleKeyDown)
})

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeyDown)
})

function handleKeyDown(e: KeyboardEvent) {
  // Ctrl+Enter - запуск скетча
  if (e.ctrlKey && e.key === 'Enter') {
    e.preventDefault()
    startSketch()
  }
  
  // Ctrl+S - сохранить
  if (e.ctrlKey && e.key === 's') {
    e.preventDefault()
    saveSketch()
  }
  
  // Ctrl+Z - отмена (если не в фокусе редактора)
  if (e.ctrlKey && e.key === 'z' && !e.target?.matches('textarea, input, .cm-content')) {
    e.preventDefault()
    undo()
  }
  
  // Ctrl+Y - повтор
  if (e.ctrlKey && e.key === 'y' && !e.target?.matches('textarea, input, .cm-content')) {
    e.preventDefault()
    redo()
  }
}
</script>

<template>
  <div class="app" :class="`theme-${theme}`">
    <!-- Декоративные элементы фона -->
    <div class="bg-decoration">
      <div class="bg-circle circle-1"></div>
      <div class="bg-circle circle-2"></div>
      <div class="bg-circle circle-3"></div>
      <div class="bg-grid"></div>
    </div>

    <div class="toolbar">
      <!-- Логотип с иконкой -->
      <div class="logo">
        <svg class="logo-icon" viewBox="0 0 24 24" width="24" height="24">
          <path d="M12 2L2 7v10l10 5 10-5V7l-10-5z" fill="#646cff" stroke="currentColor" stroke-width="1"/>
          <path d="M12 12l4-2v4l-4 2-4-2v-4l4 2z" fill="#ffffff" opacity="0.8"/>
          <circle cx="12" cy="12" r="2" fill="#ff6b6b"/>
        </svg>
        <span class="logo-text">p5.js Playground</span>
      </div>

      <div class="toolbar-buttons">
        <button @click="startSketch" class="action-btn start-btn" title="Запустить скетч (Ctrl+Enter)">
          <span class="btn-icon">▶</span>
          <span class="btn-text">Старт</span>
        </button>
        <button @click="stopSketch" class="action-btn stop-btn" title="Остановить скетч">
          <span class="btn-icon">■</span>
          <span class="btn-text">Стоп</span>
        </button>
        <button @click="saveSketch" class="action-btn save-btn" title="Сохранить скетч (Ctrl+S)">
          <span class="btn-icon">💾</span>
          <span class="btn-text">Сохранить</span>
        </button>
        <button @click="loadSketch" class="action-btn load-btn" title="Загрузить скетч">
          <span class="btn-icon">📂</span>
          <span class="btn-text">Загрузить</span>
        </button>
      </div>
      
      <input type="file" ref="fileInput" style="display: none" @change="handleFileUpload" accept=".js" />
      
      <!-- Новые кнопки для работы с кодом -->
      <div class="toolbar-divider"></div>
      
      <div class="toolbar-buttons">
        <button @click="clearEditor" class="action-btn clear-btn" title="Очистить редактор">
          <span class="btn-icon">🗑️</span>
          <span class="btn-text">Очистить</span>
        </button>
        <button @click="resetToExample" class="action-btn example-btn" title="Восстановить пример">
          <span class="btn-icon">🔄</span>
          <span class="btn-text">Пример</span>
        </button>
        <button @click="copyToClipboard" class="action-btn copy-btn" title="Копировать код">
          <span class="btn-icon">📋</span>
          <span class="btn-text">Копировать</span>
        </button>
        <button @click="formatCode" class="action-btn format-btn" title="Форматировать код">
          <span class="btn-icon">✨</span>
          <span class="btn-text">Формат</span>
        </button>
      </div>
      
      <div class="toolbar-divider"></div>
      
      <div class="toolbar-buttons">
        <button @click="undo" class="action-btn undo-btn" title="Отмена (Ctrl+Z)">
          <span class="btn-icon">↩️</span>
          <span class="btn-text">Отмена</span>
        </button>
        <button @click="redo" class="action-btn redo-btn" title="Повтор (Ctrl+Y)">
          <span class="btn-icon">↪️</span>
          <span class="btn-text">Повтор</span>
        </button>
        <button @click="clearConsole" class="action-btn console-clear-btn" title="Очистить консоль">
          <span class="btn-icon">🧹</span>
          <span class="btn-text">Консоль</span>
        </button>
        <button @click="showShortcuts" class="action-btn shortcuts-btn" title="Горячие клавиши">
          <span class="btn-icon">⌨️</span>
          <span class="btn-text">Клавиши</span>
        </button>
      </div>
      
      <!-- Кнопка переключения темы с анимацией -->
      <button @click="toggleTheme" class="theme-btn" :title="`Переключить на ${theme === 'dark' ? 'светлую' : 'тёмную'} тему`">
        <div class="theme-icon-container">
          <span class="theme-icon" :class="{ 'rotate': theme === 'light' }">
            <span v-if="theme === 'dark'">🌙</span>
            <span v-else>☀️</span>
          </span>
        </div>
      </button>
      
      <!-- Настройки шрифта с улучшенным дизайном -->
      <div class="font-controls">
        <div class="font-control-group">
          <span class="font-label">
            <svg viewBox="0 0 24 24" width="14" height="14" class="control-icon">
              <path d="M4 6h16v2H4V6zm2 4h12v2H6v-2zm3 4h6v2H9v-2z" fill="currentColor"/>
            </svg>
            Шрифт:
          </span>
          <select v-model="fontFamily" class="font-select">
            <option v-for="font in availableFonts" :key="font.value" :value="font.value">
              {{ font.name }}
            </option>
          </select>
        </div>
        
        <div class="font-control-group">
          <span class="font-label">
            <svg viewBox="0 0 24 24" width="14" height="14" class="control-icon">
              <path d="M12 6v12M8 8v8M16 8v8M4 10v4M20 10v4" stroke="currentColor" stroke-width="2"/>
            </svg>
            Размер:
          </span>
          <div class="font-size-controls">
            <button @click="decreaseFontSize" class="font-btn" title="Уменьшить шрифт">
              <span class="font-btn-icon">−</span>
            </button>
            <span class="font-size">{{ fontSize }}px</span>
            <button @click="increaseFontSize" class="font-btn" title="Увеличить шрифт">
              <span class="font-btn-icon">+</span>
            </button>
          </div>
        </div>
      </div>
    </div>

    <!-- Статус бар -->
    <div class="status-bar" :class="`theme-${theme}`">
      <div class="status-item">
        <span class="status-dot"></span>
        <span>Готов к работе</span>
      </div>
      <div class="status-item">
        <span class="status-icon">📊</span>
        <span>Сообщений: {{ messages.length }}</span>
      </div>
      <div class="status-item">
        <span class="status-icon">📝</span>
        <span>Символов: {{ code.length }}</span>
      </div>
      <div class="status-item">
        <span class="status-icon">⚡</span>
        <span>p5.js v1.9.4</span>
      </div>
    </div>

    <div class="main">
      <div class="editor-container" :class="`theme-${theme}`">
        <!-- Декоративная полоска над редактором -->
        <div class="editor-header">
          <div class="window-controls">
            <span class="window-control red"></span>
            <span class="window-control yellow"></span>
            <span class="window-control green"></span>
          </div>
          <span class="editor-title">sketch.js</span>
          <div class="editor-badge" v-if="history.length > 0">
            {{ historyIndex + 1 }}/{{ history.length }}
          </div>
        </div>
        <CodeEditor 
          v-model="code" 
          :font-size="fontSize"
          :font-family="fontFamily"
          :theme="theme"
          @update:model-value="saveToHistory"
        />
      </div>
      <div class="canvas-container">
        <div class="canvas-header">
          <span class="canvas-title">Холст p5.js</span>
          <div class="canvas-indicator"></div>
        </div>
        <P5Canvas ref="canvasRef" :add-message="addMessage" />
      </div>
    </div>
    <ConsoleOutput :messages="messages" :theme="theme" />
  </div>
</template>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

.app {
  display: flex;
  flex-direction: column;
  height: 100vh;
  transition: background-color 0.3s, color 0.3s;
  position: relative;
  overflow: hidden;
}

/* Декоративные элементы фона */
.bg-decoration {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  pointer-events: none;
  z-index: 0;
  overflow: hidden;
}

.bg-circle {
  position: absolute;
  border-radius: 50%;
  opacity: 0.03;
  filter: blur(60px);
}

.app.theme-dark .bg-circle {
  background: #646cff;
}

.app.theme-light .bg-circle {
  background: #ff6b6b;
}

.circle-1 {
  width: 400px;
  height: 400px;
  top: -100px;
  right: -100px;
}

.circle-2 {
  width: 300px;
  height: 300px;
  bottom: -50px;
  left: -50px;
}

.circle-3 {
  width: 200px;
  height: 200px;
  bottom: 200px;
  right: 200px;
}

.bg-grid {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-image: 
    linear-gradient(rgba(100, 108, 255, 0.05) 1px, transparent 1px),
    linear-gradient(90deg, rgba(100, 108, 255, 0.05) 1px, transparent 1px);
  background-size: 50px 50px;
}

/* Тёмная тема */
.app.theme-dark {
  background-color: #1a1a1a;
  color: #ffffff;
}

.app.theme-dark .toolbar {
  background: rgba(45, 45, 45, 0.8);
  backdrop-filter: blur(10px);
  border-bottom: 1px solid #404040;
}

/* Светлая тема */
.app.theme-light {
  background-color: #f8f9fa;
  color: #2c3e50;
}

.app.theme-light .toolbar {
  background: rgba(255, 255, 255, 0.9);
  backdrop-filter: blur(10px);
  border-bottom: 1px solid #e9ecef;
  box-shadow: 0 4px 20px rgba(0,0,0,0.05);
}

.toolbar {
  display: flex;
  gap: 20px;
  padding: 12px 20px;
  flex-wrap: wrap;
  align-items: center;
  transition: all 0.3s;
  z-index: 10;
  position: relative;
}

/* Логотип */
.logo {
  display: flex;
  align-items: center;
  gap: 10px;
  padding-right: 20px;
  border-right: 2px solid rgba(100, 108, 255, 0.3);
}

.logo-icon {
  animation: pulse 2s infinite;
}

.logo-text {
  font-weight: bold;
  font-size: 16px;
  background: linear-gradient(135deg, #646cff, #9089fc);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.toolbar-buttons {
  display: flex;
  gap: 8px;
}

/* Кнопки действий */
.action-btn {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 8px 14px;
  border-radius: 8px;
  font-weight: 500;
  transition: all 0.2s;
  position: relative;
  overflow: hidden;
  border: none;
  color: white;
  cursor: pointer;
}

.action-btn::before {
  content: '';
  position: absolute;
  top: 50%;
  left: 50%;
  width: 0;
  height: 0;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.2);
  transform: translate(-50%, -50%);
  transition: width 0.3s, height 0.3s;
}

.action-btn:hover::before {
  width: 200px;
  height: 200px;
}

.start-btn {
  background: linear-gradient(135deg, #4caf50, #45a049);
}

.stop-btn {
  background: linear-gradient(135deg, #f44336, #d32f2f);
}

.save-btn {
  background: linear-gradient(135deg, #2196f3, #1976d2);
}

.load-btn {
  background: linear-gradient(135deg, #ff9800, #f57c00);
}

/* Новые стили для дополнительных кнопок */
.clear-btn {
  background: linear-gradient(135deg, #9c27b0, #7b1fa2);
}

.example-btn {
  background: linear-gradient(135deg, #00bcd4, #0097a7);
}

.copy-btn {
  background: linear-gradient(135deg, #8bc34a, #689f38);
}

.format-btn {
  background: linear-gradient(135deg, #ffc107, #ff8f00);
}

.undo-btn {
  background: linear-gradient(135deg, #607d8b, #455a64);
}

.redo-btn {
  background: linear-gradient(135deg, #607d8b, #455a64);
}

.console-clear-btn {
  background: linear-gradient(135deg, #795548, #5d4037);
}

.shortcuts-btn {
  background: linear-gradient(135deg, #9e9e9e, #616161);
}

.btn-icon {
  font-size: 16px;
  line-height: 1;
}

/* Кнопка темы */
.theme-btn {
  background: transparent;
  border: 2px solid currentColor;
  padding: 6px;
  border-radius: 50%;
  width: 38px;
  height: 38px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.3s;
  cursor: pointer;
  color: inherit;
}

.theme-btn:hover {
  transform: rotate(180deg);
  background: rgba(100, 108, 255, 0.1);
}

.theme-icon {
  display: inline-block;
  font-size: 20px;
  transition: transform 0.5s;
}

.theme-icon.rotate {
  animation: spin 0.5s ease;
}

/* Настройки шрифта */
.font-controls {
  display: flex;
  align-items: center;
  gap: 15px;
  margin-left: auto;
  padding-left: 20px;
  border-left: 2px solid rgba(100, 108, 255, 0.3);
}

.font-control-group {
  display: flex;
  align-items: center;
  gap: 8px;
}

.font-label {
  display: flex;
  align-items: center;
  gap: 4px;
  font-size: 13px;
  color: inherit;
  opacity: 0.8;
}

.control-icon {
  opacity: 0.6;
}

.font-select {
  padding: 6px 10px;
  border-radius: 8px;
  background: rgba(0, 0, 0, 0.1);
  color: inherit;
  border: 1px solid rgba(100, 108, 255, 0.3);
  cursor: pointer;
  font-size: 13px;
  transition: all 0.2s;
}

.font-select:hover {
  border-color: #646cff;
  background: rgba(100, 108, 255, 0.1);
}

.font-size-controls {
  display: flex;
  align-items: center;
  gap: 6px;
  background: rgba(0, 0, 0, 0.1);
  border-radius: 8px;
  padding: 2px;
}

.font-btn {
  background: transparent;
  border: none;
  padding: 4px 10px;
  border-radius: 6px;
  cursor: pointer;
  color: inherit;
  transition: all 0.2s;
}

.font-btn:hover {
  background: rgba(100, 108, 255, 0.3);
}

.font-btn-icon {
  font-size: 16px;
  font-weight: bold;
}

.font-size {
  min-width: 45px;
  text-align: center;
  font-size: 13px;
  font-weight: 500;
}

/* Статус бар */
.status-bar {
  display: flex;
  gap: 20px;
  padding: 6px 20px;
  font-size: 12px;
  background: rgba(0, 0, 0, 0.2);
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
  z-index: 5;
  position: relative;
}

.app.theme-light .status-bar {
  background: rgba(0, 0, 0, 0.03);
  border-bottom-color: rgba(0, 0, 0, 0.05);
}

.status-item {
  display: flex;
  align-items: center;
  gap: 6px;
}

.status-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: #4caf50;
  animation: blink 2s infinite;
}

.status-icon {
  font-size: 14px;
  opacity: 0.7;
}

/* Заголовки редактора и холста */
.editor-container, .canvas-container {
  display: flex;
  flex-direction: column;
  height: 100%;
  position: relative;
}

.editor-header, .canvas-header {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 8px 12px;
  background: rgba(0, 0, 0, 0.2);
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}

.app.theme-light .editor-header,
.app.theme-light .canvas-header {
  background: rgba(255, 255, 255, 0.5);
  border-bottom-color: rgba(0, 0, 0, 0.1);
}

.window-controls {
  display: flex;
  gap: 6px;
}

.window-control {
  width: 12px;
  height: 12px;
  border-radius: 50%;
}

.window-control.red {
  background: #ff5f56;
}
.window-control.yellow {
  background: #ffbd2e;
}
.window-control.green {
  background: #27c93f;
}

.editor-title, .canvas-title {
  font-size: 13px;
  opacity: 0.7;
}

.canvas-indicator {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: #4caf50;
  animation: pulse 1.5s infinite;
}

.editor-badge {
  margin-left: auto;
  font-size: 11px;
  padding: 2px 6px;
  background: rgba(255, 255, 255, 0.1);
  border-radius: 10px;
  color: #888;
}

.toolbar-divider {
  width: 1px;
  height: 30px;
  background: rgba(255, 255, 255, 0.2);
  margin: 0 5px;
}

.app.theme-light .toolbar-divider {
  background: rgba(0, 0, 0, 0.1);
}

.main {
  display: flex;
  flex: 1;
  overflow: hidden;
  position: relative;
  z-index: 1;
}

.main > * {
  flex: 1;
  border: 1px solid transparent;
  transition: border-color 0.3s;
}

/* Анимации */
@keyframes pulse {
  0%, 100% {
    opacity: 1;
  }
  50% {
    opacity: 0.5;
  }
}

@keyframes spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

@keyframes blink {
  0%, 100% {
    opacity: 1;
  }
  50% {
    opacity: 0.3;
  }
}

@keyframes slideIn {
  from {
    opacity: 0;
    transform: translateY(-10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.action-btn {
  animation: slideIn 0.3s ease;
}

/* Улучшенные подсказки */
.action-btn[title] {
  position: relative;
}

.action-btn[title]:hover::after {
  content: attr(title);
  position: absolute;
  bottom: -30px;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(0, 0, 0, 0.8);
  color: white;
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 11px;
  white-space: nowrap;
  z-index: 1000;
  pointer-events: none;
}

/* Скроллбары */
::-webkit-scrollbar {
  width: 10px;
  height: 10px;
}

::-webkit-scrollbar-track {
  background: rgba(0, 0, 0, 0.1);
}

::-webkit-scrollbar-thumb {
  background: rgba(100, 108, 255, 0.3);
  border-radius: 5px;
}

::-webkit-scrollbar-thumb:hover {
  background: rgba(100, 108, 255, 0.5);
}
</style>