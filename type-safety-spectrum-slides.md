---
marp: true
theme: default
paginate: true
style: |
  section {
    font-size: 24px;
  }
  section.smaller {
    font-size: 22px;
  }
  pre {
    font-size: 16px;
  }
  table {
    font-size: 20px;
  }
---

# Type Safety Is a Spectrum, Not a Boolean

> Why "100% strict TypeScript" is neither realistic nor always desirable — and how to place safety where it actually matters.

---

## 1. Core Thesis

**Type safety is *not*:**
- `strict: true` — a config flag, not a philosophy
- "no `any` allowed" — dogmatic, not pragmatic
- maximal generics everywhere — often unreadable overkill

**Type safety *is*:**
- **Strategic** — where does a bug cost the most?
- **Layered** — different rigor for different layers
- **Context-dependent** — forms ≠ internal utils

> The goal is not perfect types — the goal is **preventing expensive failures**.

---

### 1.1 Why "strict: true" Is Not Enough

`strict: true` is a compiler flag, not a safety guarantee.

```ts
// tsconfig.json: strict: true
// TypeScript is happy. Runtime: 💥

const user = await fetch('/api/user').then(r => r.json()) as User
console.log(user.email)  // undefined if API changed
```

**What strict mode does:** Catches null/undefined mistakes, requires explicit types, enables stricter checks.

**What it doesn't do:** Validate data from APIs, forms, localStorage, or any external source.

---

### 1.2 Why "No Any" Is Misguided

Banning `any` is a rule that ignores context.

| Situation | `any` Acceptable? | Why |
|-----------|------------------|-----|
| API response | ❌ No | High risk — validate with schema |
| Form input | ❌ No | User-controlled — always untrusted |
| One-off migration script | ✅ Yes | Runs once, delete after |
| Internal array util | ✅ Maybe | Low risk, isolated, easy to refactor |
| Third-party lib with bad types | ✅ Maybe | Wrapper function, validate at edge |

The question isn't "is there an `any`?" — it's "what's the blast radius if this is wrong?"

---

### 1.3 Why Maximal Generics Backfire

Generics are powerful. Over-using them creates unreadable, unmaintainable code.

```ts
// "Flexible" — but what does this do?
type Merge<T, U> = Omit<T, keyof U> & U
type DeepRequired<T> = { [K in keyof T]-?: T[K] extends object ? DeepRequired<T[K]> : T[K] }
type EventHandler<E extends keyof WindowEventMap> = (event: WindowEventMap[E]) => void

// Simple — everyone gets it
type User = { id: number; name: string; email: string }
type UpdateUserInput = Partial<Pick<User, 'name' | 'email'>>
```

**Rule:** If you can't explain the type in 30 seconds, it's too complex.

---

### 1.4 Strategic: Where Does a Bug Cost the Most?

Not all bugs are equal. Prioritize type safety where failure is expensive.

| Area | Bug Cost | Type Investment |
|------|----------|-----------------|
| Pricing calculation | 💰💰💰 High (money loss) | Maximum — typed, tested, validated |
| Permission check | 🔒🔒🔒 High (security breach) | Maximum — union types, exhaustive |
| API response parsing | 🔥🔥 Medium-High (crashes) | High — runtime schema validation |
| UI button styling | 🟡 Low (visual glitch) | Low — optional props, quick iteration |
| Internal dev script | 🟢 Minimal (dev-only) | Minimal — `any` is fine |

---

### 1.5 Layered: Different Rigor for Different Layers

Think in concentric circles: strictest at boundaries, looser inside.

```
┌─────────────────────────────────────────┐
│  External World (APIs, users, config)  │  ← 🔴 Validate everything
├─────────────────────────────────────────┤
│  Boundaries (fetch, forms, env)        │  ← 🔴 Parse with Zod/Valibot
├─────────────────────────────────────────┤
│  Business Logic (pricing, perms)       │  ← 🟠 Strict types, tested
├─────────────────────────────────────────┤
│  UI Components (props, state)          │  ← 🟡 Typed, but flexible
├─────────────────────────────────────────┤
│  Internal Utils (helpers, formatters)  │  ← 🟢 Simple types, refactor-friendly
└─────────────────────────────────────────┘
```

