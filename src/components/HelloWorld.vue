<template>
    <div class="tester">
        <!-- Row 1: URL + load -->
        <div class="toolbar">
            <span class="label">iframe URL</span>
            <input
                v-model="url"
                type="text"
                placeholder="https://your-laravel-app.test/gcash-pay/invoice-id"
                @keydown.enter="loadUrl"
            />
            <button class="primary" @click="loadUrl">Load</button>

            <span class="badge" :class="loaded ? 'badge--ok' : 'badge--wait'">
                {{ loaded ? 'loaded' : 'not loaded' }}
            </span>
            <span class="badge" :class="handshake ? 'badge--ok' : 'badge--wait'">
                {{ handshake ? 'handshake' : 'no handshake' }}
            </span>
        </div>

        <!-- Row 2: sandbox matrix -->
        <div class="toolbar toolbar--alt">
            <span class="label">preset</span>
            <select v-model="presetKey" @change="applyPreset">
                <option v-for="(p, k) in presets" :key="k" :value="k">{{ p.name }}</option>
            </select>

            <label class="check">
                <input type="checkbox" v-model="useSandbox" /> sandbox
            </label>

            <input
                v-model="sandboxTokens"
                :disabled="!useSandbox"
                class="mono"
                placeholder="allow-scripts allow-same-origin"
            />

            <button @click="reloadFrame">Apply &amp; reload</button>
        </div>

        <!-- Row 3: viewport + probes -->
        <div class="toolbar toolbar--alt">
            <span class="label">frame width</span>
            <button
                v-for="w in widths"
                :key="w.label"
                :class="{ active: frameWidth === w.value }"
                @click="frameWidth = w.value"
            >{{ w.label }}</button>

            <span class="spacer" />

            <button @click="probeFrame">Probe frame</button>
            <button @click="triggerGCash" :disabled="!handshake">
                postMessage open_gcash
            </button>
            <span class="note">(expected to fail — no user activation)</span>
        </div>

        <!-- Probe results -->
        <div v-if="probe" class="probe">
            <span class="probe-title">Probe</span>
            <span
                v-for="(v, k) in probe"
                :key="k"
                class="chip"
                :class="typeof v === 'boolean' ? (v ? 'chip--yes' : 'chip--no') : ''"
            >{{ k }}: {{ v }}</span>
        </div>

        <!-- postMessage log -->
        <div class="log-bar">
            <div class="log-head">
                <span class="log-title">postMessage log</span>
                <span class="clear" @click="logs = []">clear</span>
            </div>
            <div class="log-entries">
                <span v-if="logs.length === 0" class="log-empty">No messages yet.</span>
                <div
                    v-for="(log, i) in logs"
                    :key="i"
                    class="log-entry"
                    :class="`log-entry--${log.dir}`"
                >
                    <span class="log-time">{{ log.time }}</span>
                    <span class="log-dir">{{ dirLabel(log.dir) }}</span>
                    <span v-if="log.origin" class="log-origin">{{ log.origin }}</span>
                    <code>{{ log.data }}</code>
                </div>
            </div>
        </div>

        <!-- Frame -->
        <div class="frame-wrapper">
            <div v-if="!activeUrl" class="placeholder">
                Enter a URL above and click Load
            </div>

            <div
                v-if="activeUrl"
                class="frame-shell"
                :style="frameWidth ? { width: frameWidth + 'px' } : { width: '100%' }"
            >
                <div class="frame-meta">
                    {{ frameWidth ? frameWidth + 'px' : 'full width' }}
                    <template v-if="useSandbox"> · sandbox="{{ sandboxTokens }}"</template>
                    <template v-else> · no sandbox</template>
                </div>

                <iframe
                    :key="frameKey"
                    ref="iframeRef"
                    :src="activeUrl"
                    :sandbox="useSandbox ? sandboxTokens : undefined"
                    class="frame"
                    scrolling="no"
                    @load="onLoad"
                />
            </div>
        </div>
    </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const iframeRef = ref(null)
const url       = ref('')
const activeUrl = ref('')
const loaded    = ref(false)
const handshake = ref(false)
const logs      = ref([])
const probe     = ref(null)
const frameKey  = ref(0)

/* ------------------------------------------------------------------ *
 * Sandbox matrix
 * ------------------------------------------------------------------ */
