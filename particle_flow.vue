<script setup lang="ts">
import type { Texture } from 'pixi.js'
import { Application, Graphics, Particle, ParticleContainer } from 'pixi.js'
import { createNoise3D } from 'simplex-noise'
import { effectScope, onMounted, onScopeDispose, onUnmounted, ref } from 'vue'
import { useEventListener } from '@vueuse/core'

const el = ref<HTMLDivElement>()

let w = window.innerWidth
let h = window.innerHeight

const noise3d = createNoise3D()

interface FlowParticle {
  x: number
  y: number
  vx: number
  vy: number
  particle: Particle
  life: number
  maxLife: number
  hue: number
  size: number
}

const particles: FlowParticle[] = []
const mountedScope = effectScope()

const FLOW_SCALE = 150
const PARTICLE_COUNT = 200

function createParticleTexture(app: Application) {
  const g = new Graphics().circle(0, 0, 1.5).fill(0xffffff)
  return app.renderer.generateTexture(g)
}

function getFlowField(x: number, y: number, z: number) {
  return (noise3d(x / FLOW_SCALE, y / FLOW_SCALE, z) - 0.5) * 2 * Math.PI
}

function createParticle(particleTexture: Texture, particleContainer: ParticleContainer) {
  const particle = new Particle(particleTexture)
  particle.anchorX = 0.5
  particle.anchorY = 0.5
  particleContainer.addParticle(particle)

  const colors = [0x3b82f6, 0x8b5cf6, 0xec4899, 0xf59e0b, 0x10b981, 0x06b6d4]

  particles.push({
    x: Math.random() * w,
    y: Math.random() * h,
    vx: 0,
    vy: 0,
    particle,
    life: 0,
    maxLife: 200 + Math.random() * 200,
    hue: colors[Math.floor(Math.random() * colors.length)],
    size: 0.5 + Math.random() * 1
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

  const particleContainer = new ParticleContainer({
    dynamicProperties: { position: true, alpha: true, scale: true, tint: true }
  })
  app.stage.addChild(particleContainer)

  const trailGraphics = new Graphics()
  app.stage.addChild(trailGraphics)

  const particleTexture = createParticleTexture(app)

  // Initialize particles
  for (let i = 0; i < PARTICLE_COUNT; i++) {
    createParticle(particleTexture, particleContainer)
  }

  const trails = new Map<FlowParticle, { x: number, y: number }[]>()

  app.ticker.add((ticker) => {
    const t = Date.now() / 5000
    const deltaTime = ticker.deltaMS / 16

    trailGraphics.clear()

    for (let i = particles.length - 1; i >= 0; i--) {
      const p = particles[i]

      // Get flow field direction
      const angle = getFlowField(p.x, p.y, t)
      const force = 0.5

      p.vx += Math.cos(angle) * force * deltaTime
      p.vy += Math.sin(angle) * force * deltaTime

      // Apply damping
      p.vx *= 0.95
      p.vy *= 0.95

      // Update position
      p.x += p.vx * deltaTime
      p.y += p.vy * deltaTime

      p.life += deltaTime

      // Wrap around edges
      if (p.x < -50) p.x = w + 50
      if (p.x > w + 50) p.x = -50
      if (p.y < -50) p.y = h + 50
      if (p.y > h + 50) p.y = -50

      // Update trail
      if (!trails.has(p)) {
        trails.set(p, [])
      }
      const trail = trails.get(p)!
      trail.push({ x: p.x, y: p.y })
      if (trail.length > 15) {
        trail.shift()
      }

      // Life cycle alpha
      let alpha = 1
      if (p.life < 20) {
        alpha = p.life / 20
      } else if (p.life > p.maxLife - 20) {
        alpha = (p.maxLife - p.life) / 20
      }

      // Update particle
      p.particle.x = p.x
      p.particle.y = p.y
      p.particle.alpha = alpha * 0.8
      p.particle.scaleX = p.size
      p.particle.scaleY = p.size
      p.particle.tint = p.hue

      // Draw trail
      if (trail.length > 1) {
        for (let j = 1; j < trail.length; j++) {
          const trailAlpha = (j / trail.length) * alpha * 0.2
          trailGraphics.moveTo(trail[j - 1].x, trail[j - 1].y)
          trailGraphics.lineTo(trail[j].x, trail[j].y)
          trailGraphics.stroke({
            width: p.size * 0.5,
            color: p.hue,
            alpha: trailAlpha
          })
        }
      }

      // Reset particle when life is over
      if (p.life > p.maxLife) {
        p.x = Math.random() * w
        p.y = Math.random() * h
        p.vx = 0
        p.vy = 0
        p.life = 0
        trails.set(p, [])
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
