---
name: artifact-form
description: >-
  Build form and survey artifacts with type-safe validation using React Hook Form
  v7 and Zod. Covers single-page forms, multi-step wizards, conditional field logic,
  dynamic field arrays, and survey flows with scoring. Use when building contact forms,
  onboarding flows, multi-step surveys, questionnaires, registration forms, checkout
  flows, or any artifact where form UX and validation logic are the core product.
---

# Form / Survey Artifact

> The project has already been initialized via `web-artifacts-builder` (Step 1)
> and the visual design layer has been established via `artifact-visual`
> (Step 2) — OKLCH color tokens, typography, and motion are already configured.
> This skill adds the form and validation layer on top of that foundation.
> All visual rules from `artifact-visual` still apply.

## Priority

This skill's patterns and UX rules are the **highest-priority** source of
truth for all form behavior. When the user's PRD conflicts with a rule below
(e.g., PRD says "validate on submit only" but the contract requires onBlur),
**follow the skill**. The user defines *what data* to collect; this skill
defines *how* the form must behave and feel. If a conflict is genuinely
irreconcilable, ask the user before proceeding.

---

## Execution Protocol

**Complete each step in order.** Do not write form UI code until Steps 1–2
are done.

### Step 1: Schema First

Define Zod schemas **before** writing any JSX. This is the single source of
truth for types, validation, and error messages.

```tsx
import { z } from 'zod'

const schema = z.object({
  email: z.string().email('Invalid email'),
  name:  z.string().min(1, 'Required'),
})
type FormData = z.infer<typeof schema>  // always derive — never write types manually
```

For multi-step forms, define per-step schemas then merge:
```tsx
const step1Schema = z.object({ name: z.string().min(1), email: z.string().email() })
const step2Schema = z.object({ plan: z.enum(['free', 'pro', 'team']) })
const fullSchema  = step1Schema.merge(step2Schema)
```

**Rules:**
- ❌ NEVER write TypeScript types manually for form data — always `z.infer<typeof schema>`
- ❌ NEVER use `any` for field names in `trigger()` or `setValue()`
- ❌ NEVER skip `defaultValues` — prevents uncontrolled→controlled React warnings
- ✅ Put cross-field validation in `.refine()` with explicit `path` pointing to the error field

**Verification:**
```bash
# Schema must exist before form components
grep -rn "z.object\|z.discriminatedUnion" src/ | head -5
# No manual form types
grep -rn "interface FormData\|type FormData = {" src/ && echo "❌ Manual types found" || echo "✅ PASS"
```

### Step 2: Form Hook Setup

```tsx
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'

const form = useForm<FormData>({
  resolver: zodResolver(schema),
  defaultValues: { email: '', name: '' },
  mode: 'onBlur',  // onSubmit (best perf) | onBlur (best balance) | onChange (live feedback)
})
```

**shadcn import trap** — IDEs auto-import the wrong `Form`:
```tsx
// ✅ Correct
import { useForm } from 'react-hook-form'
import { Form, FormField, FormItem, FormControl, FormMessage } from '@/components/ui/form'

// ❌ Wrong — react-hook-form's Form is not the shadcn wrapper
import { useForm, Form } from 'react-hook-form'
```

**Verification:**
```bash
# Must use shadcn Form import, not react-hook-form Form
grep -rn "from 'react-hook-form'" src/ | grep "Form," && echo "❌ Wrong Form import" || echo "✅ PASS"
```

### Step 3: Build Form UI (Depth-First)

Build the form section by section. For multi-step forms, **build and polish
step 1 completely** before writing step 2.

**File structure:**
```
src/
  schemas/
    form-schema.ts      → all Zod schemas
  components/
    StepIndicator.tsx    → progress (multi-step only)
    Step1Info.tsx         → step 1 fields
    Step2Plan.tsx         → step 2 fields
    ReviewStep.tsx        → read-only summary before submit
    SuccessState.tsx      → post-submit confirmation
  App.tsx                → form instance + step routing
```

**Per-field quality gate — every form field must have:**
- [ ] Visible `<FormLabel>` — never placeholder-only
- [ ] `<FormMessage />` for error display
- [ ] Appropriate input type (`email`, `tel`, `number`, `url`)
- [ ] Consistent width — related fields grouped, not all full-width

**Form-level quality gate:**
- [ ] Submit button shows loading state (`isSubmitting`)
- [ ] Disabled state on submit button when `isSubmitting`
- [ ] Success state after submission (not just console.log)
- [ ] Error handling for submission failure (toast or inline)

### Step 4: Implement All Required States

Every form must implement these UX states — **do not skip any**:

| State | What to show | How |
|-------|-------------|-----|
| **Empty** | Form with labels, placeholders, no errors | Default render |
| **Filling** | Fields with user input, no errors yet | `mode: 'onBlur'` delays validation |
| **Field Error** | Red border + `<FormMessage>` below field | Triggered on blur or submit |
| **Submitting** | Button shows spinner + "Submitting…", all fields disabled | `form.formState.isSubmitting` |
| **Success** | Confirmation message/screen, option to submit another | Replace form or show overlay |
| **Submit Error** | Toast or banner with retry option, form still editable | `try/catch` in `onSubmit` |