const presets = {
    none: {
        name: 'No sandbox (most permissive)',
        sandbox: false,
        tokens: '',
    },
    full: {
        name: 'Sandbox + top-nav (recommended embed)',
        sandbox: true,
        tokens: 'allow-scripts allow-same-origin allow-forms allow-top-navigation-by-user-activation',
    },
    noTopNav: {
        name: 'Sandbox, no top-nav (deeplink should fail)',
        sandbox: true,
        tokens: 'allow-scripts allow-same-origin allow-forms',
    },
    popupsOnly: {
        name: 'Sandbox + popups only',
        sandbox: true,
        tokens: 'allow-scripts allow-same-origin allow-forms allow-popups',
    },
    opaque: {
        name: 'Sandbox, no same-origin (opaque — session dies)',
        sandbox: true,
        tokens: 'allow-scripts allow-forms',
    },
}

const presetKey     = ref('none')
const useSandbox    = ref(false)
const sandboxTokens = ref('allow-scripts allow-same-origin')

function applyPreset() {
    const p = presets[presetKey.value]
    useSandbox.value    = p.sandbox
    sandboxTokens.value = p.tokens
    addLog('out', `preset → ${p.name}`)
    if (activeUrl.value) reloadFrame()
}

/* ------------------------------------------------------------------ *
 * Viewport width (reproduces the CSS media-query trap)
 * ------------------------------------------------------------------ */
const widths = [
    { label: 'full',   value: null },
    { label: '360',    value: 360  },
    { label: '400',    value: 400  },
    { label: '768',    value: 768  },
    { label: '1024',   value: 1024 },
]
const frameWidth = ref(null)

/* ------------------------------------------------------------------ *
 * Frame lifecycle
 * ------------------------------------------------------------------ */
function loadUrl() {
    if (!url.value.trim()) return
    activeUrl.value = url.value.trim()
    reloadFrame()
    addLog('out', `src = ${activeUrl.value}`)
}

function reloadFrame() {
    loaded.value    = false
    handshake.value = false
    probe.value     = null
    frameKey.value++ // force Vue to recreate the element so sandbox re-applies
}

function onLoad() {
    loaded.value = true
    addLog('sys', 'iframe load event')
}

/* ------------------------------------------------------------------ *
 * Probe: what can the framed page actually do?
 * Run from the parent side; mirrors what the cashier page should
 * self-report via sendBeacon in production.
 * ------------------------------------------------------------------ */
function probeFrame() {
    const el = iframeRef.value
    const result = {
        sandbox_attr: useSandbox.value ? sandboxTokens.value : '(none)',
        frame_width: el ? el.getBoundingClientRect().width : 0,
    }

    // Can the parent reach into the frame? Cross-origin will throw —
    // that's expected and not a failure.
    try {
        void el.contentWindow.location.href
        result.parent_can_read_frame = true
    } catch (e) {
        result.parent_can_read_frame = false
    }

    result.top_nav_allowed = useSandbox.value
        ? /allow-top-navigation/.test(sandboxTokens.value)
        : true
    result.popups_allowed = useSandbox.value
        ? /allow-popups/.test(sandboxTokens.value)
        : true
    result.same_origin_allowed = useSandbox.value
        ? /allow-same-origin/.test(sandboxTokens.value)
        : true

    probe.value = result
    addLog('sys', JSON.stringify(result))
}

/* ------------------------------------------------------------------ *
 * postMessage
 * ------------------------------------------------------------------ */
function triggerGCash() {
    iframeRef.value?.contentWindow?.postMessage({ type: 'open_gcash' }, '*')
    addLog('out', JSON.stringify({ type: 'open_gcash' }))
}

function onMessage(e) {
    if (e.source !== iframeRef.value?.contentWindow) return

    addLog('in', typeof e.data === 'string' ? e.data : JSON.stringify(e.data), e.origin)

    if (e.data?.type === 'gcash_ready')   handshake.value = true
    if (e.data?.type === 'deeplink_result') {
        probe.value = { ...(probe.value || {}), ...e.data.payload }
    }
}

/* ------------------------------------------------------------------ *
 * Logging
 * ------------------------------------------------------------------ */
function addLog(dir, data, origin) {
    const now = new Date()
    const time =
        now.toLocaleTimeString('en-US', { hour12: false }) +
        '.' + String(now.getMilliseconds()).padStart(3, '0')
    logs.value.unshift({ dir, data, time, origin })
    if (logs.value.length > 80) logs.value.pop()
}

