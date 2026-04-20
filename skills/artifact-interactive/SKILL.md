---
name: artifact-interactive
description: >-
  Build interactive utility artifacts: calculators, converters, generators, quizzes,
  configurators, mini-games, and creative tools. Focuses on state architecture,
  instant feedback loops, keyboard accessibility, and local persistence. Use when
  building tools that compute or transform input in real-time, step-by-step workflows
  with branching logic, quiz/assessment flows, or any artifact where the interaction
  itself is the product rather than a document or data display.
---

# Interactive Utility Artifact

> The project has already been initialized via `web-artifacts-builder` (Step 1)
> and the visual design layer has been established via `artifact-visual`
> (Step 2) — OKLCH color tokens, typography, and motion are already configured.
> This skill adds the interaction architecture and UX patterns on top of that
> foundation. All visual rules from `artifact-visual` still apply.

## Priority

This skill's state patterns and feedback rules are the **highest-priority**
source of truth for interactive behavior. When the user's PRD conflicts with
a rule below (e.g., PRD says "add a submit button" but the tool should update
live), **follow the skill**. The user defines *what* the tool does; this skill
defines *how* it must feel. If a conflict is genuinely irreconcilable, ask the
user before proceeding.

---

## Execution Protocol

**Complete each step in order.** Do not write UI code until Steps 1–3 are done.

### Step 1: Identify Tool Type & State Pattern

Choose architecture **first** — this determines everything else.

| Type | Examples | State pattern | Layout |
|------|----------|--------------|--------|
| **Utility** | Calculator, converter, unit transformer | `useState` + derived values | Two-panel (input/output) |
| **Generator** | Password gen, color palette, lorem ipsum | `useState` + seed/randomness | Options top, output bottom |
| **Quiz / Assessment** | Personality test, knowledge quiz | `useReducer` with step machine | Single centered card |
| **Configurator** | Pricing builder, product customizer | `useState` tree + live preview | Sidebar controls + preview |
| **Mini-game** | Wordle clone, memory game, typing test | `useReducer` + game loop | Centered game board |
| **Creative tool** | Drawing canvas, generative art | Canvas ref + render loop | Full-bleed canvas |

**Write down your choice before proceeding.** This prevents mid-build
architecture changes that lead to messy state.

**Golden rule**: Derive, never store. If a value can be computed from state,
it's `useMemo`, not `useState`.

**Verification:**
```bash
# State pattern must match tool type
grep -rn "useReducer\|useState\|useMemo" src/ | head -10
```

### Step 2: State Architecture

Implement the state layer before any UI.

**Utility / Generator** — `useState` + derived:
```tsx
const [input, setInput] = useState('')
const output = useMemo(() => transform(input), [input])
```

**Quiz / Game** — `useReducer` for explicit state machine:
```tsx
type Phase = 'idle' | 'active' | 'result'

type State = {
  phase: Phase
  step: number
  answers: Record<number, string>
  score: number
}

type Action =
  | { type: 'START' }
  | { type: 'ANSWER'; questionId: number; value: string }
  | { type: 'NEXT' }
  | { type: 'FINISH' }
  | { type: 'RESET' }

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'START':  return { ...state, phase: 'active', step: 0, answers: {}, score: 0 }
    case 'ANSWER': return { ...state, answers: { ...state.answers, [action.questionId]: action.value } }
    case 'NEXT':   return { ...state, step: state.step + 1 }
    case 'FINISH': return { ...state, phase: 'result', score: calcScore(state.answers) }
    case 'RESET':  return initialState
    default:       return state
  }
}
```

**File structure:**
```
src/
  lib/
    state.ts             → reducer or state types + logic
    compute.ts           → pure transformation functions (no React)
    persistence.ts       → useLocalStorage hook (if needed)
  components/
    InputPanel.tsx        → user controls
    OutputPanel.tsx       → result display
    EmptyState.tsx        → shown before any input
    ErrorState.tsx        → shown on computation failure
  App.tsx                 → layout + state wiring
```

**Verification:**
```bash
# State logic must be in dedicated file, not scattered in components
[ -f "src/lib/state.ts" ] || [ -f "src/lib/compute.ts" ] && echo "✅ State/logic separated" || echo "❌ Missing state/logic file"
```

