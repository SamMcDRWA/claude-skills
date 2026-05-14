# Size and complexity heuristics

Load this in Phase 0 (Orient) to identify oversized files worth investigating, and in Phase 4 (Self-review) to verify your fixes didn't blow up file size.

These are **signals to investigate**, not hard limits. A 200-line file that does one cohesive thing is fine. A 60-line file with three unrelated concerns is not.

---

## File size

| Lines | What it means |
|-------|---------------|
| < 150 | Healthy, usually does one thing well |
| 150–500 | The "AI-edit sweet spot" — large enough to be useful, small enough that an LLM can hold the whole context |
| 500–1000 | Investigate: does this file have one responsibility, or several? |
| 1000+ | Almost always a smell — likely multiple modules glued together |

When you find a 500+ line file during Orient, note it. You may not have time to investigate it in this pass, but flag it for **Remaining surface** in the loop's coverage tracking.

---

## Function size

Mark Seemann's "80/24 rule" (from *Code That Fits in Your Head*): a function that doesn't fit in an 80×24 terminal window is too big. About 60 executable lines is the comfortable upper bound.

| Lines | What it means |
|-------|---------------|
| < 20 | Healthy |
| 20–60 | Investigate at the second sign of trouble |
| 60+ | Likely doing more than one thing — flag for refactor |

---

## Complexity

**Cyclomatic complexity > 7 per function** is the modern consensus threshold (Mark Seemann; ties to working-memory limits). If a function has more than 7 independent paths, a reader can't hold them all in their head while reasoning about it.

How to estimate without tools: count `if`, `else if`, `case`, `&&`, `||`, `?:`, `catch`, and loop conditions inside the function body. Each is one path. Eight or more is a smell.

---

## Parameters

Sandi Metz: **more than 4 parameters is a smell.** When you see 5+, ask whether some belong to a config object that represents "one concept," or whether the function is doing too many unrelated things.

---

## Nesting depth

**More than 3-4 levels of nesting** is a refactor signal. Common fix: early-return guards instead of pyramid `if/else`.

---

## The deep-modules test (Ousterhout) — the strongest qualitative check

A module is **deep** when its interface (public surface) is small compared to the functionality it hides. A module is **shallow** when the interface is roughly the same size as the body.

**Shallow module example (bad):**
```ts
export function getUserEmail(u: User): string {
  return u.email
}
```
Public surface: 1 function. Hidden functionality: zero. Shallow — inline it.

**Deep module example (good):**
```ts
export function authenticateRequest(req: Request): User | null { ... 200 lines of cookie parsing, JWT verification, DB lookup, rate-limit ... }
```
Public surface: 1 function. Hidden functionality: a lot. Deep — keep it.

**Test for any module you encounter:** can you describe what the module *hides* without reading it? If no, it's too shallow and should be inlined or merged.

---

## Source

- Mark Seemann — *Code That Fits in Your Head* (2021), 80/24 rule, cyclomatic 7
- Sandi Metz — Rules for Developers (parameters, class size)
- John Ousterhout — *A Philosophy of Software Design* (deep vs shallow modules)
- Eamonn Faherty / 2025 community consensus — 150-500 line sweet spot for AI-assisted editing
- Martin Fowler — *FunctionLength* bliki
