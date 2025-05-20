<template>
  <div>
    <canvas ref="canvas" :width="width" :height="height" />
    <div class="buttons">
      <button @click="clearCanvas">Clear</button>
      <button @click="uploadImage" :disabled="loading">
        {{ loading ? 'Uploading...' : 'Upload & Predict' }}
      </button>
    </div>

    <div v-if="previewUrl" class="preview">
      <h3>Preview:</h3>
      <img :src="previewUrl" alt="Canvas preview" />
    </div>

    <div v-if="result" class="result">
      <h3>Prediction:</h3>
      <p>Digit: <strong>{{ result.predicted_digit }}</strong></p>
      <p>Confidence: <strong>{{ formatConfidence(result.confidence) }}</strong></p>
    </div>
  </div>
</template>

<style scoped>
canvas {
  border: 1px solid #ccc;
  cursor: crosshair;
  display: block;
  margin-bottom: 1rem;
  background-color: black;
}
.buttons {
  display: flex;
  gap: 1rem;
  margin-bottom: 1rem;
}
.preview img {
  border: 1px solid #aaa;
  max-width: 15%;
  height: auto;
}
.result {
  margin-top: 1rem;
  padding: 0.75rem;
  border: 1px solid #ddd;
  background: #f7f7f7;
  max-width: 400px;
}
</style>

<script setup>
import { ref, onMounted } from 'vue'

const canvas = ref(null)
const previewUrl = ref(null)
const result = ref(null)
const loading = ref(false)

let ctx = null
let drawing = false

const props = defineProps({
  width: { type: Number, default: 600 },
  height: { type: Number, default: 400 },
  endpoint: { type: String, required: true }
})

onMounted(() => {
  const el = canvas.value
  ctx = el.getContext('2d')

  // Set canvas background to black
  ctx.fillStyle = 'black'
  ctx.fillRect(0, 0, props.width, props.height)

  el.addEventListener('mousedown', startDraw)
  el.addEventListener('mouseup', stopDraw)
  el.addEventListener('mousemove', draw)
})

function startDraw(e) {
  drawing = true
  draw(e)
}

function stopDraw() {
  drawing = false
  ctx.beginPath()
}

function draw(e) {
  if (!drawing) return
  const rect = canvas.value.getBoundingClientRect()
  const x = e.clientX - rect.left
  const y = e.clientY - rect.top

  ctx.lineWidth = 12
  ctx.lineCap = 'round'
  ctx.strokeStyle = 'white'
  ctx.lineTo(x, y)
  ctx.stroke()
  ctx.beginPath()
  ctx.moveTo(x, y)
}

function clearCanvas() {
  ctx.fillStyle = 'black'
  ctx.fillRect(0, 0, props.width, props.height)
  previewUrl.value = null
  result.value = null
}

async function uploadImage() {
  canvas.value.toBlob(async (blob) => {
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
      console.log(res)

      const contentType = res.headers.get('content-type') || ''
      if (!res.ok || !contentType.includes('application/json')) {
        const text = await res.text()
        throw new Error(`Invalid response: ${text}`)
      }

      const data = await res.json()

      if (
        typeof data.predicted_digit !== 'undefined' &&
        typeof data.confidence !== 'undefined'
      ) {
        result.value = data
      } else {
        throw new Error('Missing expected keys in response')
      }
    } catch (err) {
      console.error('Upload failed:', err)
      result.value = {
        predicted_digit: '?',
        confidence: 0,
        error: true
      }
    } finally {
      loading.value = false
    }
  }, 'image/png')
}

function formatConfidence(confidence) {
  return `${(confidence * 100).toFixed(2)}%`
}
</script>
