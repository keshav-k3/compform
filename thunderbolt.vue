<script setup lang="ts">
import { Application, Graphics } from 'pixi.js'
import { effectScope, onMounted, onScopeDispose, onUnmounted, ref } from 'vue'
import { useEventListener } from '@vueuse/core'

const el = ref<HTMLDivElement>()

let w = window.innerWidth
let h = window.innerHeight

interface LightningBolt {
  segments: { x: number, y: number }[]
  branches: { x: number, y: number }[][]
  alpha: number
  lifetime: number
  maxLifetime: number
  width: number
}

const bolts: LightningBolt[] = []
const mountedScope = effectScope()

function generateLightningBolt(startX: number, startY: number): LightningBolt {
  const segments: { x: number, y: number }[] = [{ x: startX, y: startY }]
  const branches: { x: number, y: number }[][] = []

  // Main bolt - chaotic but mostly vertical
  const numSegments = 8 + Math.floor(Math.random() * 12)
  const targetY = h * (0.6 + Math.random() * 0.4) // Variable length

  let currentX = startX
  let currentY = startY
  let currentAngle = Math.PI / 2 // Start going down

  for (let i = 0; i < numSegments; i++) {
    // Jagged movement - sharp angle changes
    currentAngle += (Math.random() - 0.5) * Math.PI * 0.6

    // Bias towards downward movement
    const downwardBias = 0.3
    if (Math.cos(currentAngle - Math.PI / 2) < 0) {
      currentAngle = currentAngle * (1 - downwardBias) + (Math.PI / 2) * downwardBias
    }

    const stepLength = 25 + Math.random() * 35
    currentX += Math.cos(currentAngle) * stepLength
    currentY += Math.sin(currentAngle) * stepLength

    // Soft bounds - allow some overflow for chaos
    currentX = Math.max(-100, Math.min(w + 100, currentX))

    segments.push({ x: currentX, y: currentY })

    // More frequent branching for chaos
    if (Math.random() > 0.6 && i > 1 && i < numSegments - 1) {
      const branch: { x: number, y: number }[] = [{ x: currentX, y: currentY }]
      const branchAngle = currentAngle + (Math.random() > 0.5 ? 1 : -1) * (Math.PI / 4 + Math.random() * Math.PI / 4)
      const branchSegments = 2 + Math.floor(Math.random() * 4)

      let branchX = currentX
      let branchY = currentY
      let bAngle = branchAngle

      for (let j = 0; j < branchSegments; j++) {
        bAngle += (Math.random() - 0.5) * 0.4
        const bLength = 20 + Math.random() * 25
        branchX += Math.cos(bAngle) * bLength
        branchY += Math.sin(bAngle) * bLength
        branch.push({ x: branchX, y: branchY })

        // Sub-branches for more chaos
        if (Math.random() > 0.7 && j < branchSegments - 1) {
          const subBranch: { x: number, y: number }[] = [{ x: branchX, y: branchY }]
          const subAngle = bAngle + (Math.random() > 0.5 ? 1 : -1) * Math.PI / 3
          const subLength = 15 + Math.random() * 15
          subBranch.push({
            x: branchX + Math.cos(subAngle) * subLength,
            y: branchY + Math.sin(subAngle) * subLength
          })
          branches.push(subBranch)
        }
      }

      branches.push(branch)
    }
  }

  return {
    segments,
    branches,
    alpha: 1,
    lifetime: 0,
    maxLifetime: 200 + Math.random() * 150, // Much faster
    width: 0.8 + Math.random() * 1.2,
  }
}

function spawnRandomBolt() {
  const x = Math.random() * w
  const y = 0 // Always start from top
  bolts.push(generateLightningBolt(x, y))
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

  // Spawn initial bolts
  for (let i = 0; i < 4; i++) {
    setTimeout(() => spawnRandomBolt(), i * 100)
  }

  let lastSpawn = Date.now()
  const minInterval = 400 // Minimum time between spawns
  const maxInterval = 1200 // Maximum time between spawns
  let nextSpawnDelay = minInterval + Math.random() * (maxInterval - minInterval)

  app.ticker.add((ticker) => {
    const deltaTime = ticker.deltaMS
    const now = Date.now()

    // Continuous spawning with variable intervals for chaos
    if (now - lastSpawn > nextSpawnDelay) {
      spawnRandomBolt()
      // Sometimes spawn multiple bolts at once
      if (Math.random() > 0.7) {
        setTimeout(() => spawnRandomBolt(), 50 + Math.random() * 100)
      }
      if (Math.random() > 0.85) {
        setTimeout(() => spawnRandomBolt(), 100 + Math.random() * 150)
      }
      lastSpawn = now
      nextSpawnDelay = minInterval + Math.random() * (maxInterval - minInterval)
    }

    graphics.clear()

    // Update and draw bolts
    for (let i = bolts.length - 1; i >= 0; i--) {
      const bolt = bolts[i]
      bolt.lifetime += deltaTime

      // Very fast flash
      if (bolt.lifetime < 30) {
        bolt.alpha = bolt.lifetime / 30
      } else if (bolt.lifetime > bolt.maxLifetime - 100) {
        bolt.alpha = (bolt.maxLifetime - bolt.lifetime) / 100
      } else {
        // Intense flicker
        bolt.alpha = 0.5 + Math.random() * 0.5
      }

      // Remove dead bolts
      if (bolt.lifetime > bolt.maxLifetime) {
        bolts.splice(i, 1)
        continue
      }

      // Helper function to draw a lightning path
      const drawPath = (path: { x: number, y: number }[], width: number, isMain = false) => {
        if (path.length < 2) return

        const baseAlpha = isMain ? bolt.alpha : bolt.alpha * 0.7

        // Wide outer glow
        graphics.moveTo(path[0].x, path[0].y)
        for (let j = 1; j < path.length; j++) {
          graphics.lineTo(path[j].x, path[j].y)
        }
        graphics.stroke({
          width: width * 6,
          color: 0x6366f1,
          alpha: baseAlpha * 0.1
        })

        // Medium glow
        graphics.moveTo(path[0].x, path[0].y)
        for (let j = 1; j < path.length; j++) {
          graphics.lineTo(path[j].x, path[j].y)
        }
        graphics.stroke({
          width: width * 3,
          color: 0x818cf8,
          alpha: baseAlpha * 0.3
        })

        // Inner bright
        graphics.moveTo(path[0].x, path[0].y)
        for (let j = 1; j < path.length; j++) {
          graphics.lineTo(path[j].x, path[j].y)
        }
        graphics.stroke({
          width: width * 1.5,
          color: 0xc7d2fe,
          alpha: baseAlpha * 0.6
        })

        // Core
        graphics.moveTo(path[0].x, path[0].y)
        for (let j = 1; j < path.length; j++) {
          graphics.lineTo(path[j].x, path[j].y)
        }
        graphics.stroke({
          width: width * 0.5,
          color: 0xffffff,
          alpha: baseAlpha
        })
      }

      // Draw main bolt
      drawPath(bolt.segments, bolt.width, true)

      // Draw branches
      for (const branch of bolt.branches) {
        drawPath(branch, bolt.width * 0.5, false)
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
