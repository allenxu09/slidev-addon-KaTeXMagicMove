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
}>(), {
  duration: 500,
  easing: 'cubic-bezier(0.16, 1, 0.3, 1)',
  displayMode: true,
  fontSize: '1.6rem',
})

const emit = defineEmits<{
  (e: 'update:step', value: number): void
}>()

const internal = ref(0)
const container = ref<HTMLElement | null>(null)
const overlay = ref<HTMLElement | null>(null)
const hasRendered = ref(false)

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
  'mord', 'mop', 'mbin', 'mrel', 'mopen', 'mclose',
  'mtext', 'mspace', 'msupsub', 'frac-line', 'sqrt-sign', 'sqrt-line',
]
const atomSelector = atomClasses.map(c => `.katex-html .${c}`).join(', ')
const leafSelector = atomClasses.map(c => `.${c}`).join(', ')

const sigCounts = new Map<string, number>()

type Atom = {
  el: HTMLElement
  sig: string
  text: string
  id: string
}

function normalizeText(text: string) {
  return text.replace(/\s+/g, ' ').trim()
}

function signature(el: HTMLElement) {
  const tag = el.tagName.toLowerCase()
  const cls = Array.from(el.classList)
    .filter(c => !c.startsWith('katex'))
    .sort()
    .join('.')
  const text = normalizeText(el.textContent || '')
  return `${tag}|${cls}|${text}`
}

function collectAtoms(root: HTMLElement): Atom[] {
  const all = Array.from(root.querySelectorAll<HTMLElement>(atomSelector))
  const leafs = all.filter(el => !el.querySelector(leafSelector))
  return leafs.map(el => ({
    el,
    sig: signature(el),
    text: normalizeText(el.textContent || ''),
    id: el.dataset.kid || '',
  }))
}

function parseSuffix(id: string) {
  const m = id.match(/#(\d+)$/)
  return m ? Number(m[1]) : -1
}

function matchAtoms(prev: Atom[], next: Atom[]) {
  const prevMap = new Map<string, number[]>()
  prev.forEach((atom, i) => {
    if (!prevMap.has(atom.sig))
      prevMap.set(atom.sig, [])
    prevMap.get(atom.sig)!.push(i)
  })

  const pairs: Array<[number, number]> = []
  next.forEach((atom, j) => {
    const indices = prevMap.get(atom.sig)
    if (indices && indices.length) {
      const i = indices.shift()!
      pairs.push([i, j])
      console.log(`Matched atom ${atom.sig} (prev #${i} to next #${j})`)
    }
  })
  return pairs
}

function assignIds(prevAtoms: Atom[], host: HTMLElement) {
  const nextAtoms = collectAtoms(host)

  sigCounts.clear()
  // Seed counts from previous IDs to keep numbering stable
  prevAtoms.forEach((atom) => {
    const n = parseSuffix(atom.id)
    const prevMax = sigCounts.get(atom.sig) ?? 0
    sigCounts.set(atom.sig, Math.max(prevMax, n + 1))
  })

  const pairs = matchAtoms(prevAtoms, nextAtoms)
  const taken = new Set<string>()

  pairs.forEach(([pi, ni]) => {
    const id = prevAtoms[pi].id || `${prevAtoms[pi].sig}#${sigCounts.get(prevAtoms[pi].sig) ?? 0}`
    nextAtoms[ni].el.dataset.kid = id
    taken.add(id)
  })

  nextAtoms.forEach((atom) => {
    if (atom.el.dataset.kid)
      return
    const count = sigCounts.get(atom.sig) ?? 0
    const candidate = `${atom.sig}#${count}`
    atom.el.dataset.kid = taken.has(candidate) ? `${atom.sig}#${count + 1}` : candidate
    sigCounts.set(atom.sig, count + 1)
  })

  return nextAtoms
}

function measure(root: HTMLElement) {
  const map = new Map<string, DOMRect>()
  root.querySelectorAll<HTMLElement>('[data-kid]').forEach((el) => {
    map.set(el.dataset.kid!, el.getBoundingClientRect())
  })
  return map
}

function renderLatex(target: HTMLElement, latex: string) {
  katex.render(latex, target, {
    throwOnError: false,
    trust: true,
    displayMode: props.displayMode,
    strict: 'ignore',
  })
}

function animate(nextLatex: string) {
  const host = container.value
  if (!host)
    return

  const prevRectMap = hasRendered.value ? measure(host) : new Map<string, DOMRect>()
  const prevSnapshot = hasRendered.value ? host.cloneNode(true) as HTMLElement : null
  const prevAtoms = hasRendered.value ? collectAtoms(host) : []
  const leavingNodes = new Set(prevRectMap.keys())

  // Render new
  renderLatex(host, nextLatex)
  const nextAtoms = assignIds(prevAtoms, host)

  nextTick(() => {
    const nextRectMap = measure(host)
    const overlayRect = overlay.value?.getBoundingClientRect()

    // Animate enter & move
    nextAtoms.forEach(({ el }) => {
      const id = el.dataset.kid!
      const nextRect = nextRectMap.get(id)
      const prevRect = prevRectMap.get(id)
      leavingNodes.delete(id)

      el.style.willChange = 'transform, opacity'
      el.style.transition = 'none'

      if (prevRect) {
        const dx = prevRect.left - nextRect!.left
        const dy = prevRect.top - nextRect!.top
        const sx = prevRect.width / (nextRect!.width || 1)
        const sy = prevRect.height / (nextRect!.height || 1)
        el.style.transformOrigin = '0 0'
        el.style.transform = `translate(${dx}px, ${dy}px) scale(${sx}, ${sy})`
      }
      else {
        el.style.opacity = '0'
        el.style.transform = 'scale(0.9)'
      }

      requestAnimationFrame(() => {
        el.style.transition = `transform ${props.duration}ms ${props.easing}, opacity ${props.duration}ms ${props.easing}`
        el.style.transform = ''
        el.style.opacity = '1'
      })
    })

    // Animate leaving atoms using overlay clones
    if (leavingNodes.size && overlay.value && prevSnapshot && overlayRect) {
      leavingNodes.forEach((id) => {
        const rect = prevRectMap.get(id)!
        const original = prevSnapshot.querySelector<HTMLElement>(`[data-kid="${id}"]`)
        const clone = original?.cloneNode(true) as HTMLElement | null
        if (!clone)
          return
        clone.style.position = 'absolute'
        clone.style.left = `${rect.left - overlayRect.left}px`
        clone.style.top = `${rect.top - overlayRect.top}px`
        clone.style.transformOrigin = 'center center'
        clone.style.transition = `transform ${props.duration}ms ${props.easing}, opacity ${props.duration}ms ${props.easing}`
        clone.style.opacity = '1'
        clone.style.pointerEvents = 'none'
        overlay.value!.appendChild(clone)
        requestAnimationFrame(() => {
          clone.style.opacity = '0'
          clone.style.transform = 'scale(0.9) translate(0, 6px)'
          setTimeout(() => clone.remove(), props.duration + 32)
        })
      })
    }

    hasRendered.value = true
  })
}

onMounted(() => {
  if (container.value)
    renderLatex(container.value, currentLatex.value)
  nextTick(() => {
    assignIds([], container.value!)
    hasRendered.value = true
  })
})

watch(currentLatex, (next) => animate(next))
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
}

.katex-move__host :deep(.katex) {
  transition: none;
  will-change: transform;
}

.katex-move__host :deep([data-kid]) {
  display: inline-block;
}

.katex-move__overlay {
  position: absolute;
  inset: 0;
  pointer-events: none;
}
</style>