---

### 1.6 Context-Dependent: Forms ≠ Internal Utils

Same data, different treatment based on source.

```ts
// Form input — user-controlled, NEVER trust
const handleSubmit = (formData: unknown) => {
  const parsed = UserSchema.safeParse(formData)  // Must validate
  if (!parsed.success) return showErrors(parsed.error)
  saveUser(parsed.data)
}

// Internal util — data already validated upstream
function formatUserName(user: User): string {
  return `${user.firstName} ${user.lastName}`  // No re-validation needed
}
```

---

### 1.7 Dogmatic vs Pragmatic

**Dogmatic:** Rules above context
- "No `any` ever" — even for one-off, low-risk helpers
- "Strict mode or nothing" — regardless of legacy codebase
- Maximize type coverage everywhere — time spent on types vs value delivered

**Pragmatic:** Context above rules
- Use strict types where failure is expensive (API boundaries, forms, business logic)
- Use looser types where iteration speed matters (UI experiments, internal utils)
- Ask: "What happens if this is wrong?" — then invest accordingly

---

### 1.8 Example: Dogmatic vs Pragmatic

```ts
// Dogmatic: reject any, spend 2 hours typing a one-off util
function flatten<T extends readonly (readonly unknown[])[]>(
  arr: T
): T[number] extends readonly (infer U)[] ? U[] : never { ... }

// Pragmatic: quick, clear, isolated
function flatten(arr: any[][]): any[] {
  return arr.flat()
}
```

The second ships faster, is readable, and lives in a low-risk helper. Over-typing it doesn't prevent costly bugs elsewhere.

---

### 1.9 The Spectrum Visualized

```
         LOW SAFETY                              HIGH SAFETY
              │                                        │
              ▼                                        ▼
┌─────────────────────────────────────────────────────────────────┐
│  any  │  loose  │  typed  │  strict  │  validated  │  branded  │
└─────────────────────────────────────────────────────────────────┘
   │         │         │          │           │            │
   │         │         │          │           │            └─ UserId (not just number)
   │         │         │          │           └─ Zod/Valibot runtime parse
   │         │         │          └─ strict: true, no implicit any
   │         │         └─ explicit types, interfaces
   │         └─ optional props, Partial<T>
   └─ internal scripts, legacy migration

Choose position based on: blast radius, data source, change frequency
```

---

## 2. The Illusion of Type Safety in TypeScript

**TypeScript:**
- Exists **only at compile time** — completely gone in the browser
- Is **erased at runtime** — no runtime type information
- **Blindly trusts** external data — `as User` is a lie if the API changed

**Common failure sources:**
- API responses
- Feature flags
- User input
- JSON parsing
- Config files
- Backend/frontend version drift

---

### 2.1 API Responses

Backend changes shape; frontend types stay stale.

```ts
// Backend returns { id: 1, emailAddress: "a@b.com" } — renamed email
interface User { id: number; email: string }

const user = await fetch('/api/user').then(r => r.json()) as User
sendEmail(user.email)  // undefined — TS says string, runtime: 💥
```

**Typical causes:** Renamed fields, optional → required, number → string (IDs as UUIDs), different teams, no shared schema.

---

### 2.2 Feature Flags

External service returns unpredictable shapes. SDK types often lie.

```ts
// LaunchDarkly, Split, etc. — variant can be string, number, object
const variant = ldClient.variation('checkout-flow', 'legacy')
// TS says string. Runtime: could be { enabled: true, version: 2 }

if (variant === 'new') { ... }  // 💥 Object never equals string
```

**Typical causes:** Flag config changed, A/B test variants, rollout percentages, SDK version mismatch.

