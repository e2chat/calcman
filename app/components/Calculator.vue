<script setup lang="ts">
import { ref, computed, onMounted, onBeforeUnmount } from 'vue'

const displayPrimary = ref('0')
const displaySecondary = ref('')
const expression = ref<string>('')
const memory = ref<number | null>(null)
const hasError = ref(false)

const maxDigits = 16

function resetAll() {
  expression.value = ''
  displayPrimary.value = '0'
  displaySecondary.value = ''
  hasError.value = false
}

function clearEntry() {
  if (hasError.value) return resetAll()
  displayPrimary.value = '0'
}

function backspace() {
  if (hasError.value) return
  if (displayPrimary.value.length <= 1 || (displayPrimary.value.startsWith('-') && displayPrimary.value.length === 2)) {
    displayPrimary.value = '0'
  } else {
    displayPrimary.value = displayPrimary.value.slice(0, -1)
  }
}

function inputDigit(d: string) {
  if (hasError.value) return
  if (displayPrimary.value.replace('-', '').replace('.', '').length >= maxDigits) return
  if (displayPrimary.value === '0') {
    displayPrimary.value = d
  } else if (displayPrimary.value === '-0') {
    displayPrimary.value = '-' + d
  } else {
    displayPrimary.value += d
  }
}

function inputDot() {
  if (hasError.value) return
  if (!displayPrimary.value.includes('.')) {
    displayPrimary.value += (displayPrimary.value === '' ? '0.' : '.')
  }
}

function toggleSign() {
  if (hasError.value) return
  if (displayPrimary.value.startsWith('-')) {
    displayPrimary.value = displayPrimary.value.slice(1)
  } else if (displayPrimary.value !== '0') {
    displayPrimary.value = '-' + displayPrimary.value
  }
}

function percent() {
  if (hasError.value) return
  const n = Number(displayPrimary.value)
  if (!Number.isFinite(n)) return
  displayPrimary.value = String(n / 100)
}

function appendOperator(op: string) {
  if (hasError.value) return
  const cur = displayPrimary.value
  if (expression.value && /[+\-×÷]$/.test(expression.value.trim())) {
    expression.value = expression.value.replace(/[+\-×÷]$/, op)
  } else {
    expression.value = expression.value ? `${expression.value} ${cur} ${op}` : `${cur} ${op}`
  }
  displayPrimary.value = '0'
  displaySecondary.value = expression.value
}

function computeEquals() {
  if (hasError.value) return
  const expr = (expression.value ? `${expression.value} ${displayPrimary.value}` : displayPrimary.value).trim()
  try {
    const val = evaluate(expr)
    displaySecondary.value = expr + ' ='
    displayPrimary.value = formatNumber(val)
    expression.value = ''
  } catch (e) {
    hasError.value = true
    displayPrimary.value = 'Error'
  }
}

function formatNumber(n: number): string {
  if (!Number.isFinite(n)) return 'Error'
  const abs = Math.abs(n)
  if (abs !== 0 && (abs < 1e-6 || abs >= 1e12)) return n.toExponential(8)
  let s = n.toString()
  if (s.includes('e')) return n.toExponential(8)
  if (s.length > 18) s = n.toPrecision(16)
  return s
}

function evaluate(expr: string): number {
  const tokens = tokenize(expr)
  const rpn = toRPN(tokens)
  return evalRPN(rpn)
}

type Tok = { type: 'num' | 'op' | 'lpar' | 'rpar', v: string }

function tokenize(s: string): Tok[] {
  const out: Tok[] = []
  let i = 0
  while (i < s.length) {
    const ch = s[i]
    if (ch === ' ') { i++; continue }
    if (ch === '(') { out.push({ type: 'lpar', v: ch }); i++; continue }
    if (ch === ')') { out.push({ type: 'rpar', v: ch }); i++; continue }
    if ('+-×÷*/'.includes(ch)) {
      let op = ch
      if (ch === '*') op = '×'
      if (ch === '/') op = '÷'
      out.push({ type: 'op', v: op }); i++; continue
    }
    if (/[0-9.]/.test(ch)) {
      let j = i
      let dot = 0
      while (j < s.length && /[0-9.]/.test(s[j])) { if (s[j] === '.') dot++; if (dot > 1) break; j++ }
      out.push({ type: 'num', v: s.slice(i, j) })
      i = j
      continue
    }
    throw new Error('bad token')
  }
  // handle unary minus
  const norm: Tok[] = []
  for (let k = 0; k < out.length; k++) {
    const t = out[k]
    if (t.type === 'op' && t.v === '-' && (k === 0 || ['op','lpar'].includes(out[k-1].type))) {
      // convert "-x" to "0 - x"
      norm.push({ type: 'num', v: '0' })
      norm.push(t)
    } else norm.push(t)
  }
  return norm
}

const prec: Record<string, number> = { '+':1, '-':1, '×':2, '÷':2 }

