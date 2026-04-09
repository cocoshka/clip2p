<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import { Peer } from 'peerjs'
import { useEventListener } from '@vueuse/core'

const myPeerId = ref('')
const targetPeerId = ref('')
const connectionStatus = ref('disconnected') // disconnected, connecting, connected
const isSyncing = ref(false)
const logs = ref([])
const activeConnection = ref(null)

let peer = null

const initPeer = () => {
  peer = new Peer({
    config: {
      iceServers: [
        { urls: 'stun:stunserver2025.stunprotocol.org' },
        { urls: 'stun:stun.l.google.com:19302' },
        { urls: 'stun:stun1.l.google.com:19302' }
      ]
    }
  })

  peer.on('open', (id) => {
    myPeerId.value = id
    addLog('System', `Ready!`)
  })

  peer.on('connection', (conn) => {
    // When someone else connects to us
    setupConnection(conn)
  })

  peer.on('error', (err) => {
    console.error(err)
    addLog('Error', err.type)
    if (err.type !== 'peer-unavailable') {
      connectionStatus.value = 'disconnected'
    } else {
      connectionStatus.value = 'disconnected'
      addLog('Error', 'Peer not found')
    }
  })
}

const setupConnection = (conn) => {
  activeConnection.value = conn
  connectionStatus.value = 'connecting'

  conn.on('open', () => {
    connectionStatus.value = 'connected'
    addLog('System', 'Connected to peer!')
  })

  conn.on('data', (data) => {
    handleIncomingData(data)
  })

  conn.on('close', () => {
    connectionStatus.value = 'disconnected'
    activeConnection.value = null
    addLog('System', 'Connection closed.')
  })

  conn.on('error', (err) => {
    addLog('Error', 'Connection logic error')
  })
}

const connectToPeer = () => {
  if (!targetPeerId.value || !peer) return
  connectionStatus.value = 'connecting'
  const conn = peer.connect(targetPeerId.value, { reliable: true })
  setupConnection(conn)
}

const disconnect = () => {
  if (activeConnection.value) {
    activeConnection.value.close()
  }
}

const addLog = (type, msg) => {
  logs.value.unshift({
    time: new Date().toLocaleTimeString(),
    type: type.toLowerCase(),
    msg
  })
  if (logs.value.length > 30) logs.value.pop()
}

const copyMyId = async () => {
  try {
    await navigator.clipboard.writeText(myPeerId.value)
    addLog('Info', 'Copied ID')
  } catch (err) {
    addLog('Error', 'Failed to copy ID')
  }
}

// Keep track of the last synced content to avoid loopbacks
let lastSyncedText = ''

const pullClipboardAndSend = async () => {
  if (connectionStatus.value !== 'connected' || !activeConnection.value) return

  try {
    isSyncing.value = true

    // First, try to read complex items (images etc)
    try {
      const clipboardItems = await navigator.clipboard.read()
      let sentSomething = false

      for (const clipboardItem of clipboardItems) {
        for (const type of clipboardItem.types) {
          if (type.startsWith('text/')) {
            // we'll handle text separately below for simplicity and loopback prevention
            continue
          }
          const blob = await clipboardItem.getType(type)
          const arrayBuffer = await blob.arrayBuffer()
          activeConnection.value.send({
            isBlob: true,
            mime: type,
            payload: arrayBuffer
          })
          addLog('Sent', `File/Image (${type})`)
          sentSomething = true
        }
      }

      // Attempt text via readText since read() can sometimes be opaque for text
      const text = await navigator.clipboard.readText()
      if (text && text !== lastSyncedText) {
        lastSyncedText = text
        activeConnection.value.send({
          isText: true,
          payload: text
        })
        addLog('Sent', 'Text')
        sentSomething = true
      }

      if (!sentSomething) {
        addLog('Info', 'Clipboard unchanged')
      }

    } catch (e) {
      // Fallback to text only if read() permission denied or unsupported
      const text = await navigator.clipboard.readText()
      if (text && text !== lastSyncedText) {
        lastSyncedText = text
        activeConnection.value.send({
          isText: true,
          payload: text
        })
        addLog('Sent', 'Text')
      }
    }
  } catch (err) {
    console.warn('Clipboard read error:', err)
    // Often happens if not focused
  } finally {
    isSyncing.value = false
  }
}

const handleIncomingData = async (data) => {
  try {
    isSyncing.value = true
    if (data.isText) {
      lastSyncedText = data.payload
      await navigator.clipboard.writeText(data.payload)
      addLog('Received', 'Text')
    } else if (data.isBlob) {
      const blob = new Blob([data.payload], { type: data.mime })
      const item = new ClipboardItem({ [data.mime]: blob })
      await navigator.clipboard.write([item])
      addLog('Received', `File (${data.mime})`)
    }
  } catch (err) {
    console.error("Write fail:", err)
    addLog('Error', "Focus page to write!")
  } finally {
    setTimeout(() => { isSyncing.value = false }, 400)
  }
}