---

### 2.3 User Input

Forms, URL params, localStorage — users and extensions can send anything.

```vue
<script setup lang="ts">
import { useRoute } from 'vue-router'

const route = useRoute()
// URL: /profile?userId=123 — or ?userId=alert(1) or ?userId=
const userId = route.query.userId  // string | string[] | undefined, not number

fetch(`/api/users/${userId}`)  // TS fine. Runtime: /api/users/undefined, XSS risk
</script>
```

**Typical causes:** Malformed form data, browser extensions, direct URL edits, copy-paste from Excel, non-ASCII characters.

---

### 2.4 JSON Parsing

`JSON.parse` returns `any`. No validation.

```ts
const data = JSON.parse(localStorage.getItem('cart') ?? '{}')
// TS: any. Could be null, string, { items: "not an array" }

const total = data.items.reduce((sum, i) => sum + i.price, 0)
// 💥 data.items might not be array, i.price might not exist
```

**Typical causes:** Corrupted storage, old format after migration, manual edits, different app version wrote it.

---

### 2.5 Config Files & Env Vars

Files and env are user-controlled. No compile-time guarantee.

```ts
// .env: PORT=3000 — or PORT=not-a-number, or PORT missing
const port = process.env.PORT  // string | undefined

http.listen(port)  // TS might allow. Runtime: 💥 port is undefined or "abc"

// config.json — user edited, invalid JSON or wrong shape
const { apiUrl } = require('./config.json')  // any
fetch(apiUrl)  // Could be undefined, relative path, wrong protocol
```

**Typical causes:** Typos in .env, Docker/K8s env overrides, config from different env (dev vs prod), JSON syntax errors.

---

### 2.6 Backend/Frontend Version Drift

Frontend and backend deployed independently. Types assume same version.

```ts
// Frontend expects: { orders: Order[] }
// Backend v2 returns: { orders: Order[], pagination: { page, total } }
const { orders } = await fetchOrders()
// Old frontend: fine. New backend: extra field ignored.

// Backend v3 removes orders, returns { data: Order[] }
const { orders } = await fetchOrders()  // orders === undefined. 💥
```

**Typical causes:** Staged rollouts, canary releases, mobile app not updated, cached frontend with new API.

---

## 3. The Type Safety Spectrum

| Layer | Safety Level | Rationale |
|-------|-------------|-----------|
| UI components | 🟡 Medium | Changes often, low blast radius |
| Forms | 🔴 High | User input is untrusted |
| API boundaries | 🔴🔴 Very High | External & unstable |
| Business logic | 🔴🔴 Very High | Bugs are expensive |
| Internal helpers | 🟢 Low | Refactor-friendly, low risk |

---

### 3.1 UI Components — Medium Safety

```vue
<!-- Props can be loose — component changes frequently -->
<script setup lang="ts">
defineProps<{
  title?: string
  variant?: 'default' | 'outlined'
  onClick?: () => void
}>()
</script>

<template>
  <div class="card" @click="onClick">
    <slot />
  </div>
</template>
```

**Why medium:** Fast iteration matters. Wrong padding is cheap. Wrong calculation isn't.

---

### 3.2 Forms — High Safety

```vue
<script setup lang="ts">
import { ref } from 'vue'
import { z } from 'zod'

const LoginSchema = z.object({
  email: z.string().email('Invalid email'),
  password: z.string().min(8, 'Too short'),
})
type LoginForm = z.infer<typeof LoginSchema>

const formErrors = ref<Record<string, string[]>>({})

const handleSubmit = (data: unknown) => {
  const parsed = LoginSchema.safeParse(data)
  if (!parsed.success) {
    formErrors.value = parsed.error.flatten().fieldErrors
    return
  }
  // Now parsed.data is safe to use
}
</script>
```

---

### 3.3 API Boundaries — Very High Safety

