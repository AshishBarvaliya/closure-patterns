# Detect: When closure patterns are needed

Identify code that should be refactored with closure-based patterns. For each match: **bad pattern → risk → mark for closure refactor**.

**Out of scope — do not cover these.** Adding them makes the skill worse because the agent starts explaining instead of improving code. Do not teach or explain: lexical scope tutorials; closures inside loops (as theory); functional programming theory; currying math examples; interview puzzles; “what is a closure”; partial application libraries. Focus only on detecting bad patterns and refactoring to improve reliability and predictability.

**Production usage set (what to cover).** Execution control: once / guard removal, deduplication, preventing double actions. State encapsulation: factory instead of mutable module state, request-safe logic, isolation of side effects. Performance: memoization, caching expensive work, lazy evaluation. Async stability: debounce, throttle, queued handlers. Concurrency safety: no shared state across requests, no race conditions.

---

**Boolean execution guards**  
Pattern: A module-level or outer-scope boolean (e.g. `let ran = false`) used to run logic once or gate execution.  
Risk: Reuse or testing re-runs can leave the guard in the wrong state; order of operations becomes implicit.  
→ Mark for closure refactor (encapsulate guard in a factory or returned API).

---

**Mutable module variables**  
Pattern: Top-level `let`/`var` (or reassigned `const`-backed object) that multiple functions read/write.  
Risk: Hidden coupling, race conditions, hard-to-reason order of updates, flaky tests.  
→ Mark for closure refactor (move state inside a closure or factory, expose only minimal API).

---

**Manual cache objects**  
Pattern: A plain object or `Map` at module/outer scope used as a cache, keyed by input.  
Risk: Cache never cleared or keyed by reference; grows unbounded; shared across unrelated callers.  
→ Mark for closure refactor (cache lives inside closure/factory; consider TTL or size limits).

---

**Timers stored outside functions**  
Pattern: `setTimeout`/`setInterval` IDs or timer state held in module/outer scope.  
Risk: Timers outlive their logical lifetime; can’t cancel or scope to a request/session; leaks.  
→ Mark for closure refactor (timer IDs and cleanup owned by closure/factory).

---

**Repeated expensive computation**  
Pattern: Same heavy work done on every call with no memoization, or ad-hoc caching in globals.  
Risk: Performance spikes; duplicated logic; global cache pitfalls (see manual cache objects).  
→ Mark for closure refactor (memoize inside a closure/factory keyed by input).

---

**Shared state across requests**  
Pattern: Module-level or singleton state that multiple requests/tasks/workers read and write.  
Risk: Request A’s data visible to B; non-deterministic behavior; security and correctness bugs.  
→ Mark for closure refactor (per-request or per-session state inside closure/factory).

---

**Resource lifecycle (no cleanup / unsubscribe)**  
Pattern: Subscriptions, intervals, listeners, or handles created without a paired cleanup; cleanup stored at module scope or not at all.  
Risk: Leaks when component/request ends; duplicate handlers on re-run; can’t cancel.  
→ Mark for closure refactor (factory returns do-work + cleanup; closure holds handle so cleanup can run).

---

**Request-scoped context in globals**  
Pattern: Storing request ID, user, or per-request data in module-level or singleton variables so handlers can read them.  
Risk: One request’s context used for another; race under concurrency.  
→ Mark for closure refactor (pass context into a factory; closure captures it; no global request storage).

---

**Unstable event listener callbacks**  
Pattern: Inline or recreated callbacks passed to UI (onClick, addEventListener, etc.) so identity changes every render or call.  
Risk: Re-subscribes every time; unnecessary re-renders; focus/keyboard bugs when callback reference changes.  
→ Mark for closure refactor (stable callback from closure/factory or ref; same function reference across renders).

---

**Retry/backoff state at module scope**  
Pattern: Attempt count, delay, or “last attempt” time held at top level for async retry loops.  
Risk: Multiple callers share the same attempt state; one retry overwrites another; can’t run independent retries.  
→ Mark for closure refactor (retry/backoff state inside closure; factory returns a function that does one retry flow).

---

**Lazy init / lazy value at module scope**  
Pattern: Expensive or side-effectful value computed on first access but stored in a module-level variable (e.g. getter that assigns to `let _x` once).  
Risk: Init runs at wrong time or multiple times; shared mutable cache; not request-scoped.  
→ Mark for closure refactor (closure holds value and “done” flag; first access runs computation and caches; later returns cached).

---

**Queued async work (serialized execution)**  
Pattern: Async work that must run one-at-a-time but “in progress” or queue is at module/outer scope.  
Risk: Concurrent callers interleave; lost or duplicated work; races.  
→ Mark for closure refactor (factory holds queue and processing state in closure; returned function enqueues and runs sequentially).
