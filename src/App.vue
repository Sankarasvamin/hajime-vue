<script setup>
import { computed, nextTick, onBeforeUnmount, onMounted, ref } from 'vue'

const INTELLIGENCE_API_URL = 'http://localhost:3000/api/intelligence'

function createStaticTextEntry(id, role, text) {
  return {
    id,
    role,
    kind: 'text',
    text,
    renderedText: text,
    isTyping: false,
  }
}

const chatLogs = ref([
  createStaticTextEntry(1, 'system', '[SYSTEM]> 侦察链路已建立。等待指令...'),
  createStaticTextEntry(2, 'intel', '[INTEL]> AML anomaly detector online. RPC bridge standby.'),
])

const userInput = ref('')
const currentTime = ref(formatTime(new Date()))
const isSending = ref(false)
const isRiskFlashing = ref(false)
const chatLogsRef = ref(null)

let timerId = null
let flashTimerId = null
const typewriterTimerIds = new Set()

function formatTime(date) {
  const year = date.getFullYear()
  const month = String(date.getMonth() + 1).padStart(2, '0')
  const day = String(date.getDate()).padStart(2, '0')
  const hours = String(date.getHours()).padStart(2, '0')
  const minutes = String(date.getMinutes()).padStart(2, '0')
  const seconds = String(date.getSeconds()).padStart(2, '0')
  return `${year}-${month}-${day} ${hours}:${minutes}:${seconds}`
}

function formatPayload(value) {
  if (value === null || value === undefined) {
    return 'NO DATA'
  }

  if (typeof value === 'string') {
    return value
  }

  try {
    return JSON.stringify(value, null, 2)
  } catch {
    return String(value)
  }
}

function createTextEntry(role, text) {
  return {
    id: Date.now() + Math.random(),
    role,
    kind: 'text',
    text,
    renderedText: '',
    isTyping: true,
  }
}

function animateTextEntry(entry) {
  let index = 0

  const tick = () => {
    entry.renderedText = entry.text.slice(0, index)
    index += 2

    if (index <= entry.text.length) {
      const timeoutId = window.setTimeout(tick, 12)
      typewriterTimerIds.add(timeoutId)
      return
    }

    entry.renderedText = entry.text
    entry.isTyping = false
  }

  tick()
}

function scrollLogsToBottom() {
  nextTick(() => {
    if (chatLogsRef.value) {
      chatLogsRef.value.scrollTop = chatLogsRef.value.scrollHeight
    }
  })
}

function appendLog(role, text) {
  const entry = createTextEntry(role, text)
  chatLogs.value.push(entry)
  animateTextEntry(entry)
  scrollLogsToBottom()
}

function appendTableLog(role, title, rows) {
  chatLogs.value.push({
    id: Date.now() + Math.random(),
    role,
    kind: 'table',
    title,
    rows,
    columns: getTableColumns(rows),
  })
  scrollLogsToBottom()
}

function appendWebCardLog(role, payload) {
  chatLogs.value.push({
    id: Date.now() + Math.random(),
    role,
    kind: 'web',
    payload,
  })
  scrollLogsToBottom()
}

function triggerRiskFlash() {
  isRiskFlashing.value = false

  if (flashTimerId !== null) {
    window.clearTimeout(flashTimerId)
  }

  requestAnimationFrame(() => {
    isRiskFlashing.value = true
    flashTimerId = window.setTimeout(() => {
      isRiskFlashing.value = false
      flashTimerId = null
    }, 1600)
  })
}

function containsHighRisk(result) {
  return formatPayload(result).toUpperCase().includes('HIGH RISK')
}

function normalizeDbRows(data) {
  if (Array.isArray(data)) {
    return data.map((row, index) => normalizeRow(row, index))
  }

  if (data && typeof data === 'object') {
    const nestedArrayKey = Object.keys(data).find((key) => Array.isArray(data[key]))
    if (nestedArrayKey) {
      return data[nestedArrayKey].map((row, index) => normalizeRow(row, index))
    }

    return [normalizeRow(data, 0)]
  }

  return [normalizeRow({ value: data }, 0)]
}

function normalizeRow(row, index) {
  if (row && typeof row === 'object' && !Array.isArray(row)) {
    return {
      __rowId: row.id || row.transaction_id || row.reference_code || `row-${index}`,
      ...row,
    }
  }

  return {
    __rowId: `row-${index}`,
    value: row,
  }
}

function getTableColumns(rows) {
  const columns = []
  for (const row of rows) {
    for (const key of Object.keys(row)) {
      if (key === '__rowId' || columns.includes(key)) {
        continue
      }
      columns.push(key)
    }
  }
  return columns
}

function hasTriggeredRule(row) {
  return Boolean(row?.triggered_rule)
}