```ts
// Validate EVERY response at the boundary (e.g. in a composable)
// composables/useUser.ts
import { z } from 'zod'

const UserResponseSchema = z.object({
  id: z.number(),
  email: z.string().email(),
  createdAt: z.string().datetime(),
})
type User = z.infer<typeof UserResponseSchema>

export function useUser() {
  async function fetchUser(): Promise<User> {
    const res = await fetch('/api/user')
    const json = await res.json()
    return UserResponseSchema.parse(json)  // Throws if invalid
  }
  return { fetchUser }
}
```

Fail fast. Don't let bad data propagate into your app.

---

### 3.4 Business Logic — Very High Safety

```ts
// Pricing, permissions, domain rules — type everything
type OrderStatus = 'draft' | 'confirmed' | 'shipped' | 'cancelled'

function canCancel(status: OrderStatus): boolean {
  return status === 'draft' || status === 'confirmed'
}

// Exhaustive checking
function getStatusLabel(status: OrderStatus): string {
  switch (status) {
    case 'draft': return 'Draft'
    case 'confirmed': return 'Confirmed'
    case 'shipped': return 'Shipped'
    case 'cancelled': return 'Cancelled'
    default: return status  // TS error if we add a new status and forget
  }
}
```

---

### 3.5 Internal Helpers — Low Safety

```ts
// Simple, refactor-friendly — don't over-invest
function pick<T extends object, K extends keyof T>(obj: T, keys: K[]): Pick<T, K> {
  return keys.reduce((acc, k) => ({ ...acc, [k]: obj[k] }), {} as Pick<T, K>)
}

// Or even:
function pick(obj: Record<string, unknown>, keys: string[]) {
  return Object.fromEntries(keys.map(k => [k, obj[k]]))
}
```

If it's internal and low-risk, ship it. Add types when it stabilizes.

---

## 4. Compile-Time vs Runtime Safety

**Compile-time only** — TypeScript validates your code, not the world.

```ts
function handleUser(user: User) {
  sendEmail(user.email)  // TS happy. Runtime: user might be { email: null }
}
```

**Compile-time + Runtime** — Validate at the boundary.

```ts
import { z } from 'zod'

const UserSchema = z.object({
  id: z.number(),
  email: z.string().email(),
})

const user = UserSchema.parse(await fetchUser())  // Throws or returns valid User
handleUser(user)  // Now we KNOW the shape
```

> Runtime validation is type safety's missing half.

---

### 4.1 Zod vs Manual Validation

```ts
// Manual — verbose, easy to miss cases
function parseUser(data: unknown): User {
  if (typeof data !== 'object' || data === null) throw new Error('Invalid')
  const obj = data as Record<string, unknown>
  if (typeof obj.id !== 'number') throw new Error('id must be number')
  if (typeof obj.email !== 'string') throw new Error('email must be string')
  return { id: obj.id, email: obj.email }
}

// Zod — declarative, inferred types
const UserSchema = z.object({ id: z.number(), email: z.string().email() })
type User = z.infer<typeof UserSchema>
const user = UserSchema.parse(data)
```

---

## 5. Where Strict Typing Pays Off

| Area | Why |
|------|-----|
| **API contracts** | Single source of truth, catches drift |
| **Form models** | Invalid input → clear errors, not silent bugs |
| **Search, filters, pagination** | `page`, `limit`, `sort` — easy to typo, expensive to debug |
| **Permissions** | `canEdit`, `canDelete` — security-critical |
| **Domain calculations** | Tax, discounts, totals — money bugs are costly |

---

### 5.1 API Contracts

Single source of truth. Backend/frontend drift caught at compile time.

```ts
// Shared schema — OpenAPI or Zod — both sides validate
const OrderResponseSchema = z.object({
  id: z.string().uuid(),
  status: z.enum(['pending', 'confirmed', 'shipped']),
  items: z.array(z.object({ sku: z.string(), qty: z.number() })),
})
type Order = z.infer<typeof OrderResponseSchema>

// Backend adds "shippedAt" — frontend types regenerate, no silent ignores
// Backend renames "qty" → "quantity" — TS errors everywhere, fix before deploy
```

