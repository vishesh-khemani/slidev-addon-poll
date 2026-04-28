<script setup>
import { ref, computed } from 'vue'
import MarkdownIt from 'markdown-it'

const md = new MarkdownIt()
import { onSlideEnter, onSlideLeave } from '@slidev/client'

const props = defineProps({
  // Question content
  question: { type: String, required: true },
  type: { type: String, default: 'multiple-choice' },   // 'multiple-choice' | 'open-ended' | 'code'
  choices: { type: [Array, String], default: () => [] }, // array or comma-separated string
  answer: { type: [Number, String], default: 0 },        // index (number) or choice text (string)
  language: { type: String, default: 'python' },         // for code questions
  starterCode: { type: String, default: '' },             // pre-filled code for code questions

  // Server config — defaults read from VITE_POLL_SERVER / VITE_POLL_PASSWORD in .env.local
  server: { type: String, default: () => import.meta.env.VITE_POLL_SERVER ?? 'http://localhost:8000' },
  password: { type: String, default: () => import.meta.env.VITE_POLL_PASSWORD ?? '' },
  username: { type: String, default: () => import.meta.env.VITE_POLL_USERNAME ?? '' },
})

const questionId = ref(null)
const live = ref(false)
const error = ref(null)

const sessionId = ref(null)

const hasCredentials = computed(() => !!props.username && !!props.password)

const normalizedChoices = computed(() =>
  Array.isArray(props.choices)
    ? props.choices
    : props.choices.split(',').map(c => c.trim()).filter(Boolean)
)

const iframeSrc = computed(() => {
  if (!sessionId.value) return props.server
  const hash = new URLSearchParams({ token: sessionId.value, username: props.username, isAdmin: '1' })
  return props.server + '#' + hash.toString()
})

async function apiFetch(method, path, body) {
  const opts = { method, headers: { 'Content-Type': 'application/json' } }
  if (sessionId.value) opts.headers['Authorization'] = 'Bearer ' + sessionId.value
  if (body) opts.body = JSON.stringify(body)
  const res = await fetch(props.server + path, opts)
  return res.json()
}

async function ensureSession() {
  if (sessionId.value) return true
  const data = await apiFetch('POST', '/api/login', { username: props.username, password: props.password })
  if (data.error || !data.isAdmin) { error.value = data.error || 'Admin login failed'; return false }
  sessionId.value = data.sessionId
  return true
}

async function computeFingerprint(body) {
  // Stable-stringify: sort keys so insertion order doesn't affect the hash
  const sorted = Object.fromEntries(Object.entries(body).sort(([a], [b]) => a.localeCompare(b)))
  const str = JSON.stringify(sorted)
  const buf = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(str))
  return Array.from(new Uint8Array(buf)).map(b => b.toString(16).padStart(2, '0')).join('')
}

onSlideEnter(async () => {
  error.value = null
  if (!hasCredentials.value) return
  if (!await ensureSession()) return

  const body = { type: props.type, question: props.question }

  if (props.type === 'multiple-choice') {
    const choices = Array.isArray(props.choices)
      ? props.choices
      : props.choices.split(',').map(c => c.trim())
    const answer = typeof props.answer === 'string'
      ? choices.indexOf(props.answer)
      : Number(props.answer)
    body.choices = choices
    body.answer = answer
  }

  if (props.type === 'code') {
    body.language = props.language
    if (props.starterCode) body.starterCode = props.starterCode
  }

  const fingerprint = await computeFingerprint(body)

  let q = await apiFetch('POST', '/api/admin/question', { ...body, fingerprint, force: true })
  if (q.error) { error.value = q.error; return }
  if (q.hidden) await apiFetch('POST', '/api/admin/question/visibility', { hidden: false })
  questionId.value = q.id
  live.value = true
})

onSlideLeave(async () => {
  live.value = false
  if (hasCredentials.value && sessionId.value) {
    await apiFetch('POST', '/api/admin/question/visibility', { hidden: true })
  }
})
</script>

<template>
  <template v-if="hasCredentials">
    <div class="poll-status">
      <span v-if="error" style="color: red">Poll error: {{ error }}</span>
      <span v-else-if="!live" style="opacity: 0.5">Connecting to poll…</span>
      <span v-else style="opacity: 0.7">Play at https://poll.visheshk.org</span>
    </div>
    <iframe
      v-if="live"
      :src="iframeSrc"
      style="width: 100%; height: 100%; border: none; pointer-events: auto;"
    />
  </template>
  <div v-else class="poll-question-only">
    <div class="poll-question-text" v-html="md.renderInline(question)" />
    <ol v-if="type === 'multiple-choice' && normalizedChoices.length" class="poll-choices">
      <li v-for="(choice, i) in normalizedChoices" :key="i">{{ choice }}</li>
    </ol>
    <pre v-if="type === 'code' && starterCode" class="poll-starter-code">{{ starterCode }}</pre>
  </div>
</template>
