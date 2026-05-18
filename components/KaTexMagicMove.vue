<script setup lang="ts">
import { computed, nextTick, onMounted, ref, watch } from 'vue'
import katex from 'katex'
import 'katex/dist/katex.min.css'

const props = withDefaults(defineProps<{
  steps: string[]
  step?: number
  duration?: number
  easing?: string
  displayMode?: boolean
  fontSize?: string
  stagger?: number
}>(), {
  duration: 650,
  easing: 'cubic-bezier(0.22, 1, 0.36, 1)',
  displayMode: true,
  fontSize: '1.6rem',
  stagger: 8,
})

const emit = defineEmits<{
  (e: 'update:step', value: number): void
}>()

type Atom = {
  el: HTMLElement
  key: string
  text: string
  id: string
  index: number
}

type AtomMatch = {
  prevIndex: number
  nextIndex: number
}

const internal = ref(0)
const container = ref<HTMLElement | null>(null)
const overlay = ref<HTMLElement | null>(null)
const hasRendered = ref(false)

let idSerial = 0
let animationRun = 0
const activeAnimations = new Set<Animation>()

const currentStep = computed({
  get: () => (props.step ?? internal.value),
  set: (value: number) => {
    if (props.step === undefined)
      internal.value = value
    emit('update:step', value)
  },
})

const currentLatex = computed(() => props.steps[Math.min(props.steps.length - 1, Math.max(0, currentStep.value))] ?? '')

const atomClasses = [
  'mord',
  'mop',
  'mbin',
  'mrel',
  'mopen',
  'mclose',
  'mpunct',
  'minner',
  'mtext',
  'mspace',
  'msupsub',
  'frac-line',
  'sqrt-sign',
  'sqrt-line',
  'overline-line',
  'underline-line',
  'accent-body',
]

const atomSelector = atomClasses.map(c => `.katex-html .${c}`).join(', ')
const leafSelector = atomClasses.map(c => `.${c}`).join(', ')

function normalizeText(text: string) {
  return text.replace(/\s+/g, ' ').trim()
}

function atomRole(el: HTMLElement) {
  return atomClasses.find(c => el.classList.contains(c)) ?? el.tagName.toLowerCase()
}

function atomKey(el: HTMLElement) {
  const role = atomRole(el)
  const text = normalizeText(el.textContent || '')
  const mathClass = Array.from(el.classList)
    .filter(c => c !== role && !c.startsWith('katex'))
    .sort()
    .join('.')

  return `${role}|${mathClass}|${text}`
}

function isRenderableAtom(el: HTMLElement) {
  const rect = el.getBoundingClientRect()
  const hasVisibleBox = rect.width > 0.2 && rect.height > 0.2
  const isStructuralLine = el.classList.contains('frac-line')
    || el.classList.contains('sqrt-line')
    || el.classList.contains('overline-line')
    || el.classList.contains('underline-line')

  return hasVisibleBox || isStructuralLine
}

function collectAtoms(root: HTMLElement): Atom[] {
  const all = Array.from(root.querySelectorAll<HTMLElement>(atomSelector))
  const leafs = all.filter(el => !el.querySelector(leafSelector) && isRenderableAtom(el))

  return leafs.map((el, index) => ({
    el,
    key: atomKey(el),
    text: normalizeText(el.textContent || ''),
    id: el.dataset.katexMoveId || '',
    index,
  }))
}

function nextId() {
  return `km-${idSerial++}`
}

function seedIdSerial(atoms: Atom[]) {
  atoms.forEach((atom) => {
    const match = atom.id.match(/^km-(\d+)$/)
    if (match)
      idSerial = Math.max(idSerial, Number(match[1]) + 1)
  })
}

function ensureAtomIds(atoms: Atom[]) {
  seedIdSerial(atoms)
  atoms.forEach((atom) => {
    if (!atom.id) {
      atom.id = nextId()
      atom.el.dataset.katexMoveId = atom.id
    }
  })
}

function measureAtoms(atoms: Atom[]) {
  const map = new Map<string, DOMRect>()
  atoms.forEach((atom) => {
    if (atom.id)
      map.set(atom.id, atom.el.getBoundingClientRect())
  })
  return map
}

function measureAtomList(atoms: Atom[]) {
  return atoms.map(atom => atom.el.getBoundingClientRect())
}

function centerDistance(a: DOMRect, b: DOMRect) {
  const ax = a.left + a.width / 2
  const ay = a.top + a.height / 2
  const bx = b.left + b.width / 2
  const by = b.top + b.height / 2
  return Math.hypot(ax - bx, ay - by)
}

function sizeDelta(a: DOMRect, b: DOMRect) {
  return Math.abs(a.width - b.width) + Math.abs(a.height - b.height)
}