---

### 5.2 Form Models

Invalid input → clear errors, not silent bugs or wrong DB writes.

```vue
<script setup lang="ts">
const CheckoutSchema = z.object({
  email: z.string().email(),
  quantity: z.number().min(1).max(100),
  coupon: z.string().optional(),
})
type CheckoutForm = z.infer<typeof CheckoutSchema>

const formData = reactive({ email: '', quantity: 1, coupon: '' })

const handleSubmit = () => {
  const parsed = CheckoutSchema.safeParse(formData)
  // "quantity" as string from input? Caught. Invalid email? Caught.
  // No silent parseInt("abc") === NaN or wrong DB writes
  if (parsed.success) submitOrder(parsed.data)
}
</script>
```

---

### 5.3 Search, Filters & Pagination

`page`, `limit`, `sort` — easy to typo, expensive to debug.

```ts
// Typo: "pge" instead of "page" — silent failure, API ignores
const params = { pge: 1, limit: 10 }

// Typed — catch at compile time
type SearchParams = {
  page: number
  limit: number
  sort?: 'name' | 'date' | 'price'
  order?: 'asc' | 'desc'
}

function buildSearchUrl(params: SearchParams): string {
  return `/search?page=${params.page}&limit=${params.limit}&sort=${params.sort}`
}
// params.sort = 'pice' → TS error. params.order = 'ascending' → TS error
```

---

### 5.4 Permissions / Authorization

Security-critical. Typos or wrong checks → data leaks.

```ts
type Permission = 'read' | 'edit' | 'delete'

function canAccess(user: User, resource: Resource, action: Permission): boolean {
  const rules = user.role.permissions
  return rules.includes(action)  // Typo "delet" → TS error
}

// In template/composable
if (canAccess(user, order, 'delete')) {
  showDeleteButton.value = true
}
// Wrong: canAccess(user, order, 'remove') → TS error, no silent bypass
```

---

### 5.5 Domain Calculations

Tax, discounts, totals — money bugs are costly and hard to trace.

```ts
type Money = number  // Or use a branded type / decimal lib

function calculateTotal(
  items: { price: Money; quantity: number }[],
  discountPercent: number
): Money {
  const subtotal = items.reduce((sum, i) => sum + i.price * i.quantity, 0)
  return subtotal * (1 - discountPercent / 100)
}
// discountPercent as string? TS error. Missing item? TS error.
// Refactor: change Money to cents → TS finds every usage
```

---

## 6. Where Type Safety Hurts

**Over-typing smells:**
- Deep generic hierarchies (`A<B<C<D>>>`)
- Constraint puzzles (`T extends infer U ? ...`)
- Utility types used once (extract to a type and never reuse)
- Types no one understands without hovering 5 times

> Type safety that slows delivery is **anti-safety**.

---

### 6.1 Deep Generic Hierarchies

Nested generics — every layer adds indirection. Debugging type errors becomes archaeology.

```ts
// "Flexible" table component — 4 levels deep
type ColumnDef<T, K extends keyof T> = {
  key: K
  render?: (value: T[K], row: T) => VueNode
}

type TableProps<T, C extends ColumnDef<T, keyof T>[]> = {
  data: T[]
  columns: C
  onRowClick?: (row: T, index: number) => void
}

// Usage: TS error "Type 'string' does not satisfy constraint 'keyof User'"
// Good luck finding which generic caused it
```

**Cost:** 20 min to fix a type error. Ship a simpler prop interface instead.

---

### 6.2 Constraint Puzzles

Conditional types, `infer`, distributivity — fun for puzzles, painful for real code.

