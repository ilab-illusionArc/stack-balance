<script setup lang="ts">
import { onMounted, onBeforeUnmount, ref, computed, watch } from 'vue'
import type { Body as MatterBodyType } from 'matter-js'

const mountEl = ref<HTMLDivElement | null>(null)
const stageWrapEl = ref<HTMLDivElement | null>(null)

/* ---------------- UI state ---------------- */
type GameState = 'idle' | 'playing' | 'gameover'
const state = ref<GameState>('idle')

const score = ref(0)
const best = ref(0)
const combo = ref(0)
const placed = ref(0)
const gameOverReason = ref('')

/* Perfect meter */
const lastDxPx = ref<number | null>(null)
const lastPerfect = ref(false)
const lastDropMsg = ref('')

const meterPct = computed(() => {
  if (lastDxPx.value == null) return 0
  const dx = Math.max(0, Math.min(80, lastDxPx.value))
  return Math.max(0, 100 - (dx / 80) * 100)
})

/* Mode */
type Mode = 'daily' | 'endless'
const mode = ref<Mode>('daily')

/* Preview (DOM overlay, screen space) */
type ShapeDef = { id: string; w: number; h: number; color: string }
const previewX = ref(0)
const previewScreenY = 96
const previewShape = ref<ShapeDef>({ id: 'rect', w: 110, h: 42, color: '#60A5FA' })
const previewVisible = computed(() => state.value === 'playing')

/* Payload */
const runSeed = ref<number>(0)
const runDateKey = ref<string>('')

/* Micro animations */
const comboPop = ref(false)
watch(combo, (v, prev) => {
  if (v > prev) {
    comboPop.value = true
    setTimeout(() => (comboPop.value = false), 180)
  }
})

let cleanup: null | (() => void) = null

