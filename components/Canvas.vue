<template>
  <div class="canvas-wrapper">
    <canvas ref="canvas" :width="size" :height="size" />
    
    <div class="buttons">
      <button class="btn clear" @click="clearCanvas">🧹 Clear</button>
      <button class="btn upload" @click="uploadImage" :disabled="loading">
        {{ loading ? 'Uploading...' : '📤 Upload & Predict' }}
      </button>
    </div>

    <div v-if="previewUrl" class="preview">
      <h3>Preview</h3>
      <img :src="previewUrl" alt="Canvas preview" />
    </div>

    <div v-if="result" class="result">
      <h3>Prediction</h3>
      <p>Character: <strong>{{ result.predicted_character }}</strong></p>
      <p>Confidence: <strong>{{ formatConfidence(result.confidence) }}</strong></p>
    </div>
  </div>
</template>

<style scoped>
.canvas-wrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1rem;
}

canvas {
  border: 2px solid #888;
  background-color: black;
  cursor: crosshair;
  border-radius: 8px;
}

.buttons {
  display: flex;
  gap: 1rem;
}

.btn {
  padding: 0.5rem 1.2rem;
  font-size: 1rem;
  font-weight: bold;
  border-radius: 6px;
  cursor: pointer;
  transition: background 0.2s ease;
  border: none;
}

.btn.clear {
  background-color: #eee;
  color: #333;
}
.btn.clear:hover {
  background-color: #ddd;
}

.btn.upload {
  background-color: #1e88e5;
  color: white;
}
.btn.upload:hover {
  background-color: #1565c0;
}

.preview img {
  max-width: 150px;
  border: 1px solid #aaa;
  border-radius: 4px;
}

.result {
  background-color: #f3f3f3;
  padding: 0.75rem 1rem;
  border: 1px solid #ddd;
  border-radius: 8px;
  text-align: center;
}
</style>

<script setup lang="ts">
import { ref, onMounted } from 'vue'

const canvas = ref<HTMLCanvasElement | null>(null)
const previewUrl = ref<string | null>(null)
const result = ref<{ predicted_character: number | string; confidence: number; error?: boolean } | null>(null)
const loading = ref(false)

const size = 300 // canvas size: 300x300

const props = defineProps<{
  endpoint: string
}>()

let ctx: CanvasRenderingContext2D | null = null
let drawing = false

onMounted(() => {
  const el = canvas.value
  if (!el) return

  ctx = el.getContext('2d')
  if (!ctx) return

  ctx.fillStyle = 'black'
  ctx.fillRect(0, 0, size, size)

  el.addEventListener('mousedown', startDraw)
  el.addEventListener('mouseup', stopDraw)
  el.addEventListener('mousemove', draw)
})

function startDraw(e: MouseEvent) {
  drawing = true
  draw(e)
}

function stopDraw() {
  drawing = false
  ctx?.beginPath()
}

function draw(e: MouseEvent) {
  if (!drawing || !ctx || !canvas.value) return

  const rect = canvas.value.getBoundingClientRect()
  const x = e.clientX - rect.left
  const y = e.clientY - rect.top

  ctx.lineWidth = 16
  ctx.lineCap = 'round'
  ctx.strokeStyle = 'white'
  ctx.lineTo(x, y)
  ctx.stroke()
  ctx.beginPath()
  ctx.moveTo(x, y)
}

function clearCanvas() {
  if (!ctx) return
  ctx.fillStyle = 'black'
  ctx.fillRect(0, 0, size, size)
  previewUrl.value = null
  result.value = null
}

function uploadImage() {
  const el = canvas.value
  if (!el) return

  el.toBlob(async (blob) => {
    if (!blob) return

    previewUrl.value = URL.createObjectURL(blob)
    loading.value = true
    result.value = null

    const formData = new FormData()
    formData.append('image', blob, 'drawing.png')

    try {
      const res = await fetch(props.endpoint, {
        method: 'POST',
        body: formData,
      })

      const contentType = res.headers.get('content-type') || ''
      if (!res.ok || !contentType.includes('application/json')) {
        const text = await res.text()
        throw new Error(`Invalid response: ${text}`)
      }

      const data = await res.json()

      console.log('Response data:', data)

      if (
        typeof data.predicted_character !== 'undefined' &&
        typeof data.confidence !== 'undefined'
      ) {
        result.value = {
          predicted_character: data.predicted_character,
          confidence: data.confidence,
        }
      } else {
        throw new Error('Missing expected keys in response')
      }
    } catch (err) {
      console.error('Upload failed:', err)
      result.value = {
        predicted_character: '?',
        confidence: 0,
        error: true,
      }
    } finally {
      loading.value = false
    }
  }, 'image/png')
}

function formatConfidence(confidence: number): string {
  return `${(confidence * 100).toFixed(2)}%`
}
</script>