function toRPN(tokens: Tok[]): Tok[] {
  const out: Tok[] = []
  const stack: Tok[] = []
  for (const t of tokens) {
    if (t.type === 'num') out.push(t)
    else if (t.type === 'op') {
      while (stack.length && stack[stack.length-1].type === 'op' && prec[stack[stack.length-1].v] >= prec[t.v]) {
        out.push(stack.pop()!)
      }
      stack.push(t)
    } else if (t.type === 'lpar') stack.push(t)
    else if (t.type === 'rpar') {
      while (stack.length && stack[stack.length-1].type !== 'lpar') out.push(stack.pop()!)
      stack.pop()
    }
  }
  while (stack.length) out.push(stack.pop()!)
  return out
}

function evalRPN(rpn: Tok[]): number {
  const st: number[] = []
  for (const t of rpn) {
    if (t.type === 'num') st.push(Number(t.v))
    else if (t.type === 'op') {
      const b = st.pop()!, a = st.pop()!
      let r = 0
      if (t.v === '+') r = a + b
      else if (t.v === '-') r = a - b
      else if (t.v === '×') r = a * b
      else if (t.v === '÷') r = b === 0 ? NaN : a / b
      st.push(r)
    }
  }
  return st.pop() ?? NaN
}

function opKey(op: string) {
  appendOperator(op)
}

function handleKey(e: KeyboardEvent) {
  const k = e.key
  if (/^[0-9]$/.test(k)) return inputDigit(k)
  if (k === '.') return inputDot()
  if (k === '+') return opKey('+')
  if (k === '-') return opKey('-')
  if (k === '*') return opKey('×')
  if (k === '/') return opKey('÷')
  if (k === 'Enter' || k === '=') return computeEquals()
  if (k === 'Backspace') return backspace()
  if (k === 'Escape') return resetAll()
}

onMounted(() => {
  window.addEventListener('keydown', handleKey)
})

onBeforeUnmount(() => {
  window.removeEventListener('keydown', handleKey)
})

function memoryClear() { memory.value = null }
function memoryRecall() { if (memory.value != null) displayPrimary.value = formatNumber(memory.value) }
function memoryAdd() { const n = Number(displayPrimary.value); if (Number.isFinite(n)) memory.value = (memory.value ?? 0) + n }
function memorySub() { const n = Number(displayPrimary.value); if (Number.isFinite(n)) memory.value = (memory.value ?? 0) - n }
</script>

<template>
  <div class="min-h-dvh flex items-center justify-center p-4 bg-gradient-to-br from-slate-100 to-slate-200 dark:from-slate-900 dark:to-slate-950">
    <UCard class="w-full max-w-[380px] shadow-2xl/50 backdrop-blur supports-[backdrop-filter]:bg-white/60 dark:supports-[backdrop-filter]:bg-slate-900/60">
      <div class="flex flex-col gap-2">
        <div class="text-right select-none">
          <div class="text-sm text-slate-500 dark:text-slate-400 h-5 truncate" aria-live="polite">{{ displaySecondary }}</div>
          <div class="text-4xl font-semibold tabular-nums h-[44px] overflow-hidden" aria-live="polite">{{ displayPrimary }}</div>
        </div>
        <div class="grid grid-cols-4 gap-2">
          <UButton color="gray" variant="soft" @click="memoryClear" :aria-label="'Memory Clear'">MC</UButton>
          <UButton color="gray" variant="soft" @click="memoryRecall" :aria-label="'Memory Recall'">MR</UButton>
          <UButton color="gray" variant="soft" @click="memoryAdd" :aria-label="'Memory Add'">M+</UButton>
          <UButton color="gray" variant="soft" @click="memorySub" :aria-label="'Memory Subtract'">M-</UButton>

          <UButton color="gray" variant="soft" @click="percent" :aria-label="'Percent'">%</UButton>
          <UButton color="gray" variant="soft" @click="clearEntry" :aria-label="'Clear Entry'">CE</UButton>
          <UButton color="gray" variant="soft" @click="resetAll" :aria-label="'Clear'">C</UButton>
          <UButton color="gray" variant="soft" @click="backspace" :aria-label="'Backspace'">⌫</UButton>

          <UButton @click="opKey('÷')" color="primary" variant="soft">÷</UButton>
          <UButton @click="inputDigit('7')">7</UButton>
          <UButton @click="inputDigit('8')">8</UButton>
          <UButton @click="inputDigit('9')">9</UButton>

          <UButton @click="opKey('×')" color="primary" variant="soft">×</UButton>
          <UButton @click="inputDigit('4')">4</UButton>
          <UButton @click="inputDigit('5')">5</UButton>
          <UButton @click="inputDigit('6')">6</UButton>

          <UButton @click="opKey('-')" color="primary" variant="soft">−</UButton>
          <UButton @click="inputDigit('1')">1</UButton>
          <UButton @click="inputDigit('2')">2</UButton>
          <UButton @click="inputDigit('3')">3</UButton>

          <UButton @click="opKey('+')" color="primary" variant="soft">+</UButton>
          <UButton @click="toggleSign" color="gray" variant="soft">±</UButton>
          <UButton @click="inputDigit('0')">0</UButton>
          <UButton @click="inputDot">.</UButton>

          <UButton class="col-span-4" color="primary" @click="computeEquals">=</UButton>
        </div>
      </div>
    </UCard>
  </div>
</template>

<style scoped>
.tabular-nums { font-variant-numeric: tabular-nums; }
</style>
