<script setup lang="ts">
import { onMounted, onBeforeUnmount, ref, computed, watch } from 'vue'
import type { Body as MatterBody } from 'matter-js'

const stageWrapEl = ref<HTMLDivElement | null>(null)
const mountEl = ref<HTMLDivElement | null>(null)

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

/* Device */
const isCoarsePointer = ref(false)

/* Micro animation */
const comboPop = ref(false)
watch(combo, (v, prev) => {
  if (v > prev) {
    comboPop.value = true
    setTimeout(() => (comboPop.value = false), 180)
  }
})

/* Payload */
const runSeed = ref<number>(0)
const runDateKey = ref<string>('')

/* Exposed actions */
let startGameFn: null | (() => void) = null
let dropFn: null | (() => void) = null
const startOrDrop = () => {
  if (state.value === 'idle' || state.value === 'gameover') startGameFn?.()
  else dropFn?.()
}

let cleanup: null | (() => void) = null

onMounted(async () => {
  if (!stageWrapEl.value || !mountEl.value) return
  const stageWrap = stageWrapEl.value
  const mount = mountEl.value

  try {
    isCoarsePointer.value = window.matchMedia?.('(pointer: coarse)').matches ?? false
  } catch {
    isCoarsePointer.value = false
  }

  const Matter = await import('matter-js')
  const { Engine, Bodies, Composite, Events, Body: MBody } = Matter

  const THREE = await import('three')

  // Postprocessing
  const { EffectComposer } = await import('three/examples/jsm/postprocessing/EffectComposer.js')
  const { RenderPass } = await import('three/examples/jsm/postprocessing/RenderPass.js')
  const { UnrealBloomPass } = await import('three/examples/jsm/postprocessing/UnrealBloomPass.js')

  /* ---------- best ---------- */
  const BEST_KEY = 'stack_balance.best.neonglow.v1'
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

  /* ---------------- helpers ---------------- */
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

  const setPlugin = (b: MatterBody, key: string, val: any) => {
    const anyB = b as any
    if (!anyB.plugin) anyB.plugin = {}
    anyB.plugin[key] = val
  }
  const getPlugin = <T,>(b: MatterBody, key: string): T | undefined => {
    const anyB = b as any
    return anyB.plugin?.[key] as T | undefined
  }

  /* ---------------- shapes ---------------- */
  type ShapeDef = { id: string; w: number; h: number; color: string }
  const shapes: [ShapeDef, ...ShapeDef[]] = [
    { id: 'rect', w: 110, h: 42, color: '#FF2BD6' }, // pink
    { id: 'square', w: 72, h: 72, color: '#00D6FF' }, // cyan
    { id: 'bar', w: 150, h: 30, color: '#B067FF' }, // purple
    { id: 'tall', w: 58, h: 92, color: '#FFD24A' }, // amber
    { id: 'wide', w: 128, h: 54, color: '#FF4D8D' }  // hot pink
  ]

  let rng = mulberry32(1)
  const pickShape = (): ShapeDef => {
    const idx = Math.floor(rng() * shapes.length)
    return shapes[idx]!
  }

  /* ---------------- sizing ---------------- */
  const getSize = () => {
    const w = Math.max(320, stageWrap.clientWidth || window.innerWidth || 800)
    const h = Math.max(480, stageWrap.clientHeight || window.innerHeight || 720)
    return { w, h }
  }
  let { w: width, h: height } = getSize()

  // px -> world
  const S = 0.0095
  const DEPTH = 0.6
  const PREVIEW_Z = 1.25

  const toThreeX = (mx: number) => (mx - width / 2) * S
  const toThreeY = (my: number) => (height - my) * S
  const toThreeRotZ = (angle: number) => -angle

  /* ---------------- physics ---------------- */
  const engine = Engine.create()
  engine.gravity.y = 1
  engine.enableSleeping = true

  const groundThickness = 64
  const makeGround = () =>
    Bodies.rectangle(width / 2, height + groundThickness / 2, width * 16, groundThickness, { isStatic: true })

  let ground = makeGround()
  Composite.add(engine.world, [ground])

  /* ---------------- three scene ---------------- */
  const scene = new THREE.Scene()
  scene.background = new THREE.Color('#070012')
  scene.fog = new THREE.Fog(new THREE.Color('#070012'), 7, 46)

  const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: false, powerPreference: 'high-performance' })
  renderer.setPixelRatio(Math.min(isCoarsePointer.value ? 1.4 : 2, window.devicePixelRatio || 1))
  renderer.setSize(width, height)
  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFSoftShadowMap
  renderer.domElement.style.width = '100%'
  renderer.domElement.style.height = '100%'
  renderer.domElement.style.display = 'block'

  // tone/color
  const anyR = renderer as any
  if ('outputColorSpace' in anyR && (THREE as any).SRGBColorSpace) anyR.outputColorSpace = (THREE as any).SRGBColorSpace
  else if ('outputEncoding' in anyR && (THREE as any).sRGBEncoding) anyR.outputEncoding = (THREE as any).sRGBEncoding
  if ((THREE as any).ACESFilmicToneMapping) {
    anyR.toneMapping = (THREE as any).ACESFilmicToneMapping
    anyR.toneMappingExposure = 1.85
  }

  mount.replaceChildren(renderer.domElement)

  const camera = new THREE.PerspectiveCamera(45, width / height, 0.1, 240)

  // camera + shake
  let cam = { y: 7.8, z: 15.2 }
  let camTarget = { y: 7.8, z: 15.2 }
  let lookY = 2.4
  let lookTargetY = 2.4
  let shake = 0

  const tickCamera = () => {
    cam.y += (camTarget.y - cam.y) * 0.06
    cam.z += (camTarget.z - cam.z) * 0.06
    lookY += (lookTargetY - lookY) * 0.06
    shake *= 0.88

    const sx = (Math.random() - 0.5) * 0.14 * shake
    const sy = (Math.random() - 0.5) * 0.14 * shake

    camera.position.set(sx, cam.y + sy, cam.z)
    camera.lookAt(0, lookY, 0)
  }

  // lights (pink synthwave)
  scene.add(new THREE.AmbientLight(0xffffff, 0.40))
  scene.add(new THREE.HemisphereLight(0xffc7ff, 0x070012, 0.75))

  const key = new THREE.DirectionalLight(0xffffff, 1.0)
  key.position.set(8, 18, 12)
  key.castShadow = true
  key.shadow.mapSize.set(isCoarsePointer.value ? 1024 : 2048, isCoarsePointer.value ? 1024 : 2048)
  key.shadow.camera.near = 0.5
  key.shadow.camera.far = 120
  key.shadow.camera.left = -26
  key.shadow.camera.right = 26
  key.shadow.camera.top = 26
  key.shadow.camera.bottom = -26
  scene.add(key)

  const pink = new THREE.PointLight(0xff2bd6, 1.8, 90)
  pink.position.set(8, 9, 8)
  scene.add(pink)

  const purple = new THREE.PointLight(0xb067ff, 1.25, 90)
  purple.position.set(-9, 10, 7)
  scene.add(purple)

  const cyan = new THREE.PointLight(0x00d6ff, 1.0, 90)
  cyan.position.set(0, 13, -9)
  scene.add(cyan)

  // floor (emissive)
  const floorGeo = new THREE.PlaneGeometry((width * 16) * S, 52)
  const floorMat = new THREE.MeshStandardMaterial({
    color: new THREE.Color('#050014'),
    roughness: 0.94,
    metalness: 0.06,
    emissive: new THREE.Color('#2a0036'),
    emissiveIntensity: 1.05
  })
  const floor = new THREE.Mesh(floorGeo, floorMat)
  floor.rotation.x = -Math.PI / 2
  floor.position.set(0, 0, 0)
  floor.receiveShadow = true
  scene.add(floor)

  // neon grid (so bloom picks it up)
  const grid = new THREE.GridHelper(60, 60, 0xff2bd6, 0x2a0040)
  ;(grid.material as any).opacity = 0.55
  ;(grid.material as any).transparent = true
  grid.position.y = 0.005
  scene.add(grid)

  /* ---------------- postprocessing bloom ---------------- */
  const composer = new EffectComposer(renderer)
  composer.addPass(new RenderPass(scene, camera))

  // Stronger bloom so emissive glows
  const bloom = new UnrealBloomPass(new THREE.Vector2(width, height), 1.25, 0.65, 0.78)
  if (isCoarsePointer.value) {
    bloom.strength = 0.95
    bloom.radius = 0.42
    bloom.threshold = 0.76
  } else {
    bloom.strength = 1.35
    bloom.radius = 0.55
    bloom.threshold = 0.74
  }
  composer.addPass(bloom)

  /* ---------------- neon block factory ---------------- */
  const meshById = new Map<number, THREE.Object3D>()

  const makeNeonBlock = (wPx: number, hPx: number, hex: string, preview = false) => {
    const group = new THREE.Group()

    // CORE
    const coreGeo = new THREE.BoxGeometry(wPx * S, hPx * S, DEPTH)
    const c = new THREE.Color(hex)
    const coreMat = new THREE.MeshStandardMaterial({
      color: c,
      roughness: preview ? 0.18 : 0.28,
      metalness: 0.08,
      emissive: c.clone().multiplyScalar(preview ? 1.0 : 0.85),
      emissiveIntensity: preview ? 2.2 : 1.6
    })
    const core = new THREE.Mesh(coreGeo, coreMat)
    core.castShadow = true
    core.receiveShadow = true
    group.add(core)

    // GLOW SHELL (additive) — makes it glow even without heavy bloom
    const glowGeo = new THREE.BoxGeometry(wPx * S, hPx * S, DEPTH)
    const glowMat = new THREE.MeshBasicMaterial({
      color: c,
      transparent: true,
      opacity: preview ? 0.22 : 0.14,
      blending: THREE.AdditiveBlending,
      depthWrite: false
    })
    const glow = new THREE.Mesh(glowGeo, glowMat)
    glow.scale.set(1.09, 1.09, 1.09)
    glow.renderOrder = 999
    group.add(glow)

    // EDGE LINES (optional but looks great in synthwave)
    const edgeGeo = new THREE.EdgesGeometry(coreGeo, 25)
    const edgeMat = new THREE.LineBasicMaterial({
      color: c,
      transparent: true,
      opacity: preview ? 0.85 : 0.55
    })
    const edges = new THREE.LineSegments(edgeGeo, edgeMat)
    edges.renderOrder = 1000
    group.add(edges)

    // store disposers on group for cleanup
    ;(group as any).__dispose = () => {
      coreGeo.dispose()
      coreMat.dispose()
      glowGeo.dispose()
      glowMat.dispose()
      edgeGeo.dispose()
      edgeMat.dispose()
    }

    return group
  }

  /* ---------------- gameplay ---------------- */
  const spawned: MatterBody[] = []

  let previewShape: ShapeDef = pickShape()
  let previewObj = makeNeonBlock(previewShape.w, previewShape.h, previewShape.color, true)
  scene.add(previewObj)

  const setPreviewShape = (shape: ShapeDef) => {
    previewShape = shape
    const old = previewObj
    previewObj = makeNeonBlock(shape.w, shape.h, shape.color, true)
    previewObj.position.copy(old.position)
    previewObj.rotation.copy(old.rotation)
    scene.add(previewObj)
    scene.remove(old)
    try { ;(old as any).__dispose?.() } catch {}
  }

  let previewX = width / 2
  let previewDir: 1 | -1 = 1
  let previewSpeed = 320
  let dropLock = false

  // spawn height logic
  const PREVIEW_BASE_Y = 145
  const PREVIEW_MIN_Y = 22
  const SPAWN_MARGIN_ABOVE_TOP = 200

  const computeTowerTopY = () => {
    let topY = Infinity
    for (const b of spawned) {
      if (getPlugin<boolean>(b, 'sb_dead')) continue
      topY = Math.min(topY, b.bounds.min.y)
    }
    if (!Number.isFinite(topY)) return height - 120
    return topY
  }

  const computeSpawnY = () => {
    const topY = computeTowerTopY()
    const desired = topY - SPAWN_MARGIN_ABOVE_TOP
    const y = Math.min(PREVIEW_BASE_Y, desired)
    return clamp(y, PREVIEW_MIN_Y, PREVIEW_BASE_Y)
  }

  const updateCameraTargets = (previewY: number) => {
    const topY = computeTowerTopY()
    const topWorld = toThreeY(topY)
    const prevWorld = toThreeY(previewY)

    const focus = Math.max(2.4, Math.min(prevWorld, prevWorld * 0.7 + topWorld * 0.3))
    lookTargetY = Math.max(lookTargetY, focus)

    const desiredY = 7.4 + focus * 1.08
    const desiredZ = 13.6 + focus * 1.85
    camTarget.y = Math.max(camTarget.y, desiredY)
    camTarget.z = Math.max(camTarget.z, desiredZ)
  }

  const clearAll = () => {
    for (const b of spawned.splice(0, spawned.length)) {
      try { Composite.remove(engine.world, b) } catch {}
      const id = (b as any).id as number
      const obj = meshById.get(id)
      if (obj) {
        scene.remove(obj)
        try { ;(obj as any).__dispose?.() } catch {}
        meshById.delete(id)
      }
    }
  }

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

  const setLastDropUI = (dxPx: number, perfect: boolean) => {
    lastDxPx.value = dxPx
    lastPerfect.value = perfect
    if (perfect) lastDropMsg.value = 'Perfect!'
    else if (dxPx <= 25) lastDropMsg.value = 'Nice'
    else if (dxPx <= 55) lastDropMsg.value = 'Okay'
    else lastDropMsg.value = 'Wild'
    setTimeout(() => {
      if (lastDxPx.value === dxPx) {
        lastDxPx.value = null
        lastDropMsg.value = ''
      }
    }, 900)
  }

  const gameOver = (reason = 'Touched surface') => {
    if (state.value !== 'playing') return
    gameOverReason.value = reason
    state.value = 'gameover'
    dropLock = true
    setBestIfNeeded()
    shake = Math.max(shake, 1.1)
  }

  const spawnBodyAndObj = (sx: number, sy: number) => {
    const body = Bodies.rectangle(sx, sy, previewShape.w, previewShape.h, {
      isStatic: false,
      restitution: 0.02,
      friction: 0.92,
      frictionAir: 0.012
    }) as unknown as MatterBody

    setPlugin(body, 'sb_scored', false)
    setPlugin(body, 'sb_dead', false)

    spawned.push(body)
    Composite.add(engine.world, body)

    const obj = makeNeonBlock(previewShape.w, previewShape.h, previewShape.color, false)
    scene.add(obj)
    meshById.set((body as any).id, obj)

    return body
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
    previewSpeed = 320

    clearAll()

    try { Composite.remove(engine.world, ground) } catch {}
    ground = makeGround()
    Composite.add(engine.world, [ground])

    cam = { y: 7.8, z: 15.2 }
    camTarget = { y: 7.8, z: 15.2 }
    lookY = 2.4
    lookTargetY = 2.4
    shake = 0

    // base platform (not surface)
    const base = Bodies.rectangle(width / 2, height - 40, 240, 52, { isStatic: true }) as unknown as MatterBody
    spawned.push(base)
    Composite.add(engine.world, base)

    const baseObj = makeNeonBlock(240, 52, '#5b2bff', false)
    scene.add(baseObj)
    meshById.set((base as any).id, baseObj)

    previewX = width / 2
    previewDir = 1
    setPreviewShape(pickShape())

    state.value = 'playing'
  }

  const getTopBody = () => {
    let bestB: MatterBody | null = null
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

  const drop = () => {
    if (state.value !== 'playing') return
    if (dropLock) return
    dropLock = true

    const margin = 36
    const sx = clamp(previewX, margin, width - margin)
    const sy = computeSpawnY()

    const top = getTopBody()
    const dropped = spawnBodyAndObj(sx, sy)

    let dxPx = 9999
    if (top) dxPx = Math.abs((dropped as any).position.x - (top as any).position.x)
    const perfect = dxPx <= 10
    setLastDropUI(dxPx, perfect)

    if (perfect) {
      shake = Math.max(shake, 0.75)
      bloom.strength = isCoarsePointer.value ? 1.05 : 1.65
      setTimeout(() => (bloom.strength = isCoarsePointer.value ? 0.95 : 1.35), 220)
    }

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

        previewSpeed = Math.min(760, 320 + placed.value * 6)
      }
    }, 520)

    setTimeout(() => {
      dropLock = false
      if (state.value !== 'playing') return
      setPreviewShape(pickShape())
    }, 120)
  }

  startGameFn = startGame
  dropFn = drop

  /* ---------------- Lose on touching ground ---------------- */
  const onCollisionStart = (evt: any) => {
    if (state.value !== 'playing') return
    for (const p of evt.pairs) {
      const a = p.bodyA as MatterBody
      const b = p.bodyB as MatterBody
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

  /* ---------------- Input ---------------- */
  const onStagePointerDown = (e: PointerEvent) => {
    if (e.pointerType === 'mouse' && (e as any).button != null && (e as any).button !== 0) return
    e.preventDefault()
    startOrDrop()
  }
  stageWrap.addEventListener('pointerdown', onStagePointerDown, { passive: false })

  const onKeyDown = (e: KeyboardEvent) => {
    if (e.key !== ' ' && e.key !== 'Enter') return
    e.preventDefault()
    startOrDrop()
  }
  window.addEventListener('keydown', onKeyDown, { passive: false })

  /* ---------------- loop ---------------- */
  let raf = 0
  let last = performance.now()
  let acc = 0
  const FIXED = 1000 / 60

  const animate = (t: number) => {
    raf = requestAnimationFrame(animate)
    const dtReal = Math.min(50, t - last)
    last = t
    acc += dtReal

    // preview motion
    if (state.value === 'playing') {
      const margin = 36
      const step = (previewSpeed * dtReal) / 1000
      let nx = previewX + previewDir * step
      if (nx < margin) { nx = margin; previewDir = 1 }
      else if (nx > width - margin) { nx = width - margin; previewDir = -1 }
      previewX = nx
    }

    // physics
    while (acc >= FIXED) {
      Engine.update(engine, FIXED)
      acc -= FIXED

      // wrap-around (no walls)
      const wrap = 620
      for (const b of spawned) {
        if (b.isStatic) continue
        const x = (b as any).position.x
        if (x < -wrap) MBody.setPosition(b as any, { x: width + wrap, y: (b as any).position.y })
        else if (x > width + wrap) MBody.setPosition(b as any, { x: -wrap, y: (b as any).position.y })
      }
    }

    // preview object
    const previewY = computeSpawnY()
    previewObjUpdate(previewX, previewY)

    // sync objs
    for (const b of spawned) {
      const id = (b as any).id as number
      const obj = meshById.get(id)
      if (!obj) continue
      obj.position.set(toThreeX((b as any).position.x), toThreeY((b as any).position.y), 0)
      obj.rotation.z = toThreeRotZ((b as any).angle)
    }

    if (state.value === 'playing') updateCameraTargets(previewY)
    tickCamera()

    composer.render()
  }

  // preview helper (keeps z and rotation clean)
  const previewObjUpdate = (xPx: number, yPx: number) => {
    previewObj.position.set(toThreeX(xPx), toThreeY(yPx), PREVIEW_Z)
    previewObj.rotation.z = 0
  }
  // initialize preview placement once
  previewObjUpdate(previewX, computeSpawnY())

  raf = requestAnimationFrame(animate)

  /* ---------------- resize ---------------- */
  const onResize = () => {
    const w = Math.max(320, stageWrap.clientWidth || window.innerWidth || 800)
    const h = Math.max(480, stageWrap.clientHeight || window.innerHeight || 720)
    width = w
    height = h

    renderer.setSize(width, height)
    renderer.setPixelRatio(Math.min(isCoarsePointer.value ? 1.4 : 2, window.devicePixelRatio || 1))
    composer.setSize(width, height)

    camera.aspect = width / height
    camera.updateProjectionMatrix()

    try { Composite.remove(engine.world, ground) } catch {}
    ground = Bodies.rectangle(width / 2, height + groundThickness / 2, width * 16, groundThickness, { isStatic: true })
    Composite.add(engine.world, [ground])

    ;(floor.geometry as any)?.dispose?.()
    floor.geometry = new THREE.PlaneGeometry((width * 16) * S, 52)

    bloom.setSize(width, height)
    previewX = clamp(previewX, 36, width - 36)
  }
  window.addEventListener('resize', onResize)

  const vv = (window as any).visualViewport
  const onVV = () => onResize()
  if (vv?.addEventListener) vv.addEventListener('resize', onVV)

  /* ---------------- cleanup ---------------- */
  cleanup = () => {
    window.removeEventListener('resize', onResize)
    if (vv?.removeEventListener) vv.removeEventListener('resize', onVV)

    stageWrap.removeEventListener('pointerdown', onStagePointerDown as any)
    window.removeEventListener('keydown', onKeyDown as any)

    Events.off(engine, 'collisionStart', onCollisionStart)

    cancelAnimationFrame(raf)

    for (const [, obj] of meshById) {
      scene.remove(obj)
      try { ;(obj as any).__dispose?.() } catch {}
    }
    meshById.clear()

    scene.remove(previewObj)
    try { ;(previewObj as any).__dispose?.() } catch {}

    scene.remove(floor)
    ;(floor.geometry as any)?.dispose?.()
    ;(floor.material as any)?.dispose?.()

    scene.remove(grid)

    ;(composer as any)?.dispose?.()
    renderer.dispose()
    mount.replaceChildren()
  }
})

