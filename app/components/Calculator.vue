<script setup lang="ts">
import { ref, computed, onMounted, onBeforeUnmount } from 'vue'

const displayPrimary = ref('0')
const displaySecondary = ref('')
const expression = ref<string>('')
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
  } else {
    if (displayPrimary.value === '0') displayPrimary.value = '-0'
    else displayPrimary.value = '-' + displayPrimary.value
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
    if (/[0-9.\-]/.test(ch)) {
      let j = i
      let dot = 0
      let sawMinus = false
      while (j < s.length && /[0-9.\-]/.test(s[j])) {
        if (s[j] === '.') { dot++; if (dot > 1) break }
        if (s[j] === '-') { if (j !== i) break; sawMinus = true }
        j++
      }
      out.push({ type: 'num', v: s.slice(i, j) })
      i = j
      continue
    }
    throw new Error('bad token')
  }
  // allow unary minus as part of number when typed (handled at input time)
  return out
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
  <div class="min-h-dvh flex items-center justify-center p-6 bg-black text-white">
    <div class="w-full max-w-[380px] rounded-2xl border border-white/10 bg-black/60 backdrop-blur">
      <div class="p-5 flex flex-col gap-4">
        <div class="select-none text-right">
          <div class="text-xs text-white/40 h-4 truncate" aria-live="polite">{{ displaySecondary }}</div>
          <div class="text-5xl font-semibold h-[52px] overflow-hidden tracking-tight" aria-live="polite">{{ displayPrimary }}</div>
        </div>
        <div class="grid grid-cols-4 gap-2">
          <button class="h-11 rounded-lg bg-white/[0.06] hover:bg-white/[0.12] border border-white/10 cursor-pointer" @click="percent" aria-label="Percent">%</button>
          <button class="h-11 rounded-lg bg-white/[0.06] hover:bg-white/[0.12] border border-white/10 cursor-pointer" @click="clearEntry" aria-label="Clear Entry">CE</button>
          <button class="h-11 rounded-lg bg-white/[0.06] hover:bg-white/[0.12] border border-white/10 cursor-pointer" @click="resetAll" aria-label="All Clear">AC</button>
          <button class="h-11 rounded-lg bg-white/[0.06] hover:bg-white/[0.12] border border-white/10 cursor-pointer" @click="backspace" aria-label="Backspace">⌫</button>

          <button class="h-11 rounded-lg bg-orange-500/20 hover:bg-orange-500/30 border border-orange-500/40 text-orange-400 cursor-pointer" @click="opKey('÷')">÷</button>
          <button class="h-11 rounded-lg bg-white/[0.06] hover:bg-white/[0.12] border border-white/10 cursor-pointer" @click="inputDigit('7')">7</button>
          <button class="h-11 rounded-lg bg-white/[0.06] hover:bg-white/[0.12] border border-white/10 cursor-pointer" @click="inputDigit('8')">8</button>
          <button class="h-11 rounded-lg bg-white/[0.06] hover:bg-white/[0.12] border border-white/10 cursor-pointer" @click="inputDigit('9')">9</button>

          <button class="h-11 rounded-lg bg-orange-500/20 hover:bg-orange-500/30 border border-orange-500/40 text-orange-400 cursor-pointer" @click="opKey('×')">×</button>
          <button class="h-11 rounded-lg bg-white/[0.06] hover:bg-white/[0.12] border border-white/10 cursor-pointer" @click="inputDigit('4')">4</button>
          <button class="h-11 rounded-lg bg-white/[0.06] hover:bg-white/[0.12] border border-white/10 cursor-pointer" @click="inputDigit('5')">5</button>
          <button class="h-11 rounded-lg bg-white/[0.06] hover:bg-white/[0.12] border border-white/10 cursor-pointer" @click="inputDigit('6')">6</button>

          <button class="h-11 rounded-lg bg-orange-500/20 hover:bg-orange-500/30 border border-orange-500/40 text-orange-400 cursor-pointer" @click="opKey('-')">−</button>
          <button class="h-11 rounded-lg bg-white/[0.06] hover:bg-white/[0.12] border border-white/10 cursor-pointer" @click="inputDigit('1')">1</button>
          <button class="h-11 rounded-lg bg-white/[0.06] hover:bg-white/[0.12] border border-white/10 cursor-pointer" @click="inputDigit('2')">2</button>
          <button class="h-11 rounded-lg bg-white/[0.06] hover:bg-white/[0.12] border border-white/10 cursor-pointer" @click="inputDigit('3')">3</button>

          <button class="h-11 rounded-lg bg-orange-500/20 hover:bg-orange-500/30 border border-orange-500/40 text-orange-400 cursor-pointer" @click="opKey('+')">+</button>
          <button class="h-11 rounded-lg bg-white/[0.06] hover:bg-white/[0.12] border border-white/10 cursor-pointer" @click="toggleSign">±</button>
          <button class="h-11 rounded-lg bg-white/[0.06] hover:bg-white/[0.12] border border-white/10 cursor-pointer" @click="inputDigit('0')">0</button>
          <button class="h-11 rounded-lg bg-white/[0.06] hover:bg-white/[0.12] border border-white/10 cursor-pointer" @click="inputDot">.</button>

          <button class="h-12 rounded-xl col-span-4 bg-orange-500 hover:bg-orange-400 text-black border border-orange-500/50 font-semibold cursor-pointer" @click="computeEquals">=</button>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.tabular-nums { font-variant-numeric: tabular-nums; }
</style>