function buildWebCardPayload(result) {
  const search = result?.actions?.search?.result || {}
  const sourceResult = search?.result || {}
  return {
    query: result?.actions?.search?.query || 'UNKNOWN QUERY',
    answer: sourceResult?.answer || result?.raw_ai || 'NO OPEN WEB INTELLIGENCE',
    results: Array.isArray(sourceResult?.results) ? sourceResult.results.slice(0, 4) : [],
  }
}

function renderDbLogs(result) {
  const sqlQuery = result?.actions?.sql?.query || 'UNKNOWN SQL'
  const dbData = result?.actions?.sql?.result?.data ?? result?.final ?? result

  appendLog('db', `[DB_RESULT]> SQL ROUTE LOCKED\n${sqlQuery}`)
  appendTableLog('db', '[DB_RESULT]> SANKARA DATABASE RETURN', normalizeDbRows(dbData))
}

function renderWebLogs(result) {
  appendLog('web', `[WEB_RESULT]> OPEN WEB CHANNEL LOCKED\n${result?.actions?.search?.query || 'UNKNOWN QUERY'}`)
  appendWebCardLog('web', buildWebCardPayload(result))
}

function renderChatLogs(result) {
  const text = result?.raw_ai || result?.final || result
  appendLog('response', `[CLAUDE]> ${formatPayload(text)}`)
}

function renderResponse(result) {
  if (result.type === 'DB_RESULT') {
    renderDbLogs(result)
  } else if (result.type === 'WEB_RESULT') {
    renderWebLogs(result)
  } else {
    renderChatLogs(result)
  }

  if (containsHighRisk(result)) {
    appendLog('alert', '[ALERT]> HIGH RISK SIGNAL DETECTED. ENGAGING CRIMSON SCREEN PROTOCOL.')
    triggerRiskFlash()
  }
}

async function submitCommand() {
  const command = userInput.value.trim()
  if (!command || isSending.value) {
    return
  }

  appendLog('user', `[QUERY]> ${command}`)
  userInput.value = ''
  isSending.value = true
  appendLog('system', '>>> ANALYZING COMMAND...')
  appendLog('intel', '[ANALYZING... ACCESSING SANKARA CORE]')

  try {
    const response = await fetch(INTELLIGENCE_API_URL, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ text: command }),
    })

    const result = await response.json()

    if (!response.ok) {
      throw new Error(result?.error || 'Terminal request failed.')
    }

    renderResponse(result)
  } catch (error) {
    appendLog(
      'error',
      `[ERROR]> ${error instanceof Error ? error.message : '未知链路异常'}`
    )
  } finally {
    isSending.value = false
  }
}

const statusText = computed(() => `SYSTEM_TIME: ${currentTime.value}`)

onMounted(() => {
  timerId = window.setInterval(() => {
    currentTime.value = formatTime(new Date())
  }, 1000)
  scrollLogsToBottom()
})

onBeforeUnmount(() => {
  if (timerId !== null) {
    window.clearInterval(timerId)
  }

  if (flashTimerId !== null) {
    window.clearTimeout(flashTimerId)
  }

  for (const timeoutId of typewriterTimerIds) {
    window.clearTimeout(timeoutId)
  }
  typewriterTimerIds.clear()
})
</script>

