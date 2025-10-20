<script setup lang='ts'>
import type { Fn } from '@vueuse/core'
import { computed, onMounted, reactive, ref } from 'vue'
import { useRafFn, useWindowSize } from '@vueuse/core'

const r90 = Math.PI / 2
const r45 = Math.PI / 4
const color = '#10b98120'

const el = ref<HTMLCanvasElement | null>(null)

const { random } = Math
const size = reactive(useWindowSize())

const start = ref<Fn>(() => {})
const len = ref(4)
const stopped = ref(false)

function initCanvas(canvas: HTMLCanvasElement, width = 400, height = 400, _dpi?: number) {
  const ctx = canvas.getContext('2d')!

  const dpr = window.devicePixelRatio || 1
  // @ts-expect-error vendor
  const bsr = ctx.webkitBackingStorePixelRatio || ctx.mozBackingStorePixelRatio || ctx.msBackingStorePixelRatio || ctx.oBackingStorePixelRatio || ctx.backingStorePixelRatio || 1

  const dpi = _dpi || dpr / bsr

  canvas.style.width = `${width}px`
  canvas.style.height = `${height}px`
  canvas.width = dpi * width
  canvas.height = dpi * height
  ctx.scale(dpi, dpi)

  return { ctx, dpi }
}

function polar2cart(x = 0, y = 0, r = 0, theta = 0) {
  const dx = r * Math.cos(theta)
  const dy = r * Math.sin(theta)
  return [x + dx, y + dy]
}

onMounted(async () => {
  const canvas = el.value!
  const { ctx } = initCanvas(canvas, size.width, size.height)
  const { width, height } = canvas

  let steps: Fn[] = []
  let prevSteps: Fn[] = []

  const step = (x: number, y: number, rad: number, curveDir: number = 1, segmentCount: number = 0) => {
    const length = random() * len.value + 3
    segmentCount += 1

    // Add curve to the angle for organic vine-like growth
    const curve = (random() - 0.5) * 0.3 * curveDir
    const newRad = rad + curve

    const [nx, ny] = polar2cart(x, y, length, newRad)

    ctx.beginPath()
    ctx.moveTo(x, y)
    ctx.lineTo(nx, ny)
    ctx.stroke()

    // Draw leaves occasionally
    if (random() > 0.85 && segmentCount > 5) {
      const leafAngle = newRad + (random() > 0.5 ? r45 : -r45)
      const leafLength = 5 + random() * 8
      const [lx, ly] = polar2cart(nx, ny, leafLength, leafAngle)

      ctx.beginPath()
      ctx.moveTo(nx, ny)
      ctx.lineTo(lx, ly)
      ctx.strokeStyle = '#10b98130'
      ctx.lineWidth = 0.8
      ctx.stroke()
      ctx.strokeStyle = color
      ctx.lineWidth = 1.2
    }

    // Out of bounds
    if (nx < -100 || nx > size.width + 100 || ny < -100 || ny > size.height + 100)
      return

    if (segmentCount > 60)
      return

    // Continue growing
    if (random() < 0.88)
      steps.push(() => step(nx, ny, newRad, curveDir, segmentCount))

    // Branch off occasionally
    if (random() > 0.94 && segmentCount > 10) {
      const branchAngle = newRad + (random() > 0.5 ? r45 : -r45)
      const newCurveDir = random() > 0.5 ? 1 : -1
      steps.push(() => step(nx, ny, branchAngle, newCurveDir, 0))
    }
  }

  let lastTime = performance.now()
  const interval = 1000 / 60

  let controls: ReturnType<typeof useRafFn>

  const frame = () => {
    if (performance.now() - lastTime < interval)
      return

    prevSteps = steps
    steps = []
    lastTime = performance.now()

    // Restart when complete for continuous animation
    if (!prevSteps.length) {
      setTimeout(() => {
        start.value()
      }, 2000) // Wait 2 seconds before restarting
      return
    }

    prevSteps.forEach((i) => {
      if (random() < 0.7)
        steps.push(i)
      else
        i()
    })
  }

  controls = useRafFn(frame, { immediate: false })

  const randomMiddle = () => random() * 0.6 + 0.2

  start.value = () => {
    controls.pause()
    ctx.clearRect(0, 0, width, height)
    ctx.lineWidth = 1.2
    ctx.strokeStyle = color
    prevSteps = []

    // Start vines from edges, growing inward with curves
    steps = [
      () => step(randomMiddle() * size.width, -5, r90, random() > 0.5 ? 1 : -1),
      () => step(randomMiddle() * size.width, size.height + 5, -r90, random() > 0.5 ? 1 : -1),
      () => step(-5, randomMiddle() * size.height, 0, random() > 0.5 ? 1 : -1),
      () => step(size.width + 5, randomMiddle() * size.height, Math.PI, random() > 0.5 ? 1 : -1),
    ]

    if (size.width < 500)
      steps = steps.slice(0, 2)

    controls.resume()
    stopped.value = false
  }

  start.value()
})

const mask = computed(() => 'radial-gradient(circle, transparent, black);')
</script>

<template>
  <div
    class="fixed top-0 bottom-0 left-0 right-0 pointer-events-none print:hidden"
    style="z-index: -1"
    :style="`mask-image: ${mask};--webkit-mask-image: ${mask};`"
  >
    <canvas ref="el" width="400" height="400" />
  </div>
</template>