onBeforeUnmount(() => cleanup?.())
</script>

<template>
  <section class="fsRoot">
    <div ref="stageWrapEl" class="stageWrap">
      <div ref="mountEl" class="mount" />

      <header class="topbar">
        <div class="brand">
          <div class="logoDot" />
          <div>
            <div class="title">Stack &amp; Balance 3D — Pink Synthwave</div>
            <div class="subtitle">
              <span v-if="state === 'idle'">Tap to start • Space/Enter</span>
              <span v-else-if="state === 'playing'">Tap to drop • Touch ground = lose</span>
              <span v-else>Tap to restart</span>
            </div>
          </div>
        </div>

        <div class="hudRow">
          <button class="pillBtn" :disabled="state === 'playing'" @click.stop="mode = mode === 'daily' ? 'endless' : 'daily'">
            <span class="pillDot" />
            {{ mode === 'daily' ? 'Daily Seed' : 'Endless' }}
          </button>

          <div class="chip"><div class="k">Score</div><div class="v">{{ score }}</div></div>
          <div class="chip" :class="{ pop: comboPop }"><div class="k">Combo</div><div class="v">{{ combo }}</div></div>
          <div class="chip"><div class="k">Best</div><div class="v">{{ best }}</div></div>
        </div>
      </header>

      <div v-if="state === 'playing'" class="meterCard">
        <div class="meterTop">
          <div class="meterLabel">Alignment</div>
          <div class="meterMsg" :class="{ good: lastPerfect }">
            <span v-if="lastDxPx == null">—</span>
            <span v-else>{{ lastDropMsg }}</span>
          </div>
        </div>
        <div class="meterBar"><div class="meterFill" :style="{ width: meterPct + '%' }" /></div>
        <div class="meterFoot">
          <span class="muted">Δ</span>
          <span v-if="lastDxPx == null">—</span>
          <span v-else>{{ Math.round(lastDxPx) }}px</span>
        </div>
      </div>

      <button
        v-if="state === 'playing' && isCoarsePointer"
        class="dropBtn"
        type="button"
        @click.stop.prevent="startOrDrop()"
      >
        DROP
      </button>

      <div v-if="state !== 'playing'" class="overlay">
        <div class="overlayCard">
          <div class="overlayTitle">{{ state === 'idle' ? 'Ready?' : 'Game Over' }}</div>
          <div class="overlayText">
            <template v-if="state === 'idle'">Neon glow enabled on all pieces. Tap anywhere to start.</template>
            <template v-else>
              <div class="reason">Reason: <b>{{ gameOverReason || 'Touched surface' }}</b></div>
              <div class="summary">
                <div class="sumChip"><div class="k">Score</div><div class="v">{{ score }}</div></div>
                <div class="sumChip"><div class="k">Best</div><div class="v">{{ best }}</div></div>
                <div class="sumChip"><div class="k">Placed</div><div class="v">{{ placed }}</div></div>
              </div>
              <div class="cta">Tap to restart</div>
            </template>
          </div>
          <button class="primaryBtn" type="button" @click.stop.prevent="startOrDrop()">
            {{ state === 'idle' ? 'START' : 'RESTART' }}
          </button>
        </div>
      </div>
    </div>
  </section>