onMounted(async () => {
  if (!mountEl.value || !stageWrapEl.value) return
  const container = mountEl.value
  const stageWrap = stageWrapEl.value

  const Matter = await import('matter-js')
  const { Engine, Render, Runner, Bodies, Composite, Events } = Matter

  /* ---------- best ---------- */
  const BEST_KEY = 'stack_balance.best.v8'
  try {
    best.value = Number(localStorage.getItem(BEST_KEY) || '0') || 0
  } catch {
    best.value = 0
  }
  const setBestIfNeeded = () => {
    if (score.value > best.value) {
      best.value = score.value
      try {
        localStorage.setItem(BEST_KEY, String(best.value))
      } catch {}
    }
  }

  /* ---------------- Helpers ---------------- */
  const clamp = (n: number, a: number, b: number) => Math.max(a, Math.min(b, n))

  const mulberry32 = (seed: number) => {
    let t = seed >>> 0
    return () => {
      t += 0x6d2b79f5
      let x = Math.imul(t ^ (t >>> 15), 1 | t)
      x ^= x + Math.imul(x ^ (x >>> 7), 61 | x)
      return ((x ^ (x >>> 14)) >>> 0) / 4294967296
    }
  }

  const todayKey = () => {
    const d = new Date()
    const yyyy = d.getFullYear()
    const mm = String(d.getMonth() + 1).padStart(2, '0')
    const dd = String(d.getDate()).padStart(2, '0')
    return `${yyyy}-${mm}-${dd}`
  }

  const hashString = (s: string) => {
    let h = 2166136261
    for (let i = 0; i < s.length; i++) {
      h ^= s.charCodeAt(i)
      h = Math.imul(h, 16777619)
    }
    return h >>> 0
  }

  const randomU32 = () => {
    try {
      const a = new Uint32Array(1)
      crypto.getRandomValues(a)
      return a[0] >>> 0
    } catch {
      return (Math.floor(Math.random() * 0xffffffff) >>> 0)
    }
  }

  const setPlugin = (b: MatterBodyType, key: string, val: any) => {
    const anyB = b as any
    if (!anyB.plugin) anyB.plugin = {}
    anyB.plugin[key] = val
  }
  const getPlugin = <T,>(b: MatterBodyType, key: string): T | undefined => {
    const anyB = b as any
    return anyB.plugin?.[key] as T | undefined
  }

  /* ---------------- Shapes ---------------- */
  const shapes: [ShapeDef, ...ShapeDef[]] = [
    { id: 'rect', w: 110, h: 42, color: '#60A5FA' },
    { id: 'square', w: 72, h: 72, color: '#34D399' },
    { id: 'bar', w: 150, h: 30, color: '#A78BFA' },
    { id: 'tall', w: 58, h: 92, color: '#FBBF24' },
    { id: 'wide', w: 128, h: 54, color: '#F87171' }
  ]

  let rng = mulberry32(1)
  const pickShape = (): ShapeDef => {
    const idx = Math.floor(rng() * shapes.length)
    return shapes[idx]!
  }

  /* ---------------- Engine + Renderer ---------------- */
  const engine = Engine.create()
  engine.gravity.y = 1
  engine.enableSleeping = true

  const getSize = () => {
    const w = Math.max(320, container.clientWidth || 800)
    const h = Math.max(480, container.clientHeight || 560)
    return { w, h }
  }

  let { w: width, h: height } = getSize()

  const render = Render.create({
    element: container,
    engine,
    options: {
      width,
      height,
      wireframes: false,
      // ✅ Let our CSS background show through
      background: 'transparent',
      hasBounds: true,
      // crisper on retina without going crazy
      pixelRatio: Math.min(2, window.devicePixelRatio || 1)
    }
  })

  const runner = Runner.create({ isFixed: true, delta: 1000 / 60 })

  /* ---------------- Static arena bounds (ONLY GROUND) ---------------- */
  const groundThickness = 64

  const makeBounds = () => {
    const ground = Bodies.rectangle(width / 2, height + groundThickness / 2, width * 10, groundThickness, {
      isStatic: true,
      render: {
        fillStyle: 'rgba(15, 23, 42, 0.85)',
        strokeStyle: 'rgba(255,255,255,0.10)',
        lineWidth: 1
      }
    })
    return { ground }
  }

  let bounds = makeBounds()
  Composite.add(engine.world, [bounds.ground])

  /* ---------------- Game objects ---------------- */
  const spawned: MatterBodyType[] = []
  let previewDir: 1 | -1 = 1
  let previewSpeed = 240
  let dropLock = false

  const removeBody = (b: MatterBodyType) => Composite.remove(engine.world, b)
  const clearSpawned = () => {
    for (const b of spawned.splice(0, spawned.length)) removeBody(b)
  }

  const resetPreview = () => {
    previewShape.value = pickShape()
    previewX.value = width / 2
    previewDir = 1
  }

  const setLastDropUI = (dxPx: number, perfect: boolean) => {
    lastDxPx.value = dxPx
    lastPerfect.value = perfect
    if (perfect) lastDropMsg.value = 'Perfect!'
    else if (dxPx <= 25) lastDropMsg.value = 'Nice'
    else if (dxPx <= 55) lastDropMsg.value = 'Okay'
    else lastDropMsg.value = 'Wild'

    // small haptic on perfect (mobile)
    if (perfect && 'vibrate' in navigator) {
      try {
        ;(navigator as any).vibrate?.(10)
      } catch {}
    }

    setTimeout(() => {
      if (lastDxPx.value === dxPx) {
        lastDxPx.value = null
        lastDropMsg.value = ''
      }
    }, 900)
  }

  /* ---------------- Deterministic seed per run ---------------- */
  const initRunSeed = () => {
    if (mode.value === 'daily') {
      const dk = todayKey()
      runDateKey.value = dk
      runSeed.value = hashString(dk)
    } else {
      runDateKey.value = ''
      runSeed.value = randomU32()
    }
    rng = mulberry32(runSeed.value)
  }

  /* ---------------- Camera (ground anchored, zoom-out only) ---------------- */
  const groundY = () => height

  const groundScreenRatio = 0.88
  const topScreenRatio = 0.16
  const maxZoom = 3.0

  let viewRangeY = height
  let targetRangeY = height

  const viewRangeX = () => viewRangeY * (width / height)
  const viewMinX = () => width / 2 - viewRangeX() / 2
  const viewMaxX = () => width / 2 + viewRangeX() / 2

  const viewMinY = () => groundY() - groundScreenRatio * viewRangeY
  const viewMaxY = () => viewMinY() + viewRangeY

  const worldPerPixelX = () => viewRangeX() / width
  const worldPerPixelY = () => viewRangeY / height

  const screenToWorldX = (sx: number) => viewMinX() + sx * worldPerPixelX()
  const screenToWorldY = (sy: number) => viewMinY() + sy * worldPerPixelY()

  const applyCameraBounds = () => {
    render.bounds.min.x = viewMinX()
    render.bounds.max.x = viewMaxX()
    render.bounds.min.y = viewMinY()
    render.bounds.max.y = viewMaxY()
  }

  const computeTowerTopY = () => {
    let topY = Infinity
    for (const b of spawned) {
      if (getPlugin<boolean>(b, 'sb_dead')) continue
      topY = Math.min(topY, b.bounds.min.y)
    }
    if (!Number.isFinite(topY)) return groundY() - 120
    return topY
  }

  const updateCameraTarget = () => {
    const towerTopY = computeTowerTopY() - 34
    const denom = Math.max(0.01, groundScreenRatio - topScreenRatio)
    const neededRange = (groundY() - towerTopY) / denom

    const minRange = height
    const maxRange = height * maxZoom
    const clamped = clamp(neededRange, minRange, maxRange)

    // zoom-out only (prevents jitter)
    targetRangeY = Math.max(targetRangeY, clamped)
  }

  const tickCamera = () => {
    viewRangeY += (targetRangeY - viewRangeY) * 0.085
    if (Math.abs(targetRangeY - viewRangeY) < 0.15) viewRangeY = targetRangeY
    applyCameraBounds()
  }

  /* ---------------- Start / Game Over ---------------- */
  const gameOver = (reason = 'Touched surface') => {
    if (state.value !== 'playing') return
    gameOverReason.value = reason
    state.value = 'gameover'
    dropLock = true
    setBestIfNeeded()

    // stronger haptic on lose
    if ('vibrate' in navigator) {
      try {
        ;(navigator as any).vibrate?.([30, 40, 30])
      } catch {}
    }
  }

  const startGame = () => {
    initRunSeed()

    score.value = 0
    combo.value = 0
    placed.value = 0
    gameOverReason.value = ''
    lastDxPx.value = null
    lastPerfect.value = false
    lastDropMsg.value = ''

    dropLock = false
    previewSpeed = 240

    clearSpawned()

    viewRangeY = height
    targetRangeY = height
    applyCameraBounds()

    // base platform (not surface)
    const base = Bodies.rectangle(width / 2, height - 40, 240, 52, {
      isStatic: true,
      render: {
        fillStyle: 'rgba(31, 42, 68, 0.95)',
        strokeStyle: 'rgba(255,255,255,0.10)',
        lineWidth: 1
      }
    })
    Composite.add(engine.world, base)
    spawned.push(base as unknown as MatterBodyType)

    resetPreview()
    state.value = 'playing'
  }

  const getTopBody = (): MatterBodyType | null => {
    let bestB: MatterBodyType | null = null
    let bestY = Infinity
    for (const b of spawned) {
      if (getPlugin<boolean>(b, 'sb_dead')) continue
      if (b.bounds.min.y < bestY) {
        bestY = b.bounds.min.y
        bestB = b
      }
    }
    return bestB
  }

  /* ---------------- Spawn / Drop ---------------- */
  const spawnFallingBlock = (sx: number) => {
    const s = previewShape.value
    const x = screenToWorldX(sx)
    const y = screenToWorldY(previewScreenY)

    const body = Bodies.rectangle(x, y, s.w, s.h, {
      isStatic: false,
      restitution: 0.02,
      friction: 0.92,
      frictionAir: 0.012,
      render: {
        fillStyle: s.color,
        strokeStyle: 'rgba(255,255,255,0.20)',
        lineWidth: 1
      }
    })

    setPlugin(body as any, 'sb_scored', false)
    setPlugin(body as any, 'sb_dead', false)

    Composite.add(engine.world, body)
    spawned.push(body as unknown as MatterBodyType)
    return body as unknown as MatterBodyType
  }

  const drop = () => {
    if (state.value !== 'playing') return
    if (dropLock) return
    dropLock = true

    const sx = clamp(previewX.value, 36, width - 36) // no walls
    const top = getTopBody()
    const dropped = spawnFallingBlock(sx)

    // perfect in screen px
    let dxPx = 9999
    if (top) dxPx = Math.abs(dropped.position.x - top.position.x) / worldPerPixelX()
    const perfect = dxPx <= 10
    setLastDropUI(dxPx, perfect)

    // score after settle
    setTimeout(() => {
      if (state.value !== 'playing') return
      if (getPlugin<boolean>(dropped, 'sb_dead')) return

      if (!getPlugin<boolean>(dropped, 'sb_scored')) {
        setPlugin(dropped, 'sb_scored', true)
        placed.value += 1

        if (perfect) combo.value = Math.min(40, combo.value + 1)
        else combo.value = 0

        const basePts = 12
        const perfectBonus = perfect ? 14 : 0
        const nearBonus = !perfect && dxPx <= 25 ? 6 : 0
        const comboPts = perfect ? combo.value * 4 : 0
        score.value += basePts + perfectBonus + nearBonus + comboPts

        previewSpeed = Math.min(580, 240 + placed.value * 6)
      }
    }, 520)

    setTimeout(() => {
      dropLock = false
      if (state.value !== 'playing') return
      previewShape.value = pickShape()
      updateCameraTarget()
    }, 120)
  }

  /* ---------------- Lose on touching surface (ground) ---------------- */
  const onCollisionStart = (evt: any) => {
    if (state.value !== 'playing') return
    const ground = bounds.ground as unknown as MatterBodyType

    for (const p of evt.pairs) {
      const a = p.bodyA as MatterBodyType
      const b = p.bodyB as MatterBodyType

      if (a === ground && !b.isStatic) {
        setPlugin(b, 'sb_dead', true)
        return gameOver('Touched surface')
      }
      if (b === ground && !a.isStatic) {
        setPlugin(a, 'sb_dead', true)
        return gameOver('Touched surface')
      }
    }
  }
  Events.on(engine, 'collisionStart', onCollisionStart)

  /* ---------------- Input (reliable) ---------------- */
  const isEventInside = (e: Event) => {
    const anyE: any = e as any
    const path: any[] = typeof anyE.composedPath === 'function' ? anyE.composedPath() : []
    if (path && path.length) return path.includes(stageWrap)
    return e.target instanceof Node && stageWrap.contains(e.target)
  }

  let lastAnyAt = 0
  let lastPointerOrTouchAt = 0

  const handleInput = () => {
    if (state.value === 'idle') return startGame()
    if (state.value === 'gameover') return startGame()
    drop()
  }

  const onPointerDown = (e: PointerEvent) => {
    if (!isEventInside(e)) return
    e.preventDefault()
    const now = Date.now()
    if (now - lastAnyAt < 60) return
    lastAnyAt = now
    lastPointerOrTouchAt = now
    handleInput()
  }
  const onTouchStart = (e: TouchEvent) => {
    if (!isEventInside(e)) return
    e.preventDefault()
    const now = Date.now()
    if (now - lastAnyAt < 60) return
    lastAnyAt = now
    lastPointerOrTouchAt = now
    handleInput()
  }
  const onClick = (e: MouseEvent) => {
    if (!isEventInside(e)) return
    const now = Date.now()
    if (now - lastPointerOrTouchAt < 700) return
    if (now - lastAnyAt < 60) return
    lastAnyAt = now
    e.preventDefault()
    handleInput()
  }
  const onKeyDown = (e: KeyboardEvent) => {
    if (e.key !== ' ' && e.key !== 'Enter') return
    e.preventDefault()
    handleInput()
  }

  window.addEventListener('pointerdown', onPointerDown, { capture: true, passive: false })
  window.addEventListener('touchstart', onTouchStart, { capture: true, passive: false })
  window.addEventListener('click', onClick, { capture: true, passive: false })
  window.addEventListener('keydown', onKeyDown, { capture: true })

  /* ---------------- Loop ---------------- */
  const onBeforeUpdate = (evt: any) => {
    if (state.value !== 'playing') return
    const dt = (evt?.delta || 16.666) / 1000

    const margin = 36
    const next = previewX.value + previewDir * previewSpeed * dt
    const clamped = clamp(next, margin, width - margin)
    if (clamped <= margin) previewDir = 1
    else if (clamped >= width - margin) previewDir = -1
    previewX.value = clamped

    updateCameraTarget()
    tickCamera()
  }

  const onAfterUpdate = () => {
    if (state.value !== 'playing') return

    // safety: if a dynamic body goes ultra-far away, end run
    const bodies = Composite.allBodies(engine.world) as unknown as MatterBodyType[]
    for (const b of bodies) {
      if (b.isStatic) continue
      if (b.position.y > groundY() + 5200) return gameOver('Fell out')
      if (b.position.x < -8000 || b.position.x > width + 8000) return gameOver('Fell out')
      // optional perf: freeze very old sleeping blocks far below view
      if (b.position.y > viewMaxY() + 1400 * worldPerPixelY() && b.isSleeping) {
        Matter.Body.setStatic(b as any, true)
      }
    }
  }

  Events.on(engine, 'beforeUpdate', onBeforeUpdate)
  Events.on(engine, 'afterUpdate', onAfterUpdate)

  /* ---------------- Resize ---------------- */
  const onResize = () => {
    const s = getSize()
    width = s.w
    height = s.h

    render.canvas.width = width
    render.canvas.height = height
    render.options.width = width
    render.options.height = height

    Composite.remove(engine.world, bounds.ground)
    bounds = makeBounds()
    Composite.add(engine.world, [bounds.ground])

    previewX.value = clamp(previewX.value, 36, width - 36)

    viewRangeY = Math.max(height, Math.min(viewRangeY, height * maxZoom))
    targetRangeY = Math.max(viewRangeY, height)
    applyCameraBounds()
  }
  window.addEventListener('resize', onResize)

  /* ---------------- Run ---------------- */
  applyCameraBounds()
  Render.run(render)
  Runner.run(runner, engine)

  cleanup = () => {
    window.removeEventListener('resize', onResize)

    window.removeEventListener('pointerdown', onPointerDown, true)
    window.removeEventListener('touchstart', onTouchStart, true)
    window.removeEventListener('click', onClick, true)
    window.removeEventListener('keydown', onKeyDown, true)

    Events.off(engine, 'beforeUpdate', onBeforeUpdate)
    Events.off(engine, 'afterUpdate', onAfterUpdate)
    Events.off(engine, 'collisionStart', onCollisionStart)

    Render.stop(render)
    Runner.stop(runner)
    Engine.clear(engine)

    container.replaceChildren()
  }
})

