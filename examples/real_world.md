# Real-world before/after examples

No toy counters. These illustrate the detect → transform rules on realistic code.

---

## 1. Prevent double form submit

**Before (guard flag at module scope)**  
Risk: Reuse or SPA navigation can leave `submitted` true; second form never submits.

```javascript
let submitted = false;

async function handleSubmit(e) {
  e.preventDefault();
  if (submitted) return;
  submitted = true;
  await fetch('/api/order', { method: 'POST', body: new FormData(e.target) });
}
```

**After (once-style closure)**  
Guard lives inside the closure; each form (or each call to `createSubmitHandler`) gets its own guard.

```javascript
function createSubmitHandler() {
  let submitted = false;
  return async function handleSubmit(e) {
    e.preventDefault();
    if (submitted) return;
    submitted = true;
    await fetch('/api/order', { method: 'POST', body: new FormData(e.target) });
  };
}

const handleSubmit = createSubmitHandler();
```

---

## 2. Deduplicate API requests

**Before (manual cache at module scope)**  
Risk: Cache shared by all callers; no per-request or per-key isolation; grows unbounded.

```javascript
const pending = {};

async function getUserId(name) {
  if (pending[name]) return pending[name];
  const p = fetch(`/api/user?name=${name}`).then(r => r.json());
  pending[name] = p;
  return p;
}
```

**After (memoized factory)**  
Cache and pending map live inside the closure; returned function is the only way to call.

```javascript
function createUserFetcher() {
  const pending = {};
  return async function getUserId(name) {
    if (pending[name]) return pending[name];
    const p = fetch(`/api/user?name=${name}`).then(r => r.json());
    pending[name] = p;
    return p;
  };
}

const getUserId = createUserFetcher();
```

---

## 3. Cache DB query

**Before (module-level cache)**  
Risk: Shared across requests; stale data; no TTL or invalidation scope.

```javascript
const userCache = {};

async function getUser(db, id) {
  if (userCache[id]) return userCache[id];
  const user = await db.query('SELECT * FROM users WHERE id = ?', [id]);
  userCache[id] = user;
  return user;
}
```

**After (memoized factory)**  
Cache is per-factory instance. Create one per request/session for request-scoped caching, or one per app for a single shared cache with clear ownership.

```javascript
function createUserLoader(db) {
  const cache = {};
  return async function getUser(id) {
    if (cache[id]) return cache[id];
    const user = await db.query('SELECT * FROM users WHERE id = ?', [id]);
    cache[id] = user;
    return user;
  };
}

const getUser = createUserLoader(db);
```

---

## 4. Rate limit handler

**Before (timers and state at module scope)**  
Risk: One global throttle for all callers; can’t scope to user or route; cleanup is unclear.

```javascript
let lastCall = 0;
let timerId = null;

function sendAnalytics(event) {
  const now = Date.now();
  if (now - lastCall < 1000) {
    clearTimeout(timerId);
    timerId = setTimeout(() => sendAnalytics(event), 1000 - (now - lastCall));
    return;
  }
  lastCall = now;
  fetch('/api/analytics', { method: 'POST', body: JSON.stringify(event) });
}
```

**After (stable throttle closure)**  
Timer and last-call state live inside the factory; one throttle per returned function; no global state.

```javascript
function createThrottledAnalytics(ms = 1000) {
  let lastCall = 0;
  let timerId = null;
  return function sendAnalytics(event) {
    const now = Date.now();
    if (now - lastCall < ms) {
      clearTimeout(timerId);
      timerId = setTimeout(() => {
        lastCall = Date.now();
        fetch('/api/analytics', { method: 'POST', body: JSON.stringify(event) });
      }, ms - (now - lastCall));
      return;
    }
    lastCall = now;
    fetch('/api/analytics', { method: 'POST', body: JSON.stringify(event) });
  };
}

const sendAnalytics = createThrottledAnalytics(1000);
```

---

## 5. Resource lifecycle: cleanup / unsubscribe