const manualSync = () => {
  pullClipboardAndSend()
}

const handleVisibilityChange = () => {
  if (document.visibilityState === 'visible' && connectionStatus.value === 'connected') {
    setTimeout(pullClipboardAndSend, 150)
  }
}

onMounted(() => {
  initPeer()
})

onUnmounted(() => {
  if (peer) peer.destroy()
})

useEventListener(document, 'visibilitychange', handleVisibilityChange)
useEventListener(window, 'focus', handleVisibilityChange)
</script>

<template>
  <main class="app-container">
    <div class="glass-panel main-panel">
      <!-- Header -->
      <header class="header">
        <div class="logo">
          <svg width="28" height="28" viewBox="0 0 24 24" fill="none" stroke="url(#gradient)" stroke-width="2"
            stroke-linecap="round" stroke-linejoin="round">
            <defs>
              <linearGradient id="gradient" x1="0%" y1="0%" x2="100%" y2="100%">
                <stop offset="0%" stop-color="#4facfe" />
                <stop offset="100%" stop-color="#00f2fe" />
              </linearGradient>
            </defs>
            <path d="M16 4h2a2 2 0 0 1 2 2v14a2 2 0 0 1-2 2H6a2 2 0 0 1-2-2V6a2 2 0 0 1 2-2h2"></path>
            <rect x="8" y="2" width="8" height="4" rx="1" ry="1"></rect>
          </svg>
          <h1>Clip2P</h1>
        </div>
        <div class="status" v-if="connectionStatus !== 'disconnected'">
          <span class="status-indicator"
            :class="connectionStatus === 'connected' ? 'status-connected' : 'status-connecting'"></span>
          <span class="status-text">{{ connectionStatus === 'connected' ? 'Connected' : 'Connecting...' }}</span>
        </div>
        <div class="status" v-else>
          <span class="status-indicator status-disconnected"></span>
          <span class="status-text">Disconnected</span>
        </div>
      </header>

      <!-- Connection Setup -->
      <section v-if="connectionStatus !== 'connected'" class="setup-section">
        <div class="my-id-box">
          <label>Your ID</label>
          <div class="id-wrapper">
            <input type="text" class="input id-input" readonly :value="myPeerId || 'Generating...'" />
            <button class="btn btn-icon" @click="copyMyId" :disabled="!myPeerId" title="Copy ID">
              <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <rect x="9" y="9" width="13" height="13" rx="2" ry="2"></rect>
                <path d="M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1"></path>
              </svg>
            </button>
          </div>
        </div>

        <div class="divider">
          <span>OR</span>
        </div>

        <div class="connect-box">
          <label>Connect to Peer</label>
          <div class="id-wrapper">
            <input type="text" class="input" v-model="targetPeerId" placeholder="Enter target ID..."
              @keyup.enter="connectToPeer" />
            <button class="btn" @click="connectToPeer" :disabled="!targetPeerId || connectionStatus === 'connecting'">
              {{ connectionStatus === 'connecting' ? '...' : 'Connect' }}
            </button>
          </div>
        </div>
      </section>

      <!-- Connected State -->
      <section v-else class="action-section">
        <div class="connection-info">
          <p>Connected to: <strong>{{ activeConnection?.peer || targetPeerId }}</strong></p>
          <button class="btn btn-secondary btn-sm" @click="disconnect">Disconnect</button>
        </div>

        <div class="sync-actions">
          <button class="btn btn-large sync-btn" @click="manualSync" :disabled="isSyncing">
            <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"
              :class="{ 'spin': isSyncing }">
              <polyline points="23 4 23 10 17 10"></polyline>
              <polyline points="1 20 1 14 7 14"></polyline>
              <path d="M3.51 9a9 9 0 0 1 14.85-3.36L23 10M1 14l4.64 4.36A9 9 0 0 0 20.49 15"></path>
            </svg>
            <span>{{ isSyncing ? 'Syncing...' : 'Sync Clipboard Now' }}</span>
          </button>
          <p class="auto-sync-info">Auto-sync is active. Just copy something and focus this tab to sync automatically to
            the peer.</p>
        </div>

        <!-- Sync Log -->
        <div class="log-container" v-if="logs.length">
          <div class="log-item" v-for="(log, i) in logs" :key="i">
            <span class="log-time">{{ log.time }}</span>
            <span class="log-msg" :class="log.type">{{ log.msg }}</span>
          </div>
        </div>
      </section>
    </div>
  </main>
</template>