### Step 3: Define All UI States

Every interactive tool has these states. **Define what each looks like before
writing any JSX.** This prevents the common failure of only building the
"happy path."

| State | What the user sees | Required? |
|-------|--------------------|-----------|
| **Empty / Idle** | Tool ready, placeholder content, clear affordance to start | Always |
| **Active / Computing** | Input accepted, output updating live | Always |
| **Result** | Final output displayed, options to copy/share/reset | Always |
| **Error** | Computation failed or invalid input — clear message + recovery | Always |
| **Loading** | Async operation in progress — spinner or skeleton | If async |

**Banned:**
- ❌ Blank screens in any state — always show something helpful
- ❌ `console.log` as only error handling — must show error to user
- ❌ Output-only view with no way to reset or start over
- ❌ Missing empty state — the tool looks broken before first interaction

**Verification:**
```bash
# Must have empty/idle state component or conditional
grep -rn "empty\|idle\|EmptyState\|placeholder\|getStarted\|no results" src/ | wc -l | xargs -I{} [ {} -ge 1 ] && echo "✅ Has empty state" || echo "❌ Missing empty state"
# Must have error handling
grep -rn "error\|Error\|catch\|toast.error" src/ | wc -l | xargs -I{} [ {} -ge 1 ] && echo "✅ Has error handling" || echo "❌ Missing error handling"
```

### Step 4: Build UI (Depth-First)

Build in this order — **fully polish each before moving on:**

```
1. Layout shell     → correct archetype for tool type (two-panel, centered card, etc.)
2. Input controls   → all input fields with proper labels and types
3. Output display   → result rendering with copy/share actions
4. Empty state      → what shows before any input
5. Error state      → what shows when computation fails
6. Keyboard shortcuts → Cmd+Enter to run, Escape to reset, etc.
7. Persistence      → save state to localStorage if appropriate
```

**Layout archetypes** (use the one matching your tool type):

Converter / Calculator:
```tsx
<div className="grid grid-cols-1 md:grid-cols-2 gap-4 h-full">
  <InputPanel />
  <OutputPanel />
</div>
```

Quiz / Assessment:
```tsx
<div className="max-w-lg mx-auto">
  <Progress value={(step / total) * 100} className="mb-6 h-1" />
  <QuestionCard question={questions[step]} onAnswer={handleAnswer} />
</div>
```

Configurator:
```tsx
<div className="flex gap-6 h-full">
  <aside className="w-72 shrink-0 space-y-4 overflow-auto"><Controls /></aside>
  <div className="flex-1 border rounded-lg overflow-hidden"><Preview config={config} /></div>
</div>
```

Generator:
```tsx
<div className="space-y-4">
  <OptionsBar />
  <OutputBlock value={output} onCopy={copyOutput} onRegenerate={regenerate} />
</div>
```

### Step 5: Keyboard & Feedback

All tools must be fully operable without a mouse:
```tsx
useEffect(() => {
  const handler = (e: KeyboardEvent) => {
    if ((e.metaKey || e.ctrlKey) && e.key === 'Enter') handleRun()
    if ((e.metaKey || e.ctrlKey) && e.key === 'c' && !window.getSelection()?.toString()) copyOutput()
    if (e.key === 'Escape') handleReset()
  }
  window.addEventListener('keydown', handler)
  return () => window.removeEventListener('keydown', handler)
}, [handleRun, copyOutput, handleReset])
```

Show shortcuts as UI hints:
```tsx
<span className="text-xs text-muted-foreground flex items-center gap-1">
  Run <kbd className="px-1.5 py-0.5 bg-muted border rounded text-[11px] font-mono">⌘↵</kbd>
</span>
```

Feedback patterns:
- Input → output updates **synchronously** on every keystroke (no submit button for live tools)
- Async operations → show skeleton/spinner immediately, never blank
- State changes → 100–200ms CSS transition on `opacity`/`transform` only
- Copy / export actions → `sonner` toast confirmation, 1500ms duration

```tsx
import { toast } from 'sonner'

const copyOutput = () => {
  navigator.clipboard.writeText(output)
  toast.success('Copied to clipboard', { duration: 1500 })
}
```

