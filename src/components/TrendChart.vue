<script setup>
import { computed, nextTick, onMounted, onUnmounted, ref, watch } from 'vue'
import { normalRanges } from '../data'

const props = defineProps({ records: { type: Array, default: () => [] }, type: { type: String, default: 'weight' } })
const canvas = ref(null)
const range = computed(() => normalRanges[props.type])

function parseMetricValue(value) {
  if (value === null || value === undefined) return NaN
  if (typeof value === 'string' && value.trim() === '') return NaN
  if (typeof value !== 'number' && typeof value !== 'string') return NaN

  const number = Number(value)
  return Number.isFinite(number) && number >= 0 ? number : NaN
}

function buildChartData(records, normalRange) {
  const normalMin = Number(normalRange.min)
  const normalMax = Number(normalRange.max)
  const span = Math.max(normalMax - normalMin, 1)
  const focusMin = normalMin - span * 0.25
  const focusMax = normalMax + span * 0.25

  // 超过正常范围上沿两个跨度的值视为需要裁剪的超大异常值。
  // 记录本身仍保留在图上，只把坐标限制在聚焦区域上沿并标注真实值。
  const highCutoff = normalMax + span * 2

  const points = records.map((record, index) => {
    const value = parseMetricValue(record.value)
    const valid = Number.isFinite(value)
    const outlier = valid && value > highCutoff

    return { record, index, value, valid, outlier }
  })

  let viewMin = focusMin
  let viewMax = focusMax

  points.forEach((point) => {
    if (!point.valid || point.outlier) return
    // 真正记录下来的 0 也必须参与坐标计算；空字符串、null、undefined 不会被转换成 0。
    viewMin = Math.min(viewMin, point.value)
    viewMax = Math.max(viewMax, point.value)
  })

  if (!(viewMax > viewMin)) {
    viewMin -= 1
    viewMax += 1
  }

  return { points, viewMin, viewMax }
}

function formatValue(value) {
  return Number.isInteger(value) ? String(value) : value.toFixed(1)
}

function drawOutlierTag(ctx, text, x, y, width) {
  ctx.font = '10px Inter, sans-serif'
  const tagWidth = ctx.measureText(text).width + 10
  const left = Math.min(Math.max(0, x - tagWidth / 2), width - tagWidth)

  ctx.fillStyle = 'rgba(255, 248, 242, .94)'
  ctx.strokeStyle = '#e4b59a'
  ctx.lineWidth = 1
  ctx.fillRect(left, y, tagWidth, 16)
  ctx.strokeRect(left, y, tagWidth, 16)
  ctx.fillStyle = '#bd6f43'
  ctx.fillText(text, left + 5, y + 12)
}

function draw() {
  const element = canvas.value
  if (!element) return
  const normalRange = range.value
  if (!normalRange) return

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

  const { points, viewMin, viewMax } = buildChartData(props.records, normalRange)
  const hasOutlier = points.some((point) => point.outlier)
  const plotTop = hasOutlier ? 40 : 22
  const plotBottom = height - 16
  const plotHeight = plotBottom - plotTop

  ctx.strokeStyle = '#e7ebe4'
  ctx.lineWidth = 1
  ;[0.25, 0.5, 0.75].forEach((step) => {
    const y = plotTop + plotHeight * step
    ctx.beginPath()
    ctx.moveTo(0, y)
    ctx.lineTo(width, y)
    ctx.stroke()
  })

  const x = (index) => {
    if (points.length === 1) return width / 2
    return (index / (points.length - 1)) * (width - 24) + 12
  }
  const y = (value) => {
    const clipped = Math.max(viewMin, Math.min(viewMax, value))
    return plotBottom - ((clipped - viewMin) / (viewMax - viewMin)) * plotHeight
  }

  // 正常参考带先绘制，即使没有有效数值或存在异常值也始终可见。
  const bandTop = Math.max(plotTop, y(normalRange.max))
  const bandBottom = Math.min(plotBottom, y(normalRange.min))
  ctx.fillStyle = '#f0f7dd'
  ctx.fillRect(0, bandTop, width, Math.max(0, bandBottom - bandTop))

  ctx.fillStyle = '#8b9686'
  ctx.fillText(`${normalRange.max}`, 6, Math.max(plotTop + 10, y(normalRange.max) - 5))
  ctx.fillText(`${normalRange.min}`, 6, Math.min(plotBottom - 4, y(normalRange.min) + 13))

  const validPoints = points.filter((point) => point.valid)
  if (!validPoints.length) {
    ctx.fillStyle = '#a4ada0'
    ctx.font = '11px Inter, sans-serif'
    ctx.textAlign = 'center'
    ctx.fillText('暂无可绘制的有效数值', width / 2, height / 2)
    ctx.textAlign = 'left'
    return
  }

  ctx.save()
  ctx.beginPath()
  ctx.rect(0, plotTop, width, plotHeight)
  ctx.clip()

  ctx.beginPath()
  let penDown = false
  points.forEach((point) => {
    if (!point.valid) {
      penDown = false
      return
    }

    const pointX = x(point.index)
    const pointY = y(point.value)
    if (penDown && points[point.index - 1]?.valid) ctx.lineTo(pointX, pointY)
    else ctx.moveTo(pointX, pointY)
    penDown = true
  })
  ctx.strokeStyle = '#b5cf3e'
  ctx.lineWidth = 2.5
  ctx.stroke()
  ctx.restore()

  points.forEach((point) => {
    if (!point.valid) return

    const pointX = x(point.index)
    const pointY = y(point.outlier ? viewMax : point.value)
    ctx.beginPath()

    if (point.outlier) {
      ctx.moveTo(pointX, pointY - 5)
      ctx.lineTo(pointX + 5, pointY)
      ctx.lineTo(pointX, pointY + 5)
      ctx.lineTo(pointX - 5, pointY)
      ctx.closePath()
      ctx.fillStyle = '#fff4e8'
      ctx.fill()
      ctx.strokeStyle = '#d08355'
      ctx.lineWidth = 2
      ctx.stroke()
      return
    }

    ctx.arc(pointX, pointY, 4, 0, Math.PI * 2)
    ctx.fillStyle = '#ffffff'
    ctx.fill()
    ctx.strokeStyle = '#87a52f'
    ctx.lineWidth = 2
    ctx.stroke()
  })

  points.forEach((point) => {
    if (!point.outlier) return

    const arrow = '↑'
    const text = `${arrow} ${formatValue(point.value)} ${normalRange.unit}`
    const tagY = point.index % 2 === 0 ? 2 : 19
    drawOutlierTag(ctx, text, x(point.index), tagY, width)
  })
}

watch(() => [props.records, props.type], () => nextTick(draw), { deep: true })
onMounted(() => {
  draw()
  window.addEventListener('resize', draw)
})
onUnmounted(() => window.removeEventListener('resize', draw))
</script>

<template>
  <div class="trend-chart">
    <canvas ref="canvas"></canvas>
    <div v-if="!records.length" class="chart-empty">还没有记录，添加一次健康数据吧</div>
  </div>
</template>