onBeforeUnmount(() => cleanup?.())
</script>

<template>
  <section class="shell">
    <header class="topbar">
      <div class="brand">
        <div class="logoDot" />
        <div>
          <div class="title">Stack &amp; Balance</div>
          <div class="subtitle">
            <span v-if="state === 'idle'">Tap to start • Space/Enter</span>
            <span v-else-if="state === 'playing'">Tap to drop • Touch ground = lose</span>
            <span v-else>Tap to restart</span>
          </div>
        </div>
      </div>

      <div class="hudRow">
        <button class="pillBtn" :disabled="state === 'playing'" @click="mode = mode === 'daily' ? 'endless' : 'daily'">
          <span class="pillDot" />
          {{ mode === 'daily' ? 'Daily Seed' : 'Endless' }}
        </button>

        <div class="chip">
          <div class="k">Score</div>
          <div class="v">{{ score }}</div>
        </div>

        <div class="chip" :class="{ pop: comboPop }">
          <div class="k">Combo</div>
          <div class="v">{{ combo }}</div>
        </div>

        <div class="chip">
          <div class="k">Best</div>
          <div class="v">{{ best }}</div>
        </div>
      </div>
    </header>

    <div ref="stageWrapEl" class="stageWrap" :class="{ isPlaying: state === 'playing' }">
      <div class="bgGrid" aria-hidden="true" />
      <div class="vignette" aria-hidden="true" />

      <!-- surface glow line (visual only) -->
      <div class="surfaceLine" aria-hidden="true" />

      <div ref="mountEl" class="stage" />

      <!-- Preview (sharp corners) -->
      <div
        v-if="previewVisible"
        class="preview"
        :style="{
          width: previewShape.w + 'px',
          height: previewShape.h + 'px',
          left: previewX + 'px',
          top: previewScreenY + 'px',
          background: previewShape.color
        }"
        aria-hidden="true"
      >
        <div class="sheen" />
      </div>

      <!-- “Tap to drop” hint -->
      <div v-if="state === 'playing'" class="tapHint" aria-hidden="true">
        Tap / Space to drop
      </div>

      <!-- Perfect meter -->
      <div v-if="state === 'playing'" class="meterCard" aria-label="Perfect meter">
        <div class="meterTop">
          <div class="meterLabel">Alignment</div>
          <div class="meterMsg" :class="{ good: lastPerfect }">
            <span v-if="lastDxPx == null">—</span>
            <span v-else>{{ lastDropMsg }}</span>
          </div>
        </div>
        <div class="meterBar">
          <div class="meterFill" :style="{ width: meterPct + '%' }" />
        </div>
        <div class="meterFoot">
          <span class="muted">Δ</span>
          <span v-if="lastDxPx == null">—</span>
          <span v-else>{{ Math.round(lastDxPx) }}px</span>
        </div>
      </div>

      <!-- Overlay -->
      <div v-if="state !== 'playing'" class="overlay" role="dialog" aria-modal="true">
        <div class="overlayCard">
          <div class="overlayTitle">{{ state === 'idle' ? 'Ready?' : 'Game Over' }}</div>
          <div class="overlayText">
            <template v-if="state === 'idle'">
              Tap anywhere to start. Rule: if any block touches the surface, you lose.
            </template>
            <template v-else>
              <div class="reason">Reason: <b>{{ gameOverReason || 'Touched surface' }}</b></div>
              <div class="summary">
                <div class="sumChip">
                  <div class="k">Score</div>
                  <div class="v">{{ score }}</div>
                </div>
                <div class="sumChip">
                  <div class="k">Best</div>
                  <div class="v">{{ best }}</div>
                </div>
                <div class="sumChip">
                  <div class="k">Placed</div>
                  <div class="v">{{ placed }}</div>
                </div>
              </div>
              <div class="cta">Tap to restart</div>

              <details class="debug">
                <summary>Leaderboard payload</summary>
                <pre class="payloadBox">{{
JSON.stringify(
  {
    game: 'stack-balance',
    mode,
    date: mode === 'daily' ? runDateKey : null,
    seed: runSeed,
    score,
    placed,
    combo
  },
  null,
  2
)
}}</pre>
              </details>
            </template>
          </div>
        </div>
      </div>
    </div>
  </section>