<template>
  <div
    class="terminal-screen terminal-bg"
    :class="{ 'risk-flash': isRiskFlashing }"
  >
    <div class="terminal-shell border-alert text-surveillance">
      <div class="terminal-header">
        <span class="operator-tag">[OPERATOR: BENJAMIN_SANKARA_ACTIVE]</span>
        <span class="system-time">{{ statusText }}</span>
      </div>

      <div class="terminal-panels">
        <aside class="intel-panel">
          <div class="panel-title">SURVEILLANCE NODES</div>
          <div class="panel-card">
            <span class="label">CLAUDE</span>
            <span class="value">{{ isSending ? 'ANALYZING' : 'CONNECTED' }}</span>
          </div>
          <div class="panel-card">
            <span class="label">SUPABASE RPC</span>
            <span class="value">ARMED</span>
          </div>
          <div class="panel-card">
            <span class="label">TAVILY</span>
            <span class="value">STANDBY</span>
          </div>
          <div class="panel-card threat-card">
            <span class="label">THREAT LEVEL</span>
            <span class="value critical">{{ isRiskFlashing ? 'HIGH RISK' : 'CRIMSON' }}</span>
          </div>
        </aside>

        <main class="console-panel">
          <div
            id="chat-logs"
            ref="chatLogsRef"
            class="chat-logs"
          >
            <div
              v-for="entry in chatLogs"
              :key="entry.id"
              class="log-line"
              :class="`log-${entry.role}`"
            >
              <template v-if="entry.kind === 'table'">
                <div class="structured-title">{{ entry.title }}</div>
                <div class="table-shell">
                  <table class="intel-table">
                    <thead>
                      <tr>
                        <th
                          v-for="column in entry.columns"
                          :key="`${entry.id}-${column}`"
                        >
                          {{ column }}
                        </th>
                      </tr>
                    </thead>
                    <tbody>
                      <tr
                        v-for="row in entry.rows"
                        :key="row.__rowId"
                        :class="{ 'high-risk-row': hasTriggeredRule(row) }"
                      >
                        <td
                          v-for="column in entry.columns"
                          :key="`${row.__rowId}-${column}`"
                        >
                          {{ formatPayload(row[column]) }}
                        </td>
                      </tr>
                    </tbody>
                  </table>
                </div>
              </template>

              <template v-else-if="entry.kind === 'web'">
                <div class="web-card">
                  <div class="web-source">[SOURCE: OPEN_WEB_INTELLIGENCE]</div>
                  <div class="web-query">{{ entry.payload.query }}</div>
                  <div class="web-answer">{{ entry.payload.answer }}</div>
                  <div
                    v-for="item in entry.payload.results"
                    :key="item.url || item.title"
                    class="web-result-item"
                  >
                    <div class="web-result-title">{{ item.title || item.url }}</div>
                    <div class="web-result-body">{{ item.content || item.snippet || item.url }}</div>
                  </div>
                </div>
              </template>

              <template v-else>
                <span
                  class="typed-text"
                  :class="{ 'typing-active': entry.isTyping }"
                >
                  {{ entry.renderedText }}
                </span>
              </template>
            </div>
          </div>

          <div class="input-shell">
            <span class="prompt">SANKARA-ROOT@DB:~$</span>
            <input
              id="user-input"
              v-model="userInput"
              type="text"
              class="command-input"
              :disabled="isSending"
              :placeholder="isSending ? 'SANKARA CORE 正在解析指令...' : '输入查询指令或情报请求...'"
              @keydown.enter="submitCommand"
            >
          </div>
        </main>
      </div>
    </div>
  </div>
</template>

<style scoped>
.terminal-screen {
  min-height: 100vh;
  padding: 24px;
  background:
    radial-gradient(circle at top, rgba(139, 0, 0, 0.22), transparent 30%),
    radial-gradient(circle at bottom right, rgba(0, 255, 65, 0.12), transparent 24%),
    #000000;
}

.terminal-bg {
  background-color: #000000;
}

.risk-flash {
  animation: crimson-pulse 0.32s ease-in-out 2;
}

.terminal-shell {
  min-height: calc(100vh - 48px);
  border: 4px double #3f0000;
  padding: 24px;
  display: flex;
  flex-direction: column;
  background:
    linear-gradient(180deg, rgba(18, 0, 0, 0.9), rgba(0, 0, 0, 0.96)),
    #000000;
}

.text-surveillance {
  color: #00ff41;
  font-family: 'Courier New', monospace;
  text-shadow: 0 0 5px #00ff41;
}

.border-alert {
  border: 1px solid #8b0000;
  box-shadow:
    inset 0 0 15px #4a0000,
    0 0 24px rgba(139, 0, 0, 0.35);
}

.terminal-header {
  display: flex;
  justify-content: space-between;
  gap: 16px;
  align-items: center;
  padding-bottom: 12px;
  margin-bottom: 20px;
  border-bottom: 1px solid #5d0000;
}

.operator-tag {
  animation: blink 1s infinite;
  color: #ff0000;
  font-weight: bold;
}

.system-time {
  color: #c7ffcf;
}

.terminal-panels {
  display: grid;
  grid-template-columns: 280px 1fr;
  gap: 20px;
  flex: 1;
  min-height: 0;
}

