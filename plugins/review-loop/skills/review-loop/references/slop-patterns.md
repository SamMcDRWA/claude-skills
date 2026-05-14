# AI-slop and bloat patterns to flag

Load this when reviewing files for the **Code bloat / AI slop** finding category. These patterns are present in 80-100% of AI-assisted codebases (OX Security 2025 study of 300 repos; GitClear found duplicate code blocks grew 4-8× between 2020 and 2025).

The core insight from 2025 research: **AI slop is hard to spot because it looks polished — consistent naming, tests present, docs present — but violates architectural fit.** Locally correct, globally broken. Your job is to spot the global breaks.

---

## The 5 strongest signals — flag these first

1. **Shallow module** (Ousterhout) — interface complexity is roughly equal to or larger than the body. One-line wrapper functions, pass-through classes, files with more exports than logic. **Test:** can you describe what the module hides without reading it? If no, it's too shallow.

2. **Comment density exceeds the file's norm** — comments that narrate code rather than explain *why*. AI repos average 90-100% rate of restate-the-code comments.

3. **Defensive code where types/upstream guarantee safety** — `if (typeof x === 'string')` when TypeScript already says `string`. `try/catch` around code that cannot throw. Null guards on already-validated values.

4. **Semantic duplication** — a new helper/util that already exists elsewhere in the codebase (the AI didn't search before creating). Often the new one is slightly worse than the existing one.

5. **Convention drift** — new file uses a different DB-access, logging, error, validation, or HTTP-client pattern than the rest of the repo. "Every module works individually but nothing fits together."

---

## Full checklist by category

### Defensive/error-handling slop

- [ ] `try/catch` around code that cannot throw (pure functions, string ops, etc.)
- [ ] `try/catch` that swallows errors silently (`catch (e) {}` or `catch { return null }`)
- [ ] Null/undefined guards on values already validated upstream
- [ ] Optional chaining (`?.`) on values the type system guarantees non-null
- [ ] "This should never happen" branches and impossible-state defaults — assertions are better than branches
- [ ] `if (x === undefined || x === null)` instead of `if (x == null)` when both are equivalent

### Comment slop

- [ ] Comments that narrate what the next line literally does (`// increment counter` above `counter++`)
- [ ] JSDoc/docstrings on trivial getters or setters
- [ ] Function-name repetition in prose (`/** Gets the user. */ function getUser()`)
- [ ] Section banner comments (`// ===== HELPERS =====`) inconsistent with rest of repo
- [ ] Inline comments at a density much higher than the surrounding files

### Abstraction/structure slop

- [ ] Wrapper functions around a single native API call with no added behaviour
- [ ] New utility file when an existing one already covers the case
- [ ] Interface + single implementation + factory for a one-off thing
- [ ] Generic `<T>` parameters used at exactly one call site
- [ ] Configuration objects with one field
- [ ] `options` arguments where every caller passes the defaults
- [ ] Premature pluralisation: `processItems([single])` patterns where only one is ever passed
- [ ] Builder/factory patterns for objects with 2-3 fields

### Dead/over-produced code

- [ ] Exports never imported elsewhere (run `npx knip` or `npx ts-prune` if available)
- [ ] Public methods used only by tests
- [ ] Two functions doing the same thing under different names
- [ ] `useEffect`/lifecycle hooks with empty cleanup or no-op bodies
- [ ] "Future-proofing" enum values, parameters, or branches with no current caller
- [ ] Commented-out code blocks
- [ ] Imported modules never referenced

### Convention drift

- [ ] New file uses a different logging pattern than rest of repo
- [ ] New DB query style (raw SQL where the rest uses an ORM, or vice versa)
- [ ] Different error-response shape than other API routes
- [ ] Wrong date/HTTP/validation library when the repo already has one
- [ ] Re-implementing auth, formatting, or constants that exist in `lib/`

### Test slop

- [ ] Tests that assert on mocks (mock returns X, test asserts X — tautology)
- [ ] Tests that exist but skip the actual edge case named in the PR
- [ ] Snapshot tests on AI-generated structures (these lock in the slop)
- [ ] Tests for trivial getters/setters

---

## How to surface a slop finding

Use the standard finding format with severity LOW or MEDIUM (slop is rarely CRITICAL/HIGH):

```
### [SEVERITY: MEDIUM] Wrapper function adds no behaviour
**File:** lib/users.ts:42
**Issue:** `getUserEmail(u)` just returns `u.email` — single call site, no validation, no transformation
**Impact:** Extra indirection makes the call site harder to read; every reader has to verify the wrapper does nothing
**Fix:** Inline at the call site, delete the wrapper
```

Prefer fixes that **delete** code over fixes that add code. Slop is removed, not refactored.

---

## Reference frameworks

- **OX Security 10 anti-patterns**: Comments Everywhere, By-the-Book Fixation, Over-Specification, Avoidance of Refactors, Bugs Déjà-Vu (semantic duplication), etc.
- **Addy Osmani's 70%/80% problem**: AI nails the scaffolding, fails on the last 20-30% (edges, integration, security). Review effort should concentrate there.
- **Simon Willison's bar**: "If you've reviewed, tested, and understood every line, it's not vibe coding." Your job as reviewer is to enforce this.
- **DHH's principle (Pragmatic Engineer 2026)**: prefer deletion over abstraction.
