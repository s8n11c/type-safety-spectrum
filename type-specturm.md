# Type Safety Is a Spectrum, Not a Boolean

> Why “100% strict TypeScript” is neither realistic nor always desirable — and how to place safety where it actually matters.

---

## 1. Core Thesis

Type safety is **not**:
- `strict: true`
- “no `any` allowed”
- maximal generics everywhere

Type safety **is**:
- **Strategic**
- **Layered**
- **Context-dependent**

> The goal is not perfect types — the goal is **preventing expensive failures**.

---

## 2. The Illusion of Type Safety in TypeScript

TypeScript:
- Exists **only at compile time**
- Is erased at runtime
- Blindly trusts external data

### Common failure sources:
- API responses
- Feature flags
- User input
- JSON parsing
- Config files
- Backend/frontend version drift

```ts
interface User {
  id: number
  email: string
}

const user: User = await fetchUser() // TS trusts this completely
```

❗ If data crosses a **network, disk, or user**, TypeScript alone is lying.

---

## 3. The Type Safety Spectrum

| Layer | Safety Level | Rationale |
|-----|-------------|----------|
| UI components | 🟡 Medium | Changes often |
| Forms | 🔴 High | User input is untrusted |
| API boundaries | 🔴🔴 Very High | External & unstable |
| Business logic | 🔴🔴 Very High | Bugs are expensive |
| Internal helpers | 🟢 Low | Refactor-friendly |

---

## 4. Compile-Time vs Runtime Safety

### Compile-time only (common mistake)
```ts
function handleUser(user: User) {
  sendEmail(user.email)
}
```

### Compile-time + Runtime (correct)
```ts
import { z } from 'zod'

const UserSchema = z.object({
  id: z.number(),
  email: z.string().email(),
})

const user = UserSchema.parse(await fetchUser())
```

> Runtime validation is type safety’s missing half.

---

## 5. Where Strict Typing Pays Off

- API contracts
- Form models
- Search, filters, pagination
- Permissions / authorization
- Domain calculations

---

## 6. Where Type Safety Hurts

Over-typing smells:
- Deep generic hierarchies
- Constraint puzzles
- Utility types used once
- Types no one understands without hovering

> Type safety that slows delivery is **anti-safety**.

---

## 7. Practical Rules of Thumb

1. Validate at boundaries, trust internally
2. Prefer runtime schemas over clever generics
3. If you can’t explain the type in 30 seconds, it’s too complex
4. Use `unknown` instead of `any`
5. Don’t type what you don’t control
6. Types should help refactoring, not block it

---

## 8. OpenAPI & Full-Stack Type Safety

- Schema-first approach
- Runtime validation on API responses
- Backend as source of truth
- Fail fast on invalid data

---

## 9. Senior-Level Mindset Shift

**Junior:** How do I type this?  
**Senior:** Should this be typed, validated, or left flexible?

---

## 10. Closing

> Type safety is not about perfection.  
> It’s about placing guardrails where failure is expensive.

---

## References

- https://www.typescriptlang.org/docs/handbook/intro.html
- https://stevekinney.com/courses/full-stack-typescript/type-safety-vs-runtime-validation
- https://zod.dev
- https://valibot.dev
- https://spec.openapis.org/oas/latest.html