<script setup lang="ts">
defineProps<{ 
  messages: string[],
  theme?: 'dark' | 'light'
}>()
</script>

<template>
  <div class="console" :class="`theme-${theme}`">
    <div v-for="(msg, index) in messages" :key="index" class="console-line">
      <span class="console-prompt">›</span>
      {{ msg }}
    </div>
  </div>
</template>

<style>
.console {
  height: 150px;
  overflow-y: auto;
  padding: 10px;
  font-family: monospace;
  transition: background-color 0.3s, color 0.3s;
}

/* Тёмная тема для консоли */
.console.theme-dark {
  background: #1e1e1e;
  color: #fff;
}

/* Светлая тема для консоли */
.console.theme-light {
  background: #ffffff;
  color: #333333;
  border-top: 1px solid #e0e0e0;
}

.console-line {
  margin-bottom: 4px;
  word-break: break-word;
  display: flex;
  align-items: flex-start;
  gap: 6px;
}

.console-prompt {
  color: #4caf50;
  font-weight: bold;
  opacity: 0.7;
}

.console.theme-light .console-prompt {
  color: #2e7d32;
}

/* Стили для разных типов сообщений */
.console-line.error {
  color: #ff6b6b;
}

.console-line.warning {
  color: #ffd93d;
}

.console-line.success {
  color: #6bff6b;
}
</style>