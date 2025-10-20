<script setup lang="ts">
import type { Texture } from 'pixi.js'
import { Application, Graphics, Particle, ParticleContainer } from 'pixi.js'
import { effectScope, onMounted, onScopeDispose, onUnmounted, ref } from 'vue'
import { useEventListener } from '@vueuse/core'

const el = ref<HTMLDivElement>()

let w = window.innerWidth
let h = window.innerHeight

interface Star {
  x: number
  y: number
  vx: number
  vy: number
  particle: Particle
  brightness: number
  pulseOffset: number
}

const stars: Star[] = []
const mountedScope = effectScope()

function createStarTexture(app: Application) {
  const g = new Graphics().circle(0, 0, 2).fill(0xffffff)
  return app.renderer.generateTexture(g)
}

function addStars({ starTexture, particleContainer }: { starTexture: Texture, particleContainer: ParticleContainer }, count: number) {
  for (let i = 0; i < count; i++) {
    const particle = new Particle(starTexture)
    particle.anchorX = 0.5
    particle.anchorY = 0.5
    particleContainer.addParticle(particle)

    stars.push({
      x: Math.random() * w,
      y: Math.random() * h,
      vx: (Math.random() - 0.5) * 0.3,
      vy: (Math.random() - 0.5) * 0.3,
      particle,
      brightness: 0.3 + Math.random() * 0.7,
      pulseOffset: Math.random() * Math.PI * 2
    })
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
    dynamicProperties: { position: true, alpha: true, scale: true }
  })
  app.stage.addChild(particleContainer)

  const lineGraphics = new Graphics()
  app.stage.addChild(lineGraphics)

  const starTexture = createStarTexture(app)
  addStars({ starTexture, particleContainer }, 50)

  const connectionDistance = 150

  app.ticker.add((ticker) => {
    const t = Date.now() / 1000

    // Update star positions and appearance
    for (const star of stars) {
      star.x += star.vx
      star.y += star.vy

      // Wrap around edges
      if (star.x < -50) star.x = w + 50
      if (star.x > w + 50) star.x = -50
      if (star.y < -50) star.y = h + 50
      if (star.y > h + 50) star.y = -50

      // Update particle
      star.particle.x = star.x
      star.particle.y = star.y

      // Pulsing effect
      const pulse = Math.sin(t * 2 + star.pulseOffset) * 0.3 + 0.7
      star.particle.alpha = star.brightness * pulse
      star.particle.scaleX = 0.5 + pulse * 0.5
      star.particle.scaleY = 0.5 + pulse * 0.5
    }

    // Draw connections
    lineGraphics.clear()
    for (let i = 0; i < stars.length; i++) {
      for (let j = i + 1; j < stars.length; j++) {
        const dx = stars[i].x - stars[j].x
        const dy = stars[i].y - stars[j].y
        const distance = Math.sqrt(dx * dx + dy * dy)

        if (distance < connectionDistance) {
          const alpha = (1 - distance / connectionDistance) * 0.3
          lineGraphics.moveTo(stars[i].x, stars[i].y)
          lineGraphics.lineTo(stars[j].x, stars[j].y)
          lineGraphics.stroke({
            width: 1,
            color: 0x6366f1,
            alpha
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