function matchAtoms(prevAtoms: Atom[], nextAtoms: Atom[], prevRects: Map<string, DOMRect>, nextRects: DOMRect[]) {
  const candidates: Array<AtomMatch & { score: number }> = []

  prevAtoms.forEach((prevAtom, prevIndex) => {
    const prevRect = prevRects.get(prevAtom.id)
    if (!prevRect)
      return

    nextAtoms.forEach((nextAtom, nextIndex) => {
      if (prevAtom.key !== nextAtom.key)
        return

      const nextRect = nextRects[nextIndex]
      const orderPenalty = Math.abs(prevAtom.index - nextAtom.index) * 1.5
      const travelPenalty = centerDistance(prevRect, nextRect) * 0.55
      const shapePenalty = sizeDelta(prevRect, nextRect) * 0.35

      candidates.push({
        prevIndex,
        nextIndex,
        score: orderPenalty + travelPenalty + shapePenalty,
      })
    })
  })

  candidates.sort((a, b) => a.score - b.score)

  const usedPrev = new Set<number>()
  const usedNext = new Set<number>()
  const matches: AtomMatch[] = []

  candidates.forEach((candidate) => {
    if (usedPrev.has(candidate.prevIndex) || usedNext.has(candidate.nextIndex))
      return

    usedPrev.add(candidate.prevIndex)
    usedNext.add(candidate.nextIndex)
    matches.push(candidate)
  })

  return matches
}

function renderLatex(target: HTMLElement, latex: string) {
  katex.render(latex, target, {
    throwOnError: false,
    trust: true,
    displayMode: props.displayMode,
    strict: 'ignore',
  })
}

function prefersReducedMotion() {
  return typeof window !== 'undefined'
    && window.matchMedia?.('(prefers-reduced-motion: reduce)').matches
}

function trackAnimation(animation: Animation) {
  activeAnimations.add(animation)

  const cleanup = () => activeAnimations.delete(animation)
  animation.addEventListener('finish', cleanup, { once: true })
  animation.addEventListener('cancel', cleanup, { once: true })

  return animation
}

function cancelAnimations() {
  activeAnimations.forEach(animation => animation.cancel())
  activeAnimations.clear()
}

function clearOverlay() {
  if (overlay.value)
    overlay.value.innerHTML = ''
}

function animateTransform(el: HTMLElement, keyframes: Keyframe[], delay = 0) {
  el.style.willChange = 'transform, opacity, filter'
  el.style.transformOrigin = 'top left'

  const animation = el.animate(keyframes, {
    duration: props.duration,
    easing: props.easing,
    delay,
    fill: 'both',
  })

  trackAnimation(animation)
  animation.finished
    .then(() => {
      el.style.willChange = ''
      el.style.transformOrigin = ''
    })
    .catch(() => {})
}

function animateEnteringAtom(atom: Atom, index: number) {
  const delay = Math.min(index * props.stagger, props.duration * 0.22)

  animateTransform(atom.el, [
    {
      opacity: 0,
      transform: 'translate3d(0, 0.16em, 0) scale(0.92)',
      filter: 'blur(2px)',
    },
    {
      opacity: 1,
      transform: 'translate3d(0, 0, 0) scale(1)',
      filter: 'blur(0)',
    },
  ], delay)
}

function animateMatchedAtom(atom: Atom, prevRect: DOMRect, nextRect: DOMRect, index: number) {
  const dx = prevRect.left - nextRect.left
  const dy = prevRect.top - nextRect.top
  const sx = prevRect.width / Math.max(nextRect.width, 1)
  const sy = prevRect.height / Math.max(nextRect.height, 1)
  const distance = Math.hypot(dx, dy)
  const delay = Math.min(index * props.stagger * 0.5, props.duration * 0.16)

  animateTransform(atom.el, [
    {
      opacity: distance > 1 ? 0.92 : 1,
      transform: `translate3d(${dx}px, ${dy}px, 0) scale(${sx}, ${sy})`,
      filter: distance > 24 ? 'blur(0.4px)' : 'blur(0)',
    },
    {
      opacity: 1,
      transform: 'translate3d(0, 0, 0) scale(1)',
      filter: 'blur(0)',
    },
  ], delay)
}