</template>

<style scoped>
/* Layout shell */
.shell {
  width: min(1100px, 100%);
  margin: 0 auto;
  border-radius: 18px;
  overflow: hidden;
  border: 1px solid rgba(255, 255, 255, 0.10);
  background: rgba(255, 255, 255, 0.03);
  box-shadow: 0 24px 90px rgba(0, 0, 0, 0.55);
}

/* Topbar */
.topbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 14px;
  padding: 14px 16px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.10);
  background: linear-gradient(
    to bottom,
    rgba(255, 255, 255, 0.06),
    rgba(255, 255, 255, 0.02)
  );
  backdrop-filter: blur(14px);
}

.brand {
  display: flex;
  align-items: center;
  gap: 12px;
  min-width: 220px;
}
.logoDot {
  width: 14px;
  height: 14px;
  border-radius: 999px;
  background: radial-gradient(circle at 30% 30%, rgba(255,255,255,0.9), rgba(255,255,255,0.15));
  box-shadow: 0 0 0 1px rgba(255,255,255,0.22), 0 10px 30px rgba(0,0,0,0.45);
}
.title {
  font-weight: 950;
  letter-spacing: 0.2px;
  font-size: 16px;
}
.subtitle {
  font-size: 12px;
  opacity: 0.75;
  margin-top: 2px;
}