</template>

<style scoped>
:global(html),
:global(body),
:global(#__nuxt) { height: 100%; }
:global(body) { margin: 0; overflow: hidden; background: #070012; }

.fsRoot { position: fixed; inset: 0; width: 100vw; height: 100dvh; background: #070012; color: white; }
.stageWrap { position: absolute; inset: 0; overflow: hidden; touch-action: none; }
.mount { position: absolute; inset: 0; z-index: 1; }

.topbar {
  position: absolute; left: 0; right: 0; top: env(safe-area-inset-top);
  z-index: 5; display: flex; align-items: center; justify-content: space-between; gap: 14px;
  padding: 14px 16px;
  background: linear-gradient(to bottom, rgba(255, 43, 214, 0.14), rgba(0, 0, 0, 0.12));
  border-bottom: 1px solid rgba(255,255,255,0.10);
  backdrop-filter: blur(14px);
}

.brand { display: flex; align-items: center; gap: 12px; min-width: 240px; }
.logoDot {
  width: 14px; height: 14px; border-radius: 999px;
  background: radial-gradient(circle at 30% 30%, rgba(255,255,255,0.9), rgba(255,43,214,0.35));
  box-shadow: 0 0 0 1px rgba(255,255,255,0.18), 0 0 24px rgba(255,43,214,0.35);
}
.title { font-weight: 950; letter-spacing: 0.2px; font-size: 16px; }
.subtitle { font-size: 12px; opacity: 0.78; margin-top: 2px; }

.hudRow { display: flex; align-items: center; flex-wrap: wrap; gap: 10px; justify-content: flex-end; }
.pillBtn {
  display: inline-flex; align-items: center; gap: 8px;
  font-size: 12px; padding: 8px 10px; border-radius: 999px;
  background: rgba(255, 43, 214, 0.08);
  border: 1px solid rgba(255, 43, 214, 0.20);
  color: white; cursor: pointer;
}
.pillBtn:disabled { opacity: 0.45; cursor: not-allowed; }
.pillDot { width: 8px; height: 8px; border-radius: 999px; background: rgba(255, 43, 214, 0.9); box-shadow: 0 0 16px rgba(255, 43, 214, 0.55); }

.chip {
  display: grid; gap: 3px; padding: 8px 10px; min-width: 92px;
  border-radius: 14px; background: rgba(255,255,255,0.05);
  border: 1px solid rgba(255,43,214,0.14);
}
.chip .k { font-size: 11px; opacity: 0.72; }
.chip .v { font-size: 16px; font-weight: 950; line-height: 1; }
.chip.pop { animation: pop 180ms ease-out; }
@keyframes pop { 0%{transform:scale(1)} 60%{transform:scale(1.06)} 100%{transform:scale(1)} }

.meterCard {
  position: absolute; left: 14px; bottom: calc(14px + env(safe-area-inset-bottom));
  z-index: 5; width: 280px; padding: 12px; border-radius: 16px;
  background: rgba(0,0,0,0.42);
  border: 1px solid rgba(255,43,214,0.18);
  backdrop-filter: blur(14px); pointer-events: none;
}
.meterTop { display: flex; align-items: baseline; justify-content: space-between; gap: 10px; }
.meterLabel { font-size: 11px; opacity: 0.78; }
.meterMsg { font-size: 12px; opacity: 0.9; }
.meterMsg.good { opacity: 1; font-weight: 900; color: rgba(255, 210, 255, 1); text-shadow: 0 0 14px rgba(255, 43, 214, 0.35); }
.meterBar { margin-top: 8px; height: 10px; border-radius: 999px; background: rgba(255,255,255,0.10); overflow: hidden; }
.meterFill { height: 100%; border-radius: 999px; background: rgba(255, 43, 214, 0.85); transition: width 120ms linear; box-shadow: 0 0 18px rgba(255,43,214,0.35); }
.meterFoot { margin-top: 8px; font-size: 12px; opacity: 0.88; display: flex; gap: 6px; align-items: center; }
.muted { opacity: 0.7; }

.dropBtn {
  position: absolute; right: 14px; bottom: calc(14px + env(safe-area-inset-bottom));
  z-index: 6; padding: 14px 16px; border-radius: 16px;
  background: rgba(255, 43, 214, 0.12);
  border: 1px solid rgba(255, 43, 214, 0.25);
  color: white; font-weight: 950; letter-spacing: 0.5px;
  backdrop-filter: blur(14px);
  box-shadow: 0 18px 40px rgba(0,0,0,0.45), 0 0 28px rgba(255,43,214,0.22);
}

.overlay { position: absolute; inset: 0; z-index: 10; display: grid; place-items: center; padding: 16px; }
.overlayCard {
  width: min(720px, 92vw); padding: 18px 16px; border-radius: 18px;
  background: rgba(0,0,0,0.62);
  border: 1px solid rgba(255,43,214,0.20);
  backdrop-filter: blur(16px);
  box-shadow: 0 28px 90px rgba(0,0,0,0.65), 0 0 40px rgba(255,43,214,0.08);
  text-align: center;
}
.overlayTitle { font-weight: 950; font-size: 22px; letter-spacing: 0.2px; }
.overlayText { margin-top: 10px; font-size: 13px; opacity: 0.92; line-height: 1.45; }
.reason { margin-bottom: 10px; }
.summary { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 10px; margin: 12px 0 10px; }
.sumChip { border-radius: 14px; padding: 10px; background: rgba(255,255,255,0.06); border: 1px solid rgba(255,43,214,0.16); }
.sumChip .k { font-size: 11px; opacity: 0.7; }
.sumChip .v { font-size: 18px; font-weight: 950; margin-top: 2px; }
.cta { margin-top: 4px; font-size: 12px; opacity: 0.84; }
.primaryBtn {
  margin-top: 12px; padding: 12px 14px; border-radius: 14px;
  background: rgba(255, 43, 214, 0.14);
  border: 1px solid rgba(255, 43, 214, 0.28);
  color: white; font-weight: 950; letter-spacing: 0.6px;
  box-shadow: 0 0 24px rgba(255,43,214,0.14);
}

@media (max-width: 680px) {
  .chip { min-width: 78px; }
  .summary { grid-template-columns: 1fr; }
  .meterCard { width: 240px; }
  .topbar { padding: 12px 12px; }
}
</style>