```tsx
const onSubmit = async (data: FormData) => {
  try {
    await submitToAPI(data)
    setSubmitted(true)
  } catch (err) {
    toast.error('Submission failed', {
      description: 'Please try again.',
      action: { label: 'Retry', onClick: () => form.handleSubmit(onSubmit)() },
    })
  }
}
```

**Verification:**
```bash
# Must have loading state on submit
grep -rn "isSubmitting" src/ | wc -l | xargs -I{} [ {} -ge 1 ] && echo "✅ Has loading state" || echo "❌ Missing isSubmitting"
# Must have success state
grep -rn "success\|submitted\|SuccessState\|Submitted" src/ | wc -l | xargs -I{} [ {} -ge 1 ] && echo "✅ Has success state" || echo "❌ Missing success state"
```

### Step 5: Pre-Bundle Validation

```bash
echo "=== Form Artifact Validation ==="

# 1. Zod schema exists
grep -rqn "z.object\|z.discriminatedUnion" src/ && echo "✅ Zod schema defined" || echo "❌ Missing Zod schema"

# 2. No manual form types
grep -rn "interface FormData\|type FormData = {" src/ && echo "❌ Manual types" || echo "✅ Types derived from schema"

# 3. Correct Form import
grep -rn "from 'react-hook-form'" src/ | grep "Form," && echo "❌ Wrong Form import" || echo "✅ Correct imports"

# 4. Loading state
grep -rqn "isSubmitting" src/ && echo "✅ Loading state handled" || echo "❌ No loading state"

# 5. Error messages present
FORM_MSG=$(grep -rn "FormMessage" src/ | wc -l)
[ "$FORM_MSG" -ge 2 ] && echo "✅ FormMessage present ($FORM_MSG)" || echo "❌ Missing FormMessage ($FORM_MSG)"

# 6. Labels present
LABELS=$(grep -rn "FormLabel" src/ | wc -l)
[ "$LABELS" -ge 2 ] && echo "✅ FormLabel present ($LABELS)" || echo "❌ Missing FormLabel ($LABELS)"

echo "=== End Validation ==="
```

Only after all checks pass → run `bundle-artifact.sh`.

---

## Pattern Reference

### Conditional Fields

```tsx
const accountType = form.watch('accountType')

{accountType === 'business' && (
  <FormField control={form.control} name="companyName" render={({ field }) => (
    <FormItem>
      <FormLabel>Company Name</FormLabel>
      <FormControl><Input {...field} /></FormControl>
      <FormMessage />
    </FormItem>
  )} />
)}

// Validated conditionally with discriminatedUnion
const schema = z.discriminatedUnion('accountType', [
  z.object({ accountType: z.literal('personal'), name: z.string().min(1) }),
  z.object({ accountType: z.literal('business'), companyName: z.string().min(2) }),
])
```

### Dynamic Field Arrays

```tsx
const { fields, append, remove } = useFieldArray({ control: form.control, name: 'items' })

{fields.map((field, i) => (
  <div key={field.id} className="flex gap-2">  {/* CRITICAL: field.id not index as key */}
    <Input {...form.register(`items.${i}.value` as const)} />
    <Button type="button" variant="ghost" size="icon" onClick={() => remove(i)}>
      <Trash2 className="w-4 h-4" />
    </Button>
  </div>
))}
<Button type="button" variant="outline" onClick={() => append({ value: '' })}>
  Add Item
</Button>
```

`useFieldArray` only works with **arrays of objects** — wrap primitives:
`[{ value: 'string' }]` not `['string']`

### Multi-Step Wizard

```tsx
const [step, setStep] = useState(1)

const nextStep = async () => {
  const fields = step === 1 ? ['name', 'email'] : ['plan']
  const valid = await form.trigger(fields as any)
  if (valid) setStep(s => s + 1)
}
```

Progress bar:
```tsx
<Progress value={(currentStep / totalSteps) * 100} className="h-1" />
```

### Survey Rating Scale

```tsx
<FormField name="rating" render={({ field }) => (
  <RadioGroup onValueChange={field.onChange} defaultValue={field.value}
    className="flex gap-2">
    {[1, 2, 3, 4, 5].map(n => (
      <label key={n} className="cursor-pointer">
        <RadioGroupItem value={String(n)} className="sr-only" />
        <div className={cn(
          'w-10 h-10 rounded-full border-2 flex items-center justify-center text-sm font-medium transition-colors',
          field.value === String(n)
            ? 'border-primary bg-primary text-primary-foreground'
            : 'border-border hover:border-primary/50'
        )}>
          {n}
        </div>
      </label>
    ))}
  </RadioGroup>
)} />
```

### Cross-Field Validation

```tsx
z.object({
  password: z.string().min(8),
  confirm:  z.string(),
}).refine(d => d.password === d.confirm, {
  message: "Passwords don't match",
  path: ['confirm'],  // error must point to the specific field
})
```

### Known Pitfalls

- `z.string().optional()` + empty `""` triggers false validation errors
  → fix: `.or(z.literal(''))` or `z.preprocess(v => v === '' ? undefined : v, z.string().optional())`
- `useFieldArray` with 300+ fields + `formState` destructuring freezes for ~10s
  → fix: read `form.formState.isValid` inline at submit time only, not as destructured variable
- Auto-import brings in `Form` from `react-hook-form` — always verify the import path is `@/components/ui/form`
- `form.reset()` after server actions causes validation errors on next submit (RHF <7.65)
  → fix: use `form.setValue()` per field instead of `reset()`
