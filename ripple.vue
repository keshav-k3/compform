<script setup lang="ts">
import { Application, Graphics } from 'pixi.js'
import { effectScope, onMounted, onScopeDispose, onUnmounted, ref } from 'vue'
import { useEventListener } from '@vueuse/core'

const el = ref<HTMLDivElement>()

let w = window.innerWidth
let h = window.innerHeight

interface Ripple {
  x: number
  y: number
  radius: number
  maxRadius: number
  alpha: number
  speed: number
  color: number
}

const ripples: Ripple[] = []
const mountedScope = effectScope()

function spawnRipple(x?: number, y?: number) {
  const colors = [0x3b82f6, 0x8b5cf6, 0xec4899, 0x06b6d4, 0x10b981]
  ripples.push({
    x: x ?? Math.random() * w,
    y: y ?? Math.random() * h,
    radius: 0,
    maxRadius: 100 + Math.random() * 200,
    alpha: 0.8,
    speed: 0.5 + Math.random() * 1,
    color: colors[Math.floor(Math.random() * colors.length)]
  })
}

async function setup() {
  if (el.value == null)
    return

  const app = new Application()
  await app.init({
    background: '#ffffff',
    antialias: true,
    resolution: window.devicePixelRatio,
    resizeTo: el.value,
    eventMode: 'none',
    autoDensity: true,
  })
  el.value.appendChild(app.canvas)

  const graphics = new Graphics()
  app.stage.addChild(graphics)

  // Initial ripples
  for (let i = 0; i < 3; i++) {
    setTimeout(() => spawnRipple(), i * 200)
  }

  let lastSpawn = Date.now()
  const spawnInterval = 800

  app.ticker.add((ticker) => {
    const deltaTime = ticker.deltaMS
    const now = Date.now()

    // Spawn new ripples
    if (now - lastSpawn > spawnInterval) {
      if (Math.random() > 0.3) {
        spawnRipple()
        // Sometimes spawn clusters
        if (Math.random() > 0.7) {
          const x = Math.random() * w
          const y = Math.random() * h
          setTimeout(() => spawnRipple(x + (Math.random() - 0.5) * 100, y + (Math.random() - 0.5) * 100), 100)
          setTimeout(() => spawnRipple(x + (Math.random() - 0.5) * 100, y + (Math.random() - 0.5) * 100), 200)
        }
      }
      lastSpawn = now
    }

    graphics.clear()

    // Update and draw ripples
    for (let i = ripples.length - 1; i >= 0; i--) {
      const ripple = ripples[i]
      ripple.radius += ripple.speed * (deltaTime / 16)

      // Fade out as it expands
      ripple.alpha = Math.max(0, 1 - (ripple.radius / ripple.maxRadius))

      // Remove finished ripples
      if (ripple.radius >= ripple.maxRadius) {
        ripples.splice(i, 1)
        continue
      }

      // Draw multiple concentric circles for depth
      for (let j = 0; j < 3; j++) {
        const offsetRadius = ripple.radius - j * 15
        if (offsetRadius > 0) {
          graphics.circle(ripple.x, ripple.y, offsetRadius)
          graphics.stroke({
            width: 2 - j * 0.5,
            color: ripple.color,
            alpha: ripple.alpha * (1 - j * 0.3)
          })
        }
      }
    }
  })

  mountedScope.run(() => {
    useEventListener('resize', () => {
      w = window.innerWidth
      h = window.innerHeight
    })

    onScopeDispose(() => {
      try {
        app?.destroy(true, { children: true, texture: true, textureSource: true })
      }
      catch (error) {
        console.error(error)
      }
    })
  })
}

onMounted(async () => {
  await setup()
})

onUnmounted(() => {
  mountedScope.stop()
})
</script>

<template>
  <div ref="el" z--1 fixed size-screen left-0 right-0 top-0 bottom-0 pointer-events-none dark:invert />
</template>