function animateLeavingAtom(id: string, prevSnapshot: HTMLElement, prevRect: DOMRect, overlayRect: DOMRect, index: number) {
  if (!overlay.value)
    return

  const original = prevSnapshot.querySelector<HTMLElement>(`[data-katex-move-id="${id}"]`)
  const clone = original?.cloneNode(true) as HTMLElement | null
  if (!clone)
    return

  clone.removeAttribute('data-katex-move-id')
  clone.style.position = 'absolute'
  clone.style.left = `${prevRect.left - overlayRect.left}px`
  clone.style.top = `${prevRect.top - overlayRect.top}px`
  clone.style.width = `${prevRect.width}px`
  clone.style.height = `${prevRect.height}px`
  clone.style.margin = '0'
  clone.style.display = 'inline-block'
  clone.style.pointerEvents = 'none'
  clone.style.transformOrigin = 'top left'
  clone.style.willChange = 'transform, opacity, filter'

  overlay.value.appendChild(clone)

  const delay = Math.min(index * props.stagger * 0.4, props.duration * 0.12)
  const animation = clone.animate([
    {
      opacity: 1,
      transform: 'translate3d(0, 0, 0) scale(1)',
      filter: 'blur(0)',
    },
    {
      opacity: 0,
      transform: 'translate3d(0, 0.22em, 0) scale(0.94)',
      filter: 'blur(2px)',
    },
  ], {
    duration: props.duration,
    easing: props.easing,
    delay,
    fill: 'both',
  })

  trackAnimation(animation)
  animation.finished
    .then(() => clone.remove())
    .catch(() => clone.remove())
}

async function animate(nextLatex: string) {
  const host = container.value
  if (!host)
    return

  const run = ++animationRun
  const prevAtoms = hasRendered.value ? collectAtoms(host) : []
  ensureAtomIds(prevAtoms)

  const prevRects = hasRendered.value ? measureAtoms(prevAtoms) : new Map<string, DOMRect>()
  const prevSnapshot = hasRendered.value ? host.cloneNode(true) as HTMLElement : null
  const leavingIds = new Set(prevRects.keys())

  cancelAnimations()
  clearOverlay()
  renderLatex(host, nextLatex)

  await nextTick()
  if (run !== animationRun)
    return

  const nextAtoms = collectAtoms(host)
  const nextRects = measureAtomList(nextAtoms)
  const matches = matchAtoms(prevAtoms, nextAtoms, prevRects, nextRects)
  const matchedNext = new Set<number>()

  matches.forEach(({ prevIndex, nextIndex }) => {
    const id = prevAtoms[prevIndex].id
    nextAtoms[nextIndex].id = id
    nextAtoms[nextIndex].el.dataset.katexMoveId = id
    leavingIds.delete(id)
    matchedNext.add(nextIndex)
  })

  nextAtoms.forEach((atom) => {
    if (!atom.id) {
      atom.id = nextId()
      atom.el.dataset.katexMoveId = atom.id
    }
  })

  if (prefersReducedMotion() || props.duration <= 0) {
    hasRendered.value = true
    return
  }

  const overlayRect = overlay.value?.getBoundingClientRect()

  matches.forEach(({ prevIndex, nextIndex }, order) => {
    const atom = nextAtoms[nextIndex]
    const prevRect = prevRects.get(prevAtoms[prevIndex].id)
    const nextRect = nextRects[nextIndex]

    if (prevRect && nextRect)
      animateMatchedAtom(atom, prevRect, nextRect, order)
  })

  nextAtoms.forEach((atom, index) => {
    if (!matchedNext.has(index))
      animateEnteringAtom(atom, index)
  })

  if (prevSnapshot && overlayRect) {
    Array.from(leavingIds).forEach((id, index) => {
      const prevRect = prevRects.get(id)
      if (prevRect)
        animateLeavingAtom(id, prevSnapshot, prevRect, overlayRect, index)
    })
  }

  hasRendered.value = true
}

onMounted(async () => {
  if (container.value)
    renderLatex(container.value, currentLatex.value)

  await nextTick()

  if (!container.value)
    return

  const atoms = collectAtoms(container.value)
  ensureAtomIds(atoms)
  hasRendered.value = true
})

watch(currentLatex, next => animate(next))
</script>

<template>
  <div class="katex-move" :style="{ fontSize: props.fontSize }">
    <div ref="overlay" class="katex-move__overlay" aria-hidden="true" />
    <div ref="container" class="katex-move__host" />
  </div>
</template>

<style scoped>
.katex-move {
  position: relative;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  line-height: 1.35;
  isolation: isolate;
}

.katex-move__host {
  position: relative;
  z-index: 1;
}

.katex-move__host :deep(.katex) {
  transition: none;
}

.katex-move__host :deep([data-katex-move-id]) {
  display: inline-block;
  backface-visibility: hidden;
  transform-box: border-box;
}

.katex-move__overlay {
  position: fixed;
  inset: 0;
  z-index: 2;
  overflow: visible;
  pointer-events: none;
}

.katex-move__overlay :deep([data-katex-move-id]) {
  display: inline-block;
}
</style>