.hudRow {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 10px;
  justify-content: flex-end;
}

.pillBtn {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  font-size: 12px;
  padding: 8px 10px;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.06);
  border: 1px solid rgba(255, 255, 255, 0.12);
  color: white;
  cursor: pointer;
}
.pillBtn:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}
.pillDot {
  width: 8px;
  height: 8px;
  border-radius: 999px;
  background: rgba(255,255,255,0.75);
  box-shadow: 0 0 14px rgba(255,255,255,0.35);
}

.chip {
  display: grid;
  gap: 3px;
  padding: 8px 10px;
  min-width: 92px;
  border-radius: 14px;
  background: rgba(255, 255, 255, 0.055);
  border: 1px solid rgba(255, 255, 255, 0.10);
}
.chip .k {
  font-size: 11px;
  opacity: 0.70;
}
.chip .v {
  font-size: 16px;
  font-weight: 950;
  line-height: 1;
}
.chip.pop {
  animation: pop 180ms ease-out;
}
@keyframes pop {
  0% { transform: scale(1); }
  60% { transform: scale(1.06); }
  100% { transform: scale(1); }
}

/* Stage */
.stageWrap {
  position: relative;
  height: 580px;
  background:
    radial-gradient(1200px 600px at 50% 0%, rgba(99,102,241,0.16), rgba(0,0,0,0) 60%),
    radial-gradient(800px 420px at 10% 20%, rgba(34,197,94,0.10), rgba(0,0,0,0) 60%),
    radial-gradient(900px 520px at 90% 10%, rgba(236,72,153,0.12), rgba(0,0,0,0) 60%),
    linear-gradient(to bottom, rgba(6,8,16,1), rgba(2,3,8,1));
  overflow: hidden;
}

