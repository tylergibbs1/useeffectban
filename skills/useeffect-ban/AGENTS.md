# useEffect Ban — Complete Agent Guide

## Purpose

This document is the authoritative reference for AI agents and developers
operating under a **no-direct-useEffect** policy. Every `useEffect` call in
application code must be replaced with one of five declarative patterns.

The only permitted wrapper is `useMountEffect`, defined once in the codebase.

---

## The useMountEffect hook

```typescript
export function useMountEffect(effect: () => void | (() => void)) {
  // eslint-disable-next-line react-hooks/exhaustive-deps
  useEffect(effect, []);
}
```

This is the **only** place `useEffect` is called directly. All other usage goes
through this hook or is eliminated entirely.

---

## Why useEffect is banned

### Production failure modes

1. **Brittleness** — Dependency arrays hide coupling. A seemingly unrelated
   refactor can silently change effect behavior.
2. **Infinite loops** — `state update -> render -> effect -> state update` loops
   are easy to create, especially when dependency lists are "fixed"
   incrementally.
3. **Dependency hell** — Effect chains (A sets state that triggers B) are
   time-based control flow. They are hard to trace and easy to regress.
4. **Debugging pain** — "Why did this run?" or "Why did this not run?" has no
   clear entrypoint like an event handler does.
5. **Agent amplification** — AI agents add `useEffect` "just in case," seeding
   the next race condition or infinite loop.

### Failure mode comparison

| Pattern | Failure mode |
|---------|-------------|
| `useMountEffect` | Binary and loud — it ran once, or not at all |
| Direct `useEffect` | Gradual degradation — flaky behavior, perf issues, loops before hard failure |

---

## The five replacement patterns

### Pattern 1: Derive state inline

**When:** You are computing a value from existing state or props.

**Principle:** If value B is a pure function of value A, compute B inline.
Do not store it in state and sync it with an effect.

```tsx
// ---- BAD ----
function ProductList() {
  const [products, setProducts] = useState([]);
  const [filteredProducts, setFilteredProducts] = useState([]);

  useEffect(() => {
    setFilteredProducts(products.filter((p) => p.inStock));
  }, [products]);

  return <List items={filteredProducts} />;
}

// ---- GOOD ----
function ProductList() {
  const [products, setProducts] = useState([]);
  const filteredProducts = products.filter((p) => p.inStock);

  return <List items={filteredProducts} />;
}
```

**Loop hazard example:**

```tsx
// ---- BAD: total in deps can loop ----
function Cart({ subtotal }) {
  const [tax, setTax] = useState(0);
  const [total, setTotal] = useState(0);

  useEffect(() => {
    setTax(subtotal * 0.1);
  }, [subtotal]);

  useEffect(() => {
    setTotal(subtotal + tax);
  }, [subtotal, tax, total]); // total in deps = infinite loop

  return <span>{total}</span>;
}

// ---- GOOD ----
function Cart({ subtotal }) {
  const tax = subtotal * 0.1;
  const total = subtotal + tax;

  return <span>{total}</span>;
}
```

**Expensive computations:** Use `useMemo` for costly derivations, not
`useEffect` + `setState`.

```tsx
// GOOD: Memoized derivation
const filtered = useMemo(
  () => products.filter((p) => p.inStock),
  [products]
);
```

**Smell test:**
- You are about to write `useEffect(() => setX(deriveFromY(y)), [y])`
- You have state that only mirrors other state or props
- You have chained effects where one sets state consumed by another

---

### Pattern 2: Use data-fetching libraries

**When:** You need to fetch data from an API based on props or state.

**Principle:** Data fetching has solved problems (caching, cancellation, race
conditions, retries, stale-while-revalidate) that you should not re-implement
in an effect.

```tsx
// ---- BAD: Race condition risk ----
function ProductPage({ productId }) {
  const [product, setProduct] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    setLoading(true);
    fetchProduct(productId).then((data) => {
      setProduct(data);  // May set stale data if productId changed
      setLoading(false);
    });
  }, [productId]);
}

// ---- GOOD: TanStack Query ----
function ProductPage({ productId }) {
  const { data: product, isLoading } = useQuery({
    queryKey: ['product', productId],
    queryFn: () => fetchProduct(productId),
  });
}

// ---- GOOD: SWR ----
function ProductPage({ productId }) {
  const { data: product, isLoading } = useSWR(
    `/api/products/${productId}`,
    fetcher
  );
}

// ---- GOOD: Next.js Server Component ----
async function ProductPage({ params }) {
  const product = await fetchProduct(params.productId);
  return <ProductView product={product} />;
}
```

**Smell test:**
- Your effect does `fetch(...)` then `setState(...)`
- You are re-implementing caching, retries, cancellation, or stale handling
- You have loading/error state managed alongside the fetch effect

