<script setup lang='ts'>
import type { Fn } from '@vueuse/core'
import { computed, onMounted, reactive, ref } from 'vue'
import { useRafFn, useWindowSize } from '@vueuse/core'

const r90 = Math.PI / 2
const r15 = Math.PI / 12
const r30 = Math.PI / 6
const colors = ['#ec489918', '#f97316a18', '#f59e0b18', '#eab30818']

const el = ref<HTMLCanvasElement | null>(null)

const { random } = Math
const size = reactive(useWindowSize())

const start = ref<Fn>(() => {})
const MIN_BRANCH = 20
const len = ref(3)
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

  const step = (x: number, y: number, rad: number, thickness: number = 2.5, counter: { value: number } = { value: 0 }) => {
    const length = random() * len.value + 2
    counter.value += 1

    const [nx, ny] = polar2cart(x, y, length, rad)

    // Gradual thinning as branches extend
    const newThickness = Math.max(0.3, thickness * 0.95)

    // Color variation
    const colorIndex = Math.min(Math.floor((2.5 - thickness) / 0.6), colors.length - 1)
    ctx.strokeStyle = colors[Math.max(0, colorIndex)]
    ctx.lineWidth = thickness

    ctx.beginPath()
    ctx.moveTo(x, y)
    ctx.lineTo(nx, ny)
    ctx.stroke()

    // Draw coral polyps (dots) at branch tips occasionally
    if (random() > 0.9 && thickness < 1) {
      ctx.beginPath()
      ctx.arc(nx, ny, random() * 2 + 1, 0, Math.PI * 2)
      ctx.fillStyle = colors[colorIndex].replace('18', '35')
      ctx.fill()
    }

    // Out of bounds
    if (nx < -100 || nx > size.width + 100 || ny < -100 || ny > size.height + 100)
      return

    if (thickness < 0.4)
      return

    const rate = counter.value <= MIN_BRANCH ? 0.85 : 0.6

    // Create multiple branches for coral-like structure
    const numBranches = random() > 0.7 ? 3 : 2
    const baseAngleSpread = random() * r30 + r15

    for (let i = 0; i < numBranches; i++) {
      if (random() < rate) {
        const branchAngle = rad + (random() - 0.5) * baseAngleSpread
        steps.push(() => step(nx, ny, branchAngle, newThickness, counter))
      }
    }
  }

  let lastTime = performance.now()
  const interval = 1000 / 45

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
      if (random() < 0.55)
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
    prevSteps = []

    // Coral grows upward and outward from bottom
    const startPoints = []
    const numStarts = size.width < 500 ? 2 : 4

    for (let i = 0; i < numStarts; i++) {
      startPoints.push(
        () => step(random() * size.width, size.height + 5, -r90 + (random() - 0.5) * r30, 2.5)
      )
    }

    // Add some from sides for variety
    if (size.width >= 500) {
      startPoints.push(
        () => step(-5, randomMiddle() * size.height, random() * r30, 2.5),
        () => step(size.width + 5, randomMiddle() * size.height, Math.PI - random() * r30, 2.5)
      )
    }

    steps = startPoints
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