function dirLabel(dir) {
    if (dir === 'in')  return '← from iframe'
    if (dir === 'out') return '→ to iframe'
    return '· local'
}

onMounted(()   => window.addEventListener('message', onMessage))
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
    flex-wrap: wrap;
}

.toolbar--alt { background: #fafafa; }

.label {
    color: #666;
    white-space: nowrap;
    font-weight: 500;
}

.note {
    color: #999;
    font-size: 11px;
    white-space: nowrap;
}

.spacer { flex: 1; }

.toolbar input[type="text"],
.toolbar input:not([type]) {
    flex: 1;
    min-width: 200px;
    height: 34px;
    padding: 0 10px;
    border: 1px solid #d0d0d0;
    border-radius: 6px;
    font-size: 13px;
    outline: none;
}

.toolbar input.mono { font-family: monospace; font-size: 12px; }
.toolbar input:focus { border-color: #204FC9; }
.toolbar input:disabled { background: #f0f0f0; color: #aaa; }

.toolbar select {
    height: 34px;
    padding: 0 8px;
    border: 1px solid #d0d0d0;
    border-radius: 6px;
    background: #fff;
    font-size: 13px;
    cursor: pointer;
}

.check {
    display: flex;
    align-items: center;
    gap: 5px;
    color: #444;
    white-space: nowrap;
    cursor: pointer;
}

.check input { cursor: pointer; }

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

.toolbar button:hover:not(:disabled) { background: #f0f0f0; }
.toolbar button:disabled { opacity: 0.4; cursor: not-allowed; }
.toolbar button.primary { border-color: #204FC9; color: #204FC9; }
.toolbar button.active  { background: #204FC9; color: #fff; border-color: #204FC9; }

.badge {
    padding: 4px 10px;
    border-radius: 20px;
    font-size: 11px;
    font-weight: 600;
    white-space: nowrap;
}

.badge--ok   { background: #dcfce7; color: #166534; }
.badge--wait { background: #fef9c3; color: #854d0e; }

.probe {
    display: flex;
    align-items: center;
    gap: 6px;
    flex-wrap: wrap;
    padding: 8px 14px;
    background: #fff;
    border-bottom: 1px solid #e0e0e0;
    flex-shrink: 0;
}

.probe-title {
    font-size: 11px;
    font-weight: 600;
    color: #888;
    text-transform: uppercase;
    letter-spacing: 0.05em;
}

.chip {
    padding: 3px 8px;
    border-radius: 4px;
    background: #f0f0f0;
    color: #444;
    font-family: monospace;
    font-size: 11px;
}

.chip--yes { background: #dcfce7; color: #166534; }
.chip--no  { background: #fee2e2; color: #991b1b; }

.log-bar {
    display: flex;
    flex-direction: column;
    padding: 8px 14px;
    background: #1e1e1e;
    color: #ccc;
    max-height: 160px;
    flex-shrink: 0;
}

.log-head {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 4px;
}

.log-title {
    font-size: 11px;
    font-weight: 600;
    color: #888;
    text-transform: uppercase;
    letter-spacing: 0.05em;
}

.clear { font-size: 11px; color: #555; cursor: pointer; }
.clear:hover { color: #aaa; }

.log-entries {
    display: flex;
    flex-direction: column;
    gap: 2px;
    overflow-y: auto;
    flex: 1;
}

.log-empty { color: #555; font-style: italic; font-size: 12px; }

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
.log-entry--sys code { color: #d4d4d4; }

.log-time   { color: #555; font-size: 11px; }
.log-dir    { color: #888; font-size: 11px; white-space: nowrap; }
.log-origin { color: #a78bfa; font-size: 11px; white-space: nowrap; }

.frame-wrapper {
    flex: 1;
    position: relative;
    overflow: auto;
    display: flex;
    justify-content: center;
    background: #e8e8e8;
    padding: 12px;
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

.frame-shell {
    display: flex;
    flex-direction: column;
    background: #fff;
    box-shadow: 0 1px 4px rgba(0, 0, 0, 0.12);
    overflow: hidden;
}

.frame-meta {
    padding: 4px 8px;
    background: #333;
    color: #bbb;
    font-family: monospace;
    font-size: 11px;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
}

.frame {
    flex: 1;
    width: 100%;
    border: none;
    display: block;
}
</style>