```ts
// "Extract the payload from an event type"
type EventPayload<E> = E extends { payload: infer P } ? P : never

// "Get the second argument of a function"
type SecondArg<F> = F extends (a: any, b: infer B, ...rest: any[]) => any ? B : never

// Combine them — now what does this resolve to?
type X = SecondArg<(a: string, b: EventPayload<{ payload: number }>) => void>
// Answer: number. Obvious, right? (Nobody gets this on first read)
```

**Cost:** 10 min per person to understand. Used once. Delete it.

---

### 6.3 Utility Types Used Once

Extract to a type for "reuse" — then never use it again.

```ts
// Created for a single form component
type PickRequired<T, K extends keyof T> = Pick<T, K> & Required<Pick<T, K>>

type UserFormFields = PickRequired<User, 'email' | 'name'>

// UserFormFields used in exactly one place. The inline type was fine:
type UserFormFields = Pick<User, 'email' | 'name'> & Required<Pick<User, 'email' | 'name'>>
// Or just: { email: string; name: string } — 2 seconds, done
```

**Cost:** Extra indirection. Grep finds one usage. Delete the abstraction.

---

### 6.4 Types No One Understands Without Hovering

If you need 5 hovers to explain it, it's too complex.

```ts
// What does this do? Hover. Hover again. Read the docs. Still unsure.
type Awaited<T> = T extends null | undefined ? T
  : T extends object & { then(onfulfilled: infer F): any }
    ? F extends (value: infer V) => any ? Awaited<V> : never
    : T

// Built-in TS utility. Correct. Also: most devs never need to write it.
// If YOUR type looks like this — simplify or delete.
```

**Cost:** Onboarding slows. PRs stall. "Who wrote this?" Replace with a Zod schema or a simpler type.

---

### 6.5 Example: Over-Engineered vs Simple

```ts
// "Clever" — 30 min to write, 10 min to understand
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P]
}

// Simple — 2 min, everyone gets it
function updateUser(updates: Partial<User>) {
  return { ...user, ...updates }
}
```

---

### 6.6 Example: Types That Block Refactoring

```vue
<!-- Over-tight coupling — changing User breaks 50 components -->
<script setup lang="ts">
defineProps<{
  user: { id: number; email: string; avatar: string; role: Role }
}>()
</script>

<!-- Looser — refactor User, map display props at parent -->
<script setup lang="ts">
defineProps<{
  user: { id: number; displayName: string }
}>()
</script>
```

Types should **help** refactoring (find call sites), not **block** it (require updating every caller).

---

## 7. Principles to Follow

Principles used in production at large companies — not dogma, but proven practices.

---

### 7.1 Validate at Boundaries, Trust Internally

Validate at API edges, forms, and config. Inside your app, assume data is valid.

```ts
// Boundary: validate API response
const user = UserSchema.parse(await fetch('/api/user').then(r => r.json()))

// Boundary: validate form submit
const parsed = LoginSchema.safeParse(formData)
if (!parsed.success) return

// Inside app: trust it
function formatUserName(user: User) {
  return user.name  // No re-validation — we already validated at boundary
}
```

**Reference:** Slack, Stripe — Validate at boundaries; trust internally after type check.

---

### 7.2 Prefer Runtime Schemas Over Clever Generics

Zod / Valibot > 10 lines of conditional types. Runtime validation catches what compile-time misses.

```ts
// ❌ Clever — compile-time only, no runtime guarantee
type DeepPartial<T> = { [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P] }

// ✅ Runtime schema — validates and infers type
const OrderSchema = z.object({
  id: z.string().uuid(),
  items: z.array(z.object({ sku: z.string(), qty: z.number() })),
})
type Order = z.infer<typeof OrderSchema>
const order = OrderSchema.parse(rawJson)  // Throws if invalid
```

**Reference:** Stripe — OpenAPI + schema; runtime validation at SDK layer.

---

### 7.3 If You Can't Explain the Type in 30 Seconds, Simplify or Delete

Complex types slow onboarding and block PRs.

