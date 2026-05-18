---
theme: default
mdc: true
transition: slide-left
---

<script setup>
import { ref } from 'vue'
import KaTeXMagicMove from '/components/KaTexMagicMove.vue'

const equations = [
	String.raw`\begin{pmatrix}
 1 & 2\\
 3 & 4
\end{pmatrix}`,
    String.raw`\begin{pmatrix}
 3 & 4\\
 1 & 2
\end{pmatrix}`,
]

const step = ref(0)
const next = () => { step.value = Math.min(equations.length - 1, step.value + 1) }
const prev = () => { step.value = Math.max(0, step.value - 1) }
</script>

# KaTeX Magic Move (custom)

用自定义组件让 KaTeX 公式做「magic move」式形变。每次点击切换一步，元素按渲染顺序做 FLIP 动画：

<KaTeXMagicMove
	:steps="equations"
	v-model:step="step"
	:duration="700"
	easing="cubic-bezier(0.33, 1, 0.68, 1)"
	:font-size="'2rem'"
/> 

<div class="mt-6 flex gap-3 items-center">
	<button class="px-3 py-2 rounded border border-gray-400/70 hover:bg-gray-200/40" @click="prev">Prev</button>
	<button class="px-3 py-2 rounded border border-gray-400/70 hover:bg-gray-200/40" @click="next">Next</button>
	<span class="text-sm opacity-70">Step {{ step + 1 }} / {{ equations.length }}</span>
</div>

<v-clicks>
<div class="text-sm opacity-80 mt-6">Tip: 也可以用 `v-click` 或 `$slidev.nav.next()` 驱动 step。</div>
<div class="text-sm opacity-80">渲染节点按顺序匹配，LaTeX 结构越接近，形变越像 Manim。</div>
</v-clicks>