.stage {
  width: 100%;
  height: 100%;
  touch-action: none;
  position: relative;
  z-index: 2;
}
:deep(canvas) {
  display: block;
  width: 100%;
  height: 100%;
  touch-action: none;
}

/* Subtle grid */
.bgGrid {
  position: absolute;
  inset: 0;
  z-index: 0;
  opacity: 0.28;
  background-image:
    linear-gradient(rgba(255,255,255,0.06) 1px, transparent 1px),
    linear-gradient(90deg, rgba(255,255,255,0.06) 1px, transparent 1px);
  background-size: 48px 48px;
  mask-image: radial-gradient(circle at 50% 20%, rgba(0,0,0,1), rgba(0,0,0,0.2) 55%, rgba(0,0,0,0) 78%);
}
.vignette {
  position: absolute;
  inset: -1px;
  z-index: 1;
  pointer-events: none;
  background: radial-gradient(circle at 50% 40%, rgba(0,0,0,0) 40%, rgba(0,0,0,0.55) 100%);
}

/* Surface line: at ~88% height (matches camera ground ratio) */
.surfaceLine {
  position: absolute;
  left: -20%;
  width: 140%;
  top: 88%;
  height: 2px;
  z-index: 3;
  background: linear-gradient(90deg, rgba(255,255,255,0) 0%, rgba(255,255,255,0.32) 20%, rgba(255,255,255,0.32) 80%, rgba(255,255,255,0) 100%);
  filter: drop-shadow(0 10px 24px rgba(99,102,241,0.25));
  opacity: 0.85;
}