.intel-panel {
  border: 1px solid #3f0000;
  background: rgba(14, 14, 14, 0.82);
  padding: 16px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.panel-title {
  color: #ff4d4d;
  font-size: 14px;
  letter-spacing: 0.16em;
  margin-bottom: 4px;
}

.panel-card {
  display: flex;
  justify-content: space-between;
  gap: 12px;
  padding: 12px;
  border: 1px solid #2b5e37;
  background: rgba(0, 16, 4, 0.72);
}

.threat-card {
  border-color: #8b0000;
  background: rgba(45, 0, 0, 0.55);
}

.label {
  color: #95ac99;
}

.value {
  color: #00ff41;
  font-weight: 700;
}

.critical {
  color: #ff3a3a;
}

.console-panel {
  display: flex;
  flex-direction: column;
  min-height: 0;
  border: 1px solid #3f0000;
  background:
    linear-gradient(180deg, rgba(0, 0, 0, 0.92), rgba(10, 0, 0, 0.92)),
    #000;
  padding: 16px;
}

.chat-logs {
  height: 70vh;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-bottom: 24px;
  padding-right: 6px;
}

.log-line {
  padding: 10px 12px;
  border-left: 3px solid #234a2d;
  background: rgba(0, 22, 5, 0.48);
  white-space: pre-wrap;
  word-break: break-word;
}

.typed-text {
  display: inline-block;
  min-height: 1.4em;
}

.typing-active::after {
  content: '_';
  margin-left: 2px;
  animation: blink 0.8s infinite;
}

.log-system,
.log-error,
.log-alert {
  color: #ff6b6b;
  border-left-color: #7b0000;
  background: rgba(46, 0, 0, 0.38);
}

.log-user {
  color: #c9ffd2;
}

.log-intel,
.log-response {
  color: #00ff41;
}

.log-db {
  color: #ffd166;
  border-left-color: #805b00;
  background: rgba(55, 37, 0, 0.34);
}

.log-web {
  color: #6ee7ff;
  border-left-color: #0f5d6a;
  background: rgba(0, 32, 40, 0.34);
}

.structured-title {
  margin-bottom: 10px;
  color: #9effad;
  font-weight: 700;
}

.table-shell {
  overflow-x: auto;
}

.intel-table {
  width: 100%;
  border-collapse: collapse;
  border: 1px solid #14532d;
  background: rgba(0, 0, 0, 0.5);
}

.intel-table th,
.intel-table td {
  padding: 10px 12px;
  border: 1px solid rgba(20, 83, 45, 0.85);
  vertical-align: top;
  text-align: left;
  white-space: pre-wrap;
}

.intel-table th {
  color: #00ff41;
  font-weight: 700;
  background: rgba(0, 20, 0, 0.78);
}

.high-risk-row {
  background: rgba(127, 29, 29, 0.3);
}

.web-card {
  border: 1px dashed #1e3a8a;
  background: rgba(2, 6, 23, 0.52);
  padding: 14px;
}

.web-source {
  margin-bottom: 8px;
  color: #7dd3fc;
  font-weight: 700;
}

.web-query {
  margin-bottom: 10px;
  color: #c4f1ff;
  font-weight: 700;
}

.web-answer {
  margin-bottom: 12px;
  color: #d9fbff;
  white-space: pre-wrap;
}

.web-result-item {
  padding-top: 10px;
  border-top: 1px solid rgba(30, 58, 138, 0.45);
}

.web-result-item + .web-result-item {
  margin-top: 10px;
}

.web-result-title {
  color: #93c5fd;
  font-weight: 700;
  margin-bottom: 4px;
}

.web-result-body {
  color: #bfdbfe;
  white-space: pre-wrap;
}

.input-shell {
  position: relative;
  margin-top: auto;
}

.prompt {
  position: absolute;
  left: 14px;
  top: 14px;
  color: #ff3a3a;
  font-size: 14px;
  text-shadow: none;
}

.command-input {
  width: 100%;
  background: transparent;
  border: none;
  border-bottom: 2px solid #5d0000;
  padding: 12px 14px 12px 170px;
  color: #00ff41;
  outline: none;
  transition: border-color 0.2s ease, opacity 0.2s ease;
  font: inherit;
  text-shadow: 0 0 5px #00ff41;
}

.command-input:disabled {
  opacity: 0.65;
  cursor: wait;
}

.command-input::placeholder {
  color: rgba(167, 255, 182, 0.45);
  text-shadow: none;
}

.command-input:focus {
  border-bottom-color: #ff3a3a;
}

@keyframes blink {
  0% {
    opacity: 1;
  }

  50% {
    opacity: 0;
  }

  100% {
    opacity: 1;
  }
}

@keyframes crimson-pulse {
  0% {
    background:
      radial-gradient(circle at top, rgba(139, 0, 0, 0.22), transparent 30%),
      radial-gradient(circle at bottom right, rgba(0, 255, 65, 0.12), transparent 24%),
      #000000;
  }

  50% {
    background:
      radial-gradient(circle at center, rgba(255, 0, 0, 0.6), transparent 42%),
      radial-gradient(circle at bottom right, rgba(255, 0, 0, 0.28), transparent 36%),
      #180000;
  }

  100% {
    background:
      radial-gradient(circle at top, rgba(139, 0, 0, 0.22), transparent 30%),
      radial-gradient(circle at bottom right, rgba(0, 255, 65, 0.12), transparent 24%),
      #000000;
  }
}

@media (max-width: 900px) {
  .terminal-screen {
    padding: 12px;
  }

  .terminal-shell {
    min-height: calc(100vh - 24px);
    padding: 16px;
  }

  .terminal-header {
    flex-direction: column;
    align-items: flex-start;
  }

  .terminal-panels {
    grid-template-columns: 1fr;
  }

  .chat-logs {
    height: 58vh;
  }

  .prompt {
    position: static;
    display: block;
    margin-bottom: 8px;
  }

  .command-input {
    padding-left: 14px;
  }
}
</style>
