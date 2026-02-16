# Don’t overuse: When not to apply closure patterns

**Goal:** Avoid annoying developers. Do not suggest closure refactors in these cases.

---

**Simple scripts**  
Single-run or one-off scripts (CLI, migration, script in package.json). No reuse, no shared state across calls. Leave as-is.

**Constant values**  
`const` primitives or frozen config at module scope. No mutation, no cross-caller sharing of mutable state. No refactor.

**Configuration objects**  
Read-only config (env, options, feature flags). If it’s not mutated and not request-scoped, don’t wrap in factories.

**Correct React hooks / state**  
Proper `useState`, `useRef`, `useMemo`, or context. Don’t push closure patterns into React’s model. Only flag when there’s actual shared mutable module state or incorrect shared state.

**Trivial logic**  
Small helpers, pure functions, or logic with no shared mutable state. Adding factories or “once” wrappers here adds noise, not value.

---

If none of the detect patterns apply (no guards, manual caches, outer timers, mutable module state, shared request state, repeated expensive work, missing cleanup/unsubscribe, request context in globals, unstable UI callbacks, retry state at module scope), do not suggest a closure refactor. Do not suggest cleanup closures when the code already unsubscribes correctly. Do not suggest request-scoped factories when context is already passed per request (e.g. via middleware that attaches to `req`).