/* Preview (sharp corners) */
.preview {
  position: absolute;
  transform: translate(-50%, -50%);
  border-radius: 0px; /* ✅ sharp corners */
  opacity: 0.95;
  box-shadow: 0 18px 50px rgba(0,0,0,0.55);
  outline: 1px solid rgba(255,255,255,0.28);
  pointer-events: none;
  z-index: 4;
}
.sheen {
  position: absolute;
  inset: 0;
  background: linear-gradient(115deg, rgba(255,255,255,0.20), rgba(255,255,255,0.0) 60%);
  mix-blend-mode: overlay;
  opacity: 0.75;
}

/* Tap hint */
.tapHint {
  position: absolute;
  left: 50%;
  bottom: 14px;
  transform: translateX(-50%);
  z-index: 6;
  padding: 8px 10px;
  border-radius: 999px;
  background: rgba(0,0,0,0.40);
  border: 1px solid rgba(255,255,255,0.14);
  backdrop-filter: blur(12px);
  font-size: 12px;
  opacity: 0.85;
  pointer-events: none;
}

/* Meter */
.meterCard {
  position: absolute;
  left: 14px;
  bottom: 14px;
  z-index: 6;
  width: 280px;
  padding: 12px 12px;
  border-radius: 16px;
  background: rgba(0,0,0,0.38);
  border: 1px solid rgba(255,255,255,0.14);
  backdrop-filter: blur(14px);
  pointer-events: none;
}
.meterTop {
  display: flex;
  align-items: baseline;
  justify-content: space-between;
  gap: 10px;
}
.meterLabel {
  font-size: 11px;
  opacity: 0.75;
}
.meterMsg {
  font-size: 12px;
  opacity: 0.85;
}
.meterMsg.good {
  opacity: 1;
  font-weight: 900;
}
.meterBar {
  margin-top: 8px;
  height: 10px;
  border-radius: 999px;
  background: rgba(255,255,255,0.10);
  overflow: hidden;
}
.meterFill {
  height: 100%;
  border-radius: 999px;
  background: rgba(255,255,255,0.75);
  box-shadow: 0 0 24px rgba(255,255,255,0.18);
  transition: width 120ms linear;
}
.meterFoot {
  margin-top: 8px;
  font-size: 12px;
  opacity: 0.85;
  display: flex;
  gap: 6px;
  align-items: center;
}
.muted { opacity: 0.7; }

