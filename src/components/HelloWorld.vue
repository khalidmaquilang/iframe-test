<template>
    <div class="tester">
        <div class="toolbar">
            <span class="label">iframe URL</span>
            <input
                v-model="url"
                type="text"
                placeholder="https://your-laravel-app.test/gcash-pay/invoice-id"
                @keydown.enter="loadUrl"
            />
            <button @click="loadUrl">Load</button>
            <button @click="triggerGCash" :disabled="!ready">Send open_gcash</button>

            <span class="badge" :class="ready ? 'badge--ready' : 'badge--waiting'">
                {{ ready ? 'iframe ready' : 'waiting...' }}
            </span>
        </div>

        <div class="log-bar">
            <span class="log-title">postMessage log</span>
            <span class="clear" @click="logs = []">clear</span>
            <div class="log-entries">
                <span v-if="logs.length === 0" class="log-empty">No messages yet.</span>
                <div v-for="(log, i) in logs" :key="i" class="log-entry" :class="`log-entry--${log.dir}`">
                    <span class="log-time">{{ log.time }}</span>
                    <span class="log-dir">{{ log.dir === 'in' ? '← from iframe' : '→ to iframe' }}</span>
                    <code>{{ log.data }}</code>
                </div>
            </div>
        </div>

        <div class="frame-wrapper">
            <div v-if="!activeUrl" class="placeholder">
                Enter a URL above and click Load
            </div>

            <div v-if="activeUrl && !ready" class="loading-overlay">
                <div class="spinner" />
                <span>Loading...</span>
            </div>

            <iframe
                v-if="activeUrl"
                ref="iframeRef"
                :src="activeUrl"
                class="frame"
                scrolling="no"
                @load="onLoad"
            />
        </div>
    </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const iframeRef = ref(null)
const url       = ref('')
const activeUrl = ref('')
const ready     = ref(false)
const logs      = ref([])

function loadUrl() {
    if (!url.value.trim()) return
    ready.value     = false
    activeUrl.value = url.value.trim()
    addLog('out', `iframe src set to: ${activeUrl.value}`)
}

function onLoad() {
    // Fallback if gcash_ready never fires
    setTimeout(() => { ready.value = true }, 3000)
    addLog('out', 'iframe @load fired')
}

function triggerGCash() {
    iframeRef.value?.contentWindow?.postMessage('open_gcash', '*')
    addLog('out', 'open_gcash')
}

function onMessage(e) {
    const data = e.data
    addLog('in', JSON.stringify(data))

    if (data?.type === 'gcash_ready') {
        ready.value = true
    }
}

function addLog(dir, data) {
    const now = new Date()
    const time = now.toLocaleTimeString('en-US', { hour12: false }) +
        '.' + String(now.getMilliseconds()).padStart(3, '0')
    logs.value.unshift({ dir, data, time })
    if (logs.value.length > 50) logs.value.pop()
}

onMounted(()  => window.addEventListener('message', onMessage))
onUnmounted(() => window.removeEventListener('message', onMessage))
</script>

<style scoped>
.tester {
    display: flex;
    flex-direction: column;
    height: 100vh;
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
    font-size: 13px;
    background: #f5f5f5;
}

.toolbar {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 10px 14px;
    background: #fff;
    border-bottom: 1px solid #e0e0e0;
    flex-shrink: 0;
}

.label {
    color: #666;
    white-space: nowrap;
    font-weight: 500;
}

.toolbar input {
    flex: 1;
    height: 34px;
    padding: 0 10px;
    border: 1px solid #d0d0d0;
    border-radius: 6px;
    font-size: 13px;
    outline: none;
    font-family: monospace;
}

.toolbar input:focus {
    border-color: #204FC9;
}

.toolbar button {
    height: 34px;
    padding: 0 14px;
    border: 1px solid #d0d0d0;
    border-radius: 6px;
    background: #fff;
    cursor: pointer;
    font-size: 13px;
    white-space: nowrap;
    transition: background 0.15s;
}

.toolbar button:hover:not(:disabled) {
    background: #f0f0f0;
}

.toolbar button:disabled {
    opacity: 0.4;
    cursor: not-allowed;
}

.badge {
    padding: 4px 10px;
    border-radius: 20px;
    font-size: 11px;
    font-weight: 600;
    white-space: nowrap;
}

.badge--ready   { background: #dcfce7; color: #166534; }
.badge--waiting { background: #fef9c3; color: #854d0e; }

.log-bar {
    display: flex;
    flex-direction: column;
    padding: 8px 14px;
    background: #1e1e1e;
    color: #ccc;
    max-height: 160px;
    flex-shrink: 0;
}

.log-title {
    font-size: 11px;
    font-weight: 600;
    color: #888;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    margin-bottom: 4px;
}

.clear {
    position: absolute;
    right: 14px;
    font-size: 11px;
    color: #555;
    cursor: pointer;
    margin-top: -16px;
}

.clear:hover { color: #aaa; }

.log-entries {
    display: flex;
    flex-direction: column;
    gap: 2px;
    overflow-y: auto;
    flex: 1;
}

.log-empty {
    color: #555;
    font-style: italic;
    font-size: 12px;
}

.log-entry {
    display: flex;
    align-items: baseline;
    gap: 8px;
    font-family: monospace;
    font-size: 12px;
    line-height: 1.6;
}

.log-entry--in  code { color: #86efac; }
.log-entry--out code { color: #93c5fd; }

.log-time { color: #555; font-size: 11px; }
.log-dir  { color: #888; font-size: 11px; white-space: nowrap; }

.frame-wrapper {
    flex: 1;
    position: relative;
    overflow: hidden;
}

.placeholder {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    color: #aaa;
    font-size: 14px;
}

.loading-overlay {
    position: absolute;
    inset: 0;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 10px;
    background: #f5f5f5;
    z-index: 10;
    color: #666;
}

.spinner {
    width: 24px;
    height: 24px;
    border: 2px solid rgba(32, 79, 201, 0.15);
    border-top-color: #204FC9;
    border-radius: 50%;
    animation: spin 0.8s linear infinite;
}

@keyframes spin { to { transform: rotate(360deg); } }

.frame {
    width: 100%;
    height: 100%;
    border: none;
    display: block;
}
</style>