### Step 6: Pre-Bundle Validation

```bash
echo "=== Interactive Artifact Validation ==="

# 1. State logic separated from UI
([ -f "src/lib/state.ts" ] || [ -f "src/lib/compute.ts" ]) && echo "✅ State/logic separated" || echo "❌ State logic not separated"

# 2. Empty state exists
grep -rqn "empty\|idle\|EmptyState\|placeholder\|getStarted" src/ && echo "✅ Empty state handled" || echo "❌ Missing empty state"

# 3. Error handling exists
grep -rqn "catch\|toast.error\|ErrorState\|error" src/ && echo "✅ Error handling present" || echo "❌ Missing error handling"

# 4. Keyboard shortcuts
grep -rqn "addEventListener.*keydown\|useEffect.*KeyboardEvent" src/ && echo "✅ Keyboard shortcuts present" || echo "❌ Missing keyboard shortcuts"

# 5. Copy/share functionality
grep -rqn "clipboard\|navigator.clipboard\|copyOutput\|onCopy" src/ && echo "✅ Copy functionality" || echo "❌ Missing copy/share"

# 6. No derived state stored
DERIVED=$(grep -rn "useState.*output\|useState.*result\|useState.*computed" src/ | wc -l)
[ "$DERIVED" -eq 0 ] && echo "✅ No derived state stored" || echo "⚠️  Possible derived state in useState ($DERIVED hits — verify manually)"

# 7. Layout matches tool type
grep -rqn "grid-cols\|max-w-lg\|flex gap" src/App.tsx && echo "✅ Layout archetype applied" || echo "❌ No layout archetype in App.tsx"

echo "=== End Validation ==="
```

Only after all checks pass → run `bundle-artifact.sh`.

---

## Pattern Reference

### Persistence

```tsx
function useLocalStorage<T>(key: string, initial: T) {
  const [value, setValue] = useState<T>(() => {
    try { return JSON.parse(localStorage.getItem(key) ?? '') ?? initial }
    catch { return initial }
  })
  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value))
  }, [key, value])
  return [value, setValue] as const
}
```

### Copyable Output Block

```tsx
<div className="relative group">
  <pre className="bg-muted rounded-lg p-4 font-mono text-sm overflow-auto whitespace-pre-wrap">
    {output}
  </pre>
  <Button
    size="icon" variant="ghost"
    className="absolute top-2 right-2 opacity-0 group-hover:opacity-100 transition-opacity"
    onClick={copyOutput}
  >
    <Copy className="w-4 h-4" />
  </Button>
</div>
```

### Mini-Game Loop

```tsx
const TICK_MS = 16  // ~60fps

useEffect(() => {
  if (phase !== 'active') return
  const id = setInterval(() => dispatch({ type: 'TICK' }), TICK_MS)
  return () => clearInterval(id)
}, [phase])
```

### Canvas-Based Tools

```tsx
const canvasRef = useRef<HTMLCanvasElement>(null)
const animRef   = useRef<number>(0)

useEffect(() => {
  const ctx = canvasRef.current?.getContext('2d')
  if (!ctx) return

  const render = () => {
    ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height)
    drawFrame(ctx, stateRef.current)
    animRef.current = requestAnimationFrame(render)
  }

  animRef.current = requestAnimationFrame(render)
  return () => cancelAnimationFrame(animRef.current)
}, [])

<canvas ref={canvasRef} width={800} height={600}
  className="rounded-lg border w-full" style={{ touchAction: 'none' }} />
```

### Result / Score Display

```tsx
{phase === 'result' && (
  <div className="text-center space-y-4 py-8">
    <div className="text-5xl font-bold">
      {score}<span className="text-2xl text-muted-foreground">/{maxScore}</span>
    </div>
    <p className="text-muted-foreground">{getResultLabel(score, maxScore)}</p>
    <div className="flex gap-2 justify-center">
      <Button onClick={() => dispatch({ type: 'RESET' })}>Try Again</Button>
      <Button variant="outline" onClick={copyResult}>Share Result</Button>
    </div>
  </div>
)}
```