---

### Pattern 3: Event handlers, not effects

**When:** A user action or discrete event should trigger work.

**Principle:** If there is a clear cause (click, submit, keypress, message),
do the work at the point of the cause. Do not relay it through state + effect.

```tsx
// ---- BAD: Effect as action relay ----
function LikeButton() {
  const [liked, setLiked] = useState(false);

  useEffect(() => {
    if (liked) {
      postLike();
      setLiked(false);
    }
  }, [liked]);

  return <button onClick={() => setLiked(true)}>Like</button>;
}

// ---- GOOD: Direct event-driven action ----
function LikeButton() {
  return <button onClick={() => postLike()}>Like</button>;
}
```

**Form submission example:**

```tsx
// ---- BAD ----
function ContactForm() {
  const [submitted, setSubmitted] = useState(false);
  const [formData, setFormData] = useState({});

  useEffect(() => {
    if (submitted) {
      sendForm(formData);
      setSubmitted(false);
    }
  }, [submitted, formData]);

  return (
    <form onSubmit={(e) => { e.preventDefault(); setSubmitted(true); }}>
      ...
    </form>
  );
}

// ---- GOOD ----
function ContactForm() {
  const [formData, setFormData] = useState({});

  function handleSubmit(e) {
    e.preventDefault();
    sendForm(formData);
  }

  return <form onSubmit={handleSubmit}>...</form>;
}
```

**Smell test:**
- State is used as a flag so an effect can do the real action
- You are building "set flag -> effect runs -> reset flag" mechanics
- The effect is responding to a discrete event, not continuous synchronization

---

### Pattern 4: useMountEffect for one-time external sync

**When:** You need to synchronize with an external system exactly once on
mount and clean up on unmount.

**Principle:** Some side effects are inherently imperative (DOM manipulation,
third-party libraries, browser APIs). Wrap them in `useMountEffect` to make
intent explicit and keep the codebase searchable.

```tsx
// ---- GOOD: DOM integration ----
function AutoFocusInput() {
  const ref = useRef<HTMLInputElement>(null);

  useMountEffect(() => {
    ref.current?.focus();
  });

  return <input ref={ref} />;
}

// ---- GOOD: Third-party widget ----
function MapWidget({ center }) {
  const containerRef = useRef<HTMLDivElement>(null);

  useMountEffect(() => {
    const map = new MapLibrary(containerRef.current!, { center });
    return () => map.destroy();
  });

  return <div ref={containerRef} />;
}

// ---- GOOD: Browser API subscription ----
function WindowSize() {
  const [size, setSize] = useState({ w: innerWidth, h: innerHeight });

  useMountEffect(() => {
    const handler = () => setSize({ w: innerWidth, h: innerHeight });
    window.addEventListener('resize', handler);
    return () => window.removeEventListener('resize', handler);
  });

  return <span>{size.w} x {size.h}</span>;
}
```

**Conditional mounting — prefer tree structure over guards:**

```tsx
// ---- BAD: Guard inside effect ----
function VideoPlayer({ isLoading }) {
  useEffect(() => {
    if (!isLoading) playVideo();
  }, [isLoading]);
}

// ---- GOOD: Mount only when preconditions are met ----
function VideoPlayerWrapper({ isLoading }) {
  if (isLoading) return <LoadingScreen />;
  return <VideoPlayer />;
}

function VideoPlayer() {
  useMountEffect(() => playVideo());
}

// ---- ALSO GOOD: Persistent shell + conditional instance ----
function VideoPlayerContainer({ isLoading }) {
  return (
    <>
      <VideoPlayerShell isLoading={isLoading} />
      {!isLoading && <VideoPlayerInstance />}
    </>
  );
}

function VideoPlayerInstance() {
  useMountEffect(() => playVideo());
}
```

**Smell test:**
- You are synchronizing with an external system
- The behavior is naturally "setup on mount, cleanup on unmount"
- You have an effect with an empty dependency array

---

### Pattern 5: Reset with `key`, not dependency choreography

**When:** A component should start completely fresh when an identifier changes.

**Principle:** React already has a mechanism for "destroy and recreate" — the
`key` prop. Use it instead of writing an effect that manually resets state.

```tsx
// ---- BAD ----
function VideoPlayer({ videoId }) {
  const [progress, setProgress] = useState(0);

  useEffect(() => {
    setProgress(0);
    loadVideo(videoId);
  }, [videoId]);
}

// ---- GOOD ----
function VideoPlayer({ videoId }) {
  const [progress, setProgress] = useState(0);

  useMountEffect(() => {
    loadVideo(videoId);
  });
}

// Parent:
function VideoPage({ videoId }) {
  return <VideoPlayer key={videoId} videoId={videoId} />;
}
```

