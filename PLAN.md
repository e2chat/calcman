# Plan: Calculator App

1) UI
- Create components/Calculator.vue using Nuxt UI + Tailwind.
- Display: primary and secondary (expression, live value), responsive layout.
- Buttons: 0-9, ., +, -, ×, ÷, %, =, C, CE, ⌫, ±, M+, M-, MR, MC.
- Keyboard: digits, operators (+-*/%), Enter, Backspace, Escape, Delete, ., parentheses optional.

2) Logic
- Expression model with tokenization and shunting-yard to RPN, evaluate with float64 Big.js-like safe ops via JS number and decimal handling; limit precision and show error states.
- Support unary minus, percent, toggling sign, memory register.

3) Accessibility
- Focus ring, aria-labels, roving tabindex for grid, button titles with keyboard shortcuts.

4) Theming
- Modern glass card, dark/light via Nuxt UI theme; Tailwind utility classes.

5) Integration
- Page at pages/index.vue imports Calculator.

6) QA
- Basic runtime checks; ensure keyboard works; mobile layout.