**Before (subscription, no cleanup)**  
Risk: Listener never removed; leaks on unmount; duplicate handlers if effect re-runs.

```javascript
function useData() {
  const [data, setData] = useState(null);
  useEffect(() => {
    const sub = eventBus.on('data', setData);
    return () => {}; // forgot to unsubscribe
  }, []);
  return data;
}
```

**After (closure holds subscription; cleanup in return)**  
Subscription handle lives in effect scope; cleanup function closes over it and unsubscribes.

```javascript
function useData() {
  const [data, setData] = useState(null);
  useEffect(() => {
    const sub = eventBus.on('data', setData);
    return () => sub.unsubscribe();
  }, []);
  return data;
}
```

Or with a factory when the subscription is created outside React:

```javascript
function createDataSubscription(eventBus) {
  let sub = null;
  return {
    start(onData) {
      if (sub) sub.unsubscribe();
      sub = eventBus.on('data', onData);
    },
    stop() {
      if (sub) sub.unsubscribe();
      sub = null;
    },
  };
}
```

---

## 6. Request-scoped context (no globals)

**Before (global request context)**  
Risk: Concurrent requests overwrite `currentRequest`; handler for B may see A’s user.

```javascript
let currentRequest = null;

function middleware(req, res, next) {
  currentRequest = { id: req.id, user: req.user };
  next();
}

function getLogger() {
  return {
    log(msg) {
      console.log(currentRequest.id, currentRequest.user, msg);
    },
  };
}
```

**After (context passed into closure)**  
No global. Factory receives request context; returned API closes over it.

```javascript
function createRequestContext(requestId, user) {
  return {
    log(msg) {
      console.log(requestId, user, msg);
    },
  };
}

function middleware(req, res, next) {
  req.context = createRequestContext(req.id, req.user);
  next();
}
```

---

## 7. Event listener stability (UI)

**Before (new callback every render)**  
Risk: Child re-renders every time because onClick identity changes; focus/keyboard issues.

```javascript
function SearchInput() {
  const [q, setQ] = useState('');
  return (
    <input
      value={q}
      onChange={e => setQ(e.target.value)}
      onKeyDown={e => {
        if (e.key === 'Enter') doSearch(q);
      }}
    />
  );
}
```

**After (stable callback via ref or closure)**  
Stable key-down handler; reads latest `q` from ref so identity doesn’t change.

```javascript
function SearchInput() {
  const [q, setQ] = useState('');
  const qRef = useRef(q);
  qRef.current = q;
  const handleKeyDown = useCallback(e => {
    if (e.key === 'Enter') doSearch(qRef.current);
  }, []);
  return (
    <input
      value={q}
      onChange={e => setQ(e.target.value)}
      onKeyDown={handleKeyDown}
    />
  );
}
```

---

## 8. Retry & backoff controller

**Before (retry state at module scope)**  
Risk: Two callers retrying share `attempts` and `lastAttempt`; one overwrites the other.

```javascript
let attempts = 0;
let lastAttempt = 0;

async function fetchWithRetry(url) {
  while (attempts < 3) {
    try {
      const res = await fetch(url);
      return res.json();
    } catch (e) {
      attempts++;
      const delay = Math.min(1000 * 2 ** (attempts - 1), 10000);
      await new Promise(r => setTimeout(r, delay));
    }
  }
  throw new Error('Failed after retries');
}
```

**After (stateful retry in closure)**  
Each call to `createRetryFetcher()` gets its own retry state; safe for concurrent use.

```javascript
function createRetryFetcher(maxAttempts = 3) {
  return async function fetchWithRetry(url) {
    let attempts = 0;
    while (attempts < maxAttempts) {
      try {
        const res = await fetch(url);
        return res.json();
      } catch (e) {
        attempts++;
        const delay = Math.min(1000 * 2 ** (attempts - 1), 10000);
        await new Promise(r => setTimeout(r, delay));
      }
    }
    throw new Error('Failed after retries');
  };
}

const fetchWithRetry = createRetryFetcher(3);
```