```ts
// ❌ "Extract second arg of a function" — 30 seconds to explain
type SecondArg<F> = F extends (a: any, b: infer B, ...rest: any[]) => any ? B : never

// ✅ Simple — everyone gets it in 5 seconds
type SearchParams = { page: number; limit: number; sort?: 'name' | 'date' }
```

**Reference:** Slack — "Phase features in slowly... reap benefits with the most basic use of TypeScript."

---

### 7.4 Use `unknown` Instead of `any`

Forces you to narrow. `any` opts out of safety.

```ts
// ❌ any — silently allows anything
function handleWebhook(payload: any) {
  return payload.data.id  // No error. Runtime: 💥 if payload malformed
}

// ✅ unknown — must narrow
function handleWebhook(payload: unknown) {
  const parsed = WebhookSchema.safeParse(payload)
  if (!parsed.success) throw new Error('Invalid webhook')
  return parsed.data.data.id  // Safe
}
```

**Reference:** Microsoft — Use `unknown` for untrusted data; avoid `any` except during migration.

---

### 7.5 Don't Type What You Don't Control

External libs, API responses, user input: validate, don't assert with `as`.

```ts
// ❌ Assert — lies if API changes
const user = (await res.json()) as User

// ✅ Validate — fails fast if API changes
const user = UserSchema.parse(await res.json())

// Same for: route.query, localStorage, process.env, third-party callbacks
const userId = z.coerce.number().parse(route.query.userId)
```

**Reference:** Stripe — SDKs validate API responses; don't trust raw JSON.

---

### 7.6 Types Should Help Refactoring, Not Block It

If changing a type breaks the world, it's too coupled.

```vue
<!-- ❌ Tight — User change breaks 50 components -->
<script setup lang="ts">
defineProps<{ user: { id: number; email: string; avatar: string; role: Role } }>()
</script>

<!-- ✅ Loose — map at parent, refactor User in one place -->
<script setup lang="ts">
defineProps<{ user: { id: number; displayName: string } }>()
</script>
```

**Reference:** Slack — Types help find call sites; shouldn't require updating every caller for a minor change.

---

### 7.7 Example: unknown vs any

```ts
// any — silently allows anything (avoid except during migration)
function process(data: any) {
  return data.foo.bar.baz  // No error. Runtime: 💥
}

// unknown — must narrow first (recommended)
function process(data: unknown) {
  if (typeof data !== 'object' || data === null) throw new Error('Invalid')
  const obj = data as { foo?: { bar?: { baz?: unknown } } }
  return obj.foo?.bar?.baz  // Or use a schema to parse
}
```

---

## 8. OpenAPI & Full-Stack Type Safety

- **Schema-first** — Define OpenAPI spec, generate types for frontend and backend.
- **Runtime validation** — Use generated Zod/Valibot schemas from spec — validate responses.
- **Backend as source of truth** — API spec is the contract. Frontend consumes it.
- **Fail fast** — Invalid response? Throw at the fetch layer. Don't let bad data spread.

---

### 8.1 Schema-First: Define Once, Generate Everywhere

Write the spec first. Types flow from it — no hand-written drift.

```yaml
# openapi.yaml — single source of truth
paths:
  /api/users/{id}:
    get:
      responses:
        '200':
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'

components:
  schemas:
    User:
      type: object
      required: [id, email]
      properties:
        id: { type: integer }
        email: { type: string, format: email }
        createdAt: { type: string, format: date-time }
```

```ts
// Generated for frontend (orval) and backend (openapi-typescript)
type User = { id: number; email: string; createdAt: string }
// Add field in spec → regenerate → both sides get it
```

---

### 8.2 Runtime Validation: Schema → Zod/Valibot

Generate runtime schemas from the spec. Validate every response.

