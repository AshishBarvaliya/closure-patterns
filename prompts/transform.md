# Transform: Refactor by applying closure patterns

**You must output the refactored code. Do not give instructions or steps—rewrite the user’s code directly and preserve behavior.** Do not teach (no lexical scope, closure-in-loops theory, FP theory, currying examples, interview puzzles, “what is closure,” or partial application). Improve the code only.

---

## Transformation rules

**Guard flags** → **Once-style closures**  
Replace module/outer booleans that gate “run once” logic with a closure that holds the guard and returns a function. That function runs the logic once; later calls do nothing (or return the same result). No shared guard in outer scope.

**Manual cache** → **Memoized factory**  
Replace a top-level cache object/Map with a factory that returns a function. The cache lives inside the factory closure. The returned function does the work and caches by argument (or key). No cache at module scope.

**Timers** → **Stable debounce/throttle closures**  
Replace timers stored in module/outer scope with a factory that returns a debounced or throttled function. Timer IDs and cleanup live inside the closure. Callers use the returned function; no global timer state.

**Mutable module state** → **Factory functions**  
Replace shared module-level variables with a factory. The factory closure holds the state; it returns an object or set of functions that are the only way to read/write that state. No mutable bindings at module scope.

**Resource lifecycle** → **Cleanup/unsubscribe closures**  
Replace resources (subscriptions, intervals, listeners) that have no cleanup or cleanup at module scope with a factory. The factory closure holds the handle/subscription; it returns a function (or `{ run, cleanup }`) that does the work and a way to unsubscribe/clear. Caller invokes cleanup when the logical lifetime ends (e.g. component unmount, request done).

**Request-scoped context** → **Context-in-closure**  
Replace global or singleton “current request” storage with a factory that accepts context (requestId, user, etc.). The closure captures that context; the returned function uses it. No module-level request state. Create the returned function per request and pass it through (or create the factory per request).

**Unstable event callbacks** → **Stable callback from closure**  
Replace inline or recreated UI callbacks with a stable reference: either a function from a closure/factory that doesn’t change identity, or a ref that holds the latest handler while exposing one stable callback. Same function reference across renders so the framework doesn’t re-subscribe or re-render unnecessarily.

**Retry/backoff** → **Stateful retry closure**  
Replace module-level attempt count, delay, or last-attempt time with a factory. The closure holds retry state; the returned async function runs one retry loop (with backoff). Each caller or each call to the factory gets an independent retry flow.

**Preserve behavior**  
Inputs and outputs of the refactored code must match the original. Do not change APIs or semantics unless the user asks. Fix only structure and scope.

---

## Output

- Emit the full refactored file(s) or snippet(s).
- No “here’s how to do it” or “next, wrap in a factory”—only the transformed code and, if needed, a one-line note (e.g. “Call `createX()` once, then use the returned function.”).