**Chat room example:**

```tsx
// ---- BAD ----
function ChatRoom({ roomId }) {
  const [messages, setMessages] = useState([]);

  useEffect(() => {
    setMessages([]);
    const unsub = subscribeToRoom(roomId, setMessages);
    return unsub;
  }, [roomId]);
}

// ---- GOOD ----
function ChatRoom({ roomId }) {
  const [messages, setMessages] = useState([]);

  useMountEffect(() => {
    return subscribeToRoom(roomId, setMessages);
  });
}

// Parent:
function ChatPage({ roomId }) {
  return <ChatRoom key={roomId} roomId={roomId} />;
}
```

**Smell test:**
- Your effect's only job is to reset local state when an ID/prop changes
- You want the component to behave like a brand-new instance for each entity
- You are writing cleanup logic that mirrors setup logic inside the same effect

---

## Decision flowchart

When you encounter or are about to write a `useEffect`, follow this decision
tree:

1. **Is the value derived from state/props?** → Compute inline (Pattern 1)
2. **Is it fetching data?** → Use a data-fetching library (Pattern 2)
3. **Is it responding to a user action?** → Move to event handler (Pattern 3)
4. **Is it one-time setup/teardown on mount?** → `useMountEffect` (Pattern 4)
5. **Does the component need to reset when an ID changes?** → Use `key` (Pattern 5)
6. **None of the above?** → Reconsider whether the effect is necessary at all.
   If it truly is, use `useMountEffect` with a comment explaining why.

---

## Migration guide

### Step 1: Audit

Find all `useEffect` calls:

```bash
grep -rn "useEffect" --include="*.tsx" --include="*.ts" --include="*.jsx" --include="*.js" src/
```

### Step 2: Categorize

For each call, determine which pattern applies using the decision flowchart.

### Step 3: Add the useMountEffect hook

Create `src/hooks/useMountEffect.ts`:

```typescript
import { useEffect } from 'react';

/**
 * Runs an effect exactly once on mount with optional cleanup on unmount.
 * This is the only sanctioned way to call useEffect in this codebase.
 */
export function useMountEffect(effect: () => void | (() => void)) {
  // eslint-disable-next-line react-hooks/exhaustive-deps
  useEffect(effect, []);
}
```

### Step 4: Add ESLint enforcement

```json
{
  "rules": {
    "no-restricted-imports": ["error", {
      "paths": [{
        "name": "react",
        "importNames": ["useEffect"],
        "message": "Direct useEffect is banned. Use useMountEffect from '@/hooks/useMountEffect' or a declarative pattern. See AGENTS.md for alternatives."
      }]
    }]
  }
}
```

### Step 5: Refactor incrementally

Replace each `useEffect` with the appropriate pattern. Prioritize:
1. Effects that have caused production bugs
2. Effects with complex dependency arrays
3. Effects that chain with other effects
4. Simple derivations (quick wins)

---

## Edge cases and exceptions

### Custom hooks that wrap useEffect internally

Library-level hooks (e.g., `useEventListener`, `useInterval`,
`useMediaQuery`) may use `useEffect` internally. This is acceptable because:
- The effect logic is encapsulated behind a clear interface
- The dependency management is tested and centralized
- Application code never sees `useEffect` directly

### Server components

Server components cannot use hooks at all. This rule applies only to client
components (`"use client"`).

### Refs that need post-render measurement

```tsx
// This is a valid useMountEffect use case
function Tooltip({ children }) {
  const ref = useRef<HTMLDivElement>(null);
  const [position, setPosition] = useState({ x: 0, y: 0 });

  useMountEffect(() => {
    if (ref.current) {
      const rect = ref.current.getBoundingClientRect();
      setPosition({ x: rect.x, y: rect.y });
    }
  });

  return <div ref={ref} style={{ left: position.x, top: position.y }}>{children}</div>;
}
```

### Animation frame loops

```tsx
function AnimatedValue({ target }) {
  const [value, setValue] = useState(target);

  useMountEffect(() => {
    let frame: number;
    function animate() {
      setValue((v) => v + (target - v) * 0.1);
      frame = requestAnimationFrame(animate);
    }
    frame = requestAnimationFrame(animate);
    return () => cancelAnimationFrame(frame);
  });
}
// Note: For animations that depend on changing props, use key-based
// remounting (Pattern 5) or a dedicated animation library.
```

---

## Architectural benefit

Banning `useEffect` is a forcing function for better component tree design:

- **Parents** own orchestration and lifecycle boundaries
- **Children** can assume preconditions are already met
- **Each component** does one job (Unix philosophy)
- **Coordination** happens at clear boundaries, not inside hidden effect chains

This produces simpler components, fewer hidden side effects, and a codebase
that is easier to reason about for both humans and AI agents.