```ts
// orval generates both types AND Zod schemas from OpenAPI
// orval.config.ts → output: { client: 'vue-query', override: { mutator: { path: './fetcher.ts' } } }

// Generated: api/generated/schemas.ts
export const UserSchema = z.object({
  id: z.number(),
  email: z.string().email(),
  createdAt: z.string().datetime(),
})
export type User = z.infer<typeof UserSchema>

// In your composable
async function fetchUser(id: number): Promise<User> {
  const res = await fetch(`/api/users/${id}`)
  const json = await res.json()
  return UserSchema.parse(json)  // Throws if backend returns wrong shape
}
```

---

### 8.3 Backend as Source of Truth

API spec is the contract. Frontend consumes; backend owns it.

```ts
// ❌ Frontend defines its own User type — drift when backend changes
interface User { id: number; email: string }

// ✅ Frontend imports from generated API client
import type { User } from '@/api/generated/schema'
import { UserSchema } from '@/api/generated/schemas'

// Backend adds "role"? Regenerate. Frontend gets User.role.
// Backend renames "email" → "emailAddress"? Regenerate. TS errors until you fix usages.
```

```ts
// Vue composable — types from spec
export function useUserApi() {
  const fetchUser = async (id: number) => {
    const res = await api.get(`/users/${id}`)
    return UserSchema.parse(res.data)  // Contract enforced
  }
  return { fetchUser }
}
```

---

### 8.4 Fail Fast: Throw at the Fetch Layer

Invalid response? Throw immediately. Don't let bad data propagate.

```ts
// ❌ Let invalid data spread — crash deep in the app
const user = await fetch('/api/user').then(r => r.json())
renderProfile(user)  // 💥 undefined.email somewhere inside

// ✅ Fail at boundary — clear error, fix API or spec
async function fetchUser(): Promise<User> {
  const res = await fetch('/api/user')
  const json = await res.json()
  const parsed = UserSchema.safeParse(json)
  if (!parsed.success) {
    throw new ApiValidationError('/api/user', parsed.error)
  }
  return parsed.data
}
```

```ts
// Centralized fetch wrapper — validates all API responses
async function apiGet<T>(path: string, schema: z.ZodType<T>): Promise<T> {
  const res = await fetch(path)
  const json = await res.json()
  return schema.parse(json)  // Throws here, not in component
}
```

---

### 8.5 End-to-End Example: OpenAPI → Vue

```yaml
# openapi.yaml
paths:
  /api/orders:
    get:
      responses:
        '200':
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/Order'
components:
  schemas:
    Order:
      type: object
      required: [id, status, total]
      properties:
        id: { type: string, format: uuid }
        status: { type: string, enum: [pending, confirmed, shipped] }
        total: { type: number }
```

```vue
<script setup lang="ts">
// Generated types + schemas
import { OrderSchema, type Order } from '@/api/generated'

const { data: orders, error } = await useAsyncData('orders', async () => {
  const res = await $fetch('/api/orders')
  return z.array(OrderSchema).parse(res)  // Validate array of orders
})
</script>

<template>
  <div v-for="order in orders" :key="order.id">
    {{ order.status }} — ${{ order.total }}
  </div>
</template>
```

One spec. Types + runtime validation. No drift.

---

## 9. Closing

> Type safety is not about perfection.  
> It's about placing guardrails where failure is expensive.

**Remember:** Strategic > dogmatic. Validate at boundaries. Don't over-type internals.

---

## References

**TypeScript & Principles:**
- https://www.typescriptlang.org/docs/handbook/intro.html
- https://www.typescriptlang.org/docs/handbook/declaration-files/do-s-and-don-ts.html
- https://github.com/microsoft/TypeScript/wiki/TypeScript-Design-Goals

**Company Practices:**
- Slack: https://slack.engineering/typescript-at-slack/
- Stripe: https://github.com/stripe/openapi | https://docs.stripe.com/stripe-data/schema
- Microsoft: TypeScript handbook (unknown vs any, strict mode)

**Runtime Validation & Schema:**
- https://zod.dev
- https://valibot.dev
- https://spec.openapis.org/oas/latest.html
- https://stevekinney.com/courses/full-stack-typescript/type-safety-vs-runtime-validation