/* Overlay */
.overlay {
  position: absolute;
  inset: 0;
  z-index: 10;
  display: grid;
  place-items: center;
  padding: 16px;
  pointer-events: none;
}
.overlayCard {
  pointer-events: none;
  width: min(720px, 92vw);
  padding: 16px 16px;
  border-radius: 18px;
  background: rgba(0,0,0,0.48);
  border: 1px solid rgba(255,255,255,0.16);
  backdrop-filter: blur(16px);
  box-shadow: 0 28px 90px rgba(0,0,0,0.60);
  text-align: center;
}
.overlayTitle {
  font-weight: 950;
  font-size: 20px;
  letter-spacing: 0.2px;
}
.overlayText {
  margin-top: 10px;
  font-size: 13px;
  opacity: 0.88;
  line-height: 1.45;
}
.reason { margin-bottom: 10px; }

.summary {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 10px;
  margin: 10px 0 8px;
}
.sumChip {
  border-radius: 14px;
  padding: 10px 10px;
  background: rgba(255,255,255,0.06);
  border: 1px solid rgba(255,255,255,0.12);
}
.sumChip .k { font-size: 11px; opacity: 0.7; }
.sumChip .v { font-size: 18px; font-weight: 950; margin-top: 2px; }

.cta {
  margin-top: 6px;
  font-size: 12px;
  opacity: 0.8;
}

.debug {
  margin-top: 12px;
  text-align: left;
  opacity: 0.95;
}
.debug summary {
  cursor: pointer;
  font-size: 12px;
  opacity: 0.85;
}
.payloadBox {
  margin-top: 8px;
  padding: 10px;
  border-radius: 14px;
  background: rgba(255,255,255,0.06);
  border: 1px solid rgba(255,255,255,0.12);
  max-height: 220px;
  overflow: auto;
  font-size: 11px;
  line-height: 1.35;
}
@media (max-width: 680px) {
  .chip { min-width: 80px; }
  .summary { grid-template-columns: 1fr; }
  .meterCard { width: 240px; }
}
</style>
