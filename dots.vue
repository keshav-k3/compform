<script setup lang="ts">
import type { Texture } from 'pixi.js'
import { Application, Graphics, Particle, ParticleContainer } from 'pixi.js'
import { createNoise3D } from 'simplex-noise'
import { effectScope, onMounted, onScopeDispose, onUnmounted, ref } from 'vue'
import { useEventListener } from '@vueuse/core'

const el = ref<HTMLDivElement>()

let w = window.innerWidth
let h = window.innerHeight

const SCALE = 150
const SPACING = 20
const WAVE_INTENSITY = 30
const ROTATION_SPEED = 0.3

const noise3d = createNoise3D()

const existingPoints = new Set<string>()
const points: {
  x: number
  y: number
  baseOpacity: number
  particle: Particle
  phase: number
}[] = []

function getWaveValue(x: number, y: number, z: number) {
  return noise3d(x / SCALE, y / SCALE, z)
}

const mountedScope = effectScope()

function createRingTexture(app: Application) {
  const g = new Graphics()
  g.circle(0, 0, 2.5).fill(0xCCCCCC)
  g.circle(0, 0, 1.5).fill(0xFFFFFF)
  return app.renderer.generateTexture(g)
}

function addPoints({ ringTexture, particleContainer }: { ringTexture: Texture, particleContainer: ParticleContainer }) {
  for (let x = -SPACING; x < w + SPACING; x += SPACING) {
    for (let y = -SPACING; y < h + SPACING; y += SPACING) {
      const id = `${x}-${y}`
      if (existingPoints.has(id))
        continue
      existingPoints.add(id)

      const particle = new Particle(ringTexture)
      particle.anchorX = 0.5
      particle.anchorY = 0.5
      particleContainer.addParticle(particle)

      const baseOpacity = Math.random() * 0.3 + 0.7
      const phase = Math.random() * Math.PI * 2
      points.push({ x, y, baseOpacity, particle, phase })
    }
  }
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

  const particleContainer = new ParticleContainer({
    dynamicProperties: { position: true, alpha: true, scale: true, rotation: true }
  })
  app.stage.addChild(particleContainer)

  const ringTexture = createRingTexture(app)
  addPoints({ ringTexture, particleContainer })

  app.ticker.add(() => {
    const t = Date.now() / 8000

    for (const p of points) {
      const { x, y, baseOpacity, particle, phase } = p

      // Wave motion in multiple directions
      const wave1 = getWaveValue(x, y, t)
      const wave2 = getWaveValue(x + 1000, y, t * 1.3)

      // Combined wave displacement
      const dx = Math.sin(wave1 * Math.PI * 2) * WAVE_INTENSITY
      const dy = Math.cos(wave2 * Math.PI * 2) * WAVE_INTENSITY

      particle.x = x + dx
      particle.y = y + dy

      // Pulsing scale effect
      const scaleFactor = 0.8 + Math.sin(t * 2 + phase) * 0.3
      particle.scaleX = scaleFactor
      particle.scaleY = scaleFactor

      // Gentle rotation
      particle.rotation = (wave1 + wave2) * ROTATION_SPEED

      // Dynamic opacity based on wave values
      const opacityMod = (wave1 + 1) * 0.5 // normalize to 0-1
      particle.alpha = baseOpacity * (0.4 + opacityMod * 0.6)
    }
  })

  mountedScope.run(() => {
    useEventListener('resize', () => {
      w = window.innerWidth
      h = window.innerHeight
      addPoints({ ringTexture, particleContainer })
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
