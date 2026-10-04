<script setup>
import { computed, nextTick, onMounted, ref, watch } from 'vue'
import { normalRanges } from '../data'

const props = defineProps({ records: { type: Array, default: () => [] }, type: { type: String, default: 'weight' } })
const canvas = ref(null)
const range = computed(() => normalRanges[props.type])

function draw() {
  const element = canvas.value
  if (!element) return
  const rect = element.getBoundingClientRect()
  const ratio = window.devicePixelRatio || 1
  element.width = rect.width * ratio
  element.height = rect.height * ratio
  const ctx = element.getContext('2d')
  ctx.scale(ratio, ratio)
  const width = rect.width
  const height = rect.height
  ctx.clearRect(0, 0, width, height)
  ctx.font = '11px Inter, sans-serif'
  ctx.strokeStyle = '#e7ebe4'
  ctx.lineWidth = 1
  ;[0.25, 0.5, 0.75].forEach((step) => {
    ctx.beginPath(); ctx.moveTo(0, height * step); ctx.lineTo(width, height * step); ctx.stroke()
  })
  const span = range.value.max - range.value.min
  // 空值（''、null、空白、非数字）跳过；0 是合法记录，正常参与
  const points = props.records.map((item, index) => {
    const raw = item.value
    const empty = raw === null || raw === undefined || (typeof raw === 'string' && raw.trim() === '')
    const value = empty ? NaN : Number(raw)
    return { index, value, valid: Number.isFinite(value) }
  })
  // 超出正常范围上下 2 倍量程的视为极端值：不参与坐标计算，裁剪到边缘并标注
  const outlierMin = range.value.min - span * 2
  const outlierMax = range.value.max + span * 2
  const isOutlier = (point) => point.valid && (point.value < outlierMin || point.value > outlierMax)
  const normalValues = points.filter((point) => point.valid && !isOutlier(point)).map((point) => point.value)
  // 坐标范围始终包含完整正常范围，参考带保持可见
  const min = Math.min(...normalValues, range.value.min) - span * 0.2
  const max = Math.max(...normalValues, range.value.max) + span * 0.2
  const x = (index) => points.length === 1 ? width / 2 : (index / (points.length - 1)) * (width - 24) + 12
  const y = (value) => height - ((value - min) / (max - min)) * (height - 24) - 12
  const clampY = (value) => Math.min(Math.max(y(value), 12), height - 12)
  ctx.fillStyle = '#f0f7dd'
  ctx.fillRect(0, Math.max(0, y(range.value.max)), width, Math.max(0, y(range.value.min) - y(range.value.max)))
  ctx.beginPath()
  let penDown = false
  points.forEach((point) => {
    if (!point.valid) { penDown = false; return }
    if (penDown) ctx.lineTo(x(point.index), clampY(point.value))
    else ctx.moveTo(x(point.index), clampY(point.value))
    penDown = true
  })
  ctx.strokeStyle = '#b5cf3e'
  ctx.lineWidth = 2.5
  ctx.stroke()
  points.forEach((point) => {
    if (!point.valid) return
    const outlier = isOutlier(point)
    const px = x(point.index)
    const py = clampY(point.value)
    ctx.beginPath(); ctx.arc(px, py, 4, 0, Math.PI * 2); ctx.fillStyle = '#ffffff'; ctx.fill(); ctx.strokeStyle = outlier ? '#dfa172' : '#87a52f'; ctx.lineWidth = 2; ctx.stroke()
    if (outlier) {
      const above = point.value > outlierMax
      ctx.fillStyle = '#c58661'
      ctx.textAlign = 'center'
      ctx.fillText(`${above ? '↑' : '↓'}${point.value}`, Math.min(Math.max(px, 24), width - 24), above ? py + 16 : py - 8)
      ctx.textAlign = 'start'
    }
  })
  ctx.fillStyle = '#8b9686'
  ctx.fillText(`${range.value.max}`, 6, Math.max(12, y(range.value.max) - 6))
  ctx.fillText(`${range.value.min}`, 6, Math.min(height - 4, y(range.value.min) + 13))
}

watch(() => [props.records, props.type], () => nextTick(draw), { deep: true })
onMounted(() => { draw(); window.addEventListener('resize', draw) })
</script>

<template>
  <div class="trend-chart">
    <canvas ref="canvas"></canvas>
    <div v-if="!records.length" class="chart-empty">还没有记录，添加一次健康数据吧</div>
  </div>
</template>
