---
name: useeffect-ban
description: >-
  Enforce a no-direct-useEffect policy in React codebases. Replaces useEffect
  with derived state, event handlers, data-fetching libraries, useMountEffect,
  and key-based remounting.
license: MIT
author: Tyler Gibbs
version: 1.0.0
triggers:
  - useEffect
  - use effect
  - effect hook
  - no useEffect
  - ban useEffect
  - useeffect ban
---

# useEffect Ban

Ban direct `useEffect` calls in React components. Replace them with five
declarative patterns that eliminate race conditions, infinite loops, and
dependency-array hell.

## When to apply

- Writing or reviewing React components that contain `useEffect`
- Building new features where an agent or developer reaches for `useEffect`
- Refactoring existing code that suffers from effect-related bugs
- Onboarding contributors to a codebase that enforces this rule
- Configuring AI agents (AGENTS.md, Cursor rules, etc.) to follow the policy

## Quick reference

| # | Rule | Replaces |
|---|------|----------|
| 1 | **Derive state inline** | `useEffect(() => setX(f(y)), [y])` |
| 2 | **Use data-fetching libraries** | `useEffect(() => fetch(...).then(setState), [id])` |
| 3 | **Use event handlers** | `useEffect(() => { if (flag) doWork() }, [flag])` |
| 4 | **useMountEffect for external sync** | `useEffect(cb, [])` |
| 5 | **Reset with `key`** | `useEffect(() => reset(), [id])` |

## Rule 1: Derive state, do not sync it

Most effects that set state from other state are unnecessary and add extra
renders.

**Smell test:**
- You are about to write `useEffect(() => setX(deriveFromY(y)), [y])`
- You have state that only mirrors other state or props

```tsx
// BAD: Two render cycles
const [products, setProducts] = useState([]);
const [filtered, setFiltered] = useState([]);
useEffect(() => {
  setFiltered(products.filter((p) => p.inStock));
}, [products]);

// GOOD: Compute inline in one render
const [products, setProducts] = useState([]);
const filtered = products.filter((p) => p.inStock);
```

## Rule 2: Use data-fetching libraries

Effect-based fetching creates race conditions and duplicated caching logic.

**Smell test:**
- Your effect does `fetch(...)` then `setState(...)`
- You are re-implementing caching, retries, cancellation, or stale handling

```tsx
// BAD: Race condition risk
useEffect(() => {
  fetchProduct(id).then(setProduct);
}, [id]);

// GOOD: Library handles cancellation/caching/staleness
const { data: product } = useQuery(['product', id], () => fetchProduct(id));
```

## Rule 3: Event handlers, not effects

If a user action triggers work, do it in the handler.

**Smell test:**
- State is used as a flag so an effect can do the real action
- You are building "set flag -> effect runs -> reset flag" mechanics

```tsx
// BAD: Effect as action relay
const [liked, setLiked] = useState(false);
useEffect(() => {
  if (liked) { postLike(); setLiked(false); }
}, [liked]);
return <button onClick={() => setLiked(true)}>Like</button>;

// GOOD: Direct event-driven action
return <button onClick={() => postLike()}>Like</button>;
```

## Rule 4: useMountEffect for one-time external sync

Wrap `useEffect(..., [])` in a named hook to make intent explicit.

```tsx
export function useMountEffect(effect: () => void | (() => void)) {
  // eslint-disable-next-line react-hooks/exhaustive-deps
  useEffect(effect, []);
}
```

**Good uses:** DOM integration (focus, scroll), third-party widget lifecycles,
browser API subscriptions.

**Prefer conditional mounting over guards inside effects:**

```tsx
// BAD: Guard inside effect
useEffect(() => { if (!isLoading) playVideo(); }, [isLoading]);

// GOOD: Mount only when preconditions are met
function Wrapper({ isLoading }) {
  if (isLoading) return <Loading />;
  return <VideoPlayer />;
}
function VideoPlayer() {
  useMountEffect(() => playVideo());
}
```

## Rule 5: Reset with `key`, not dependency choreography

If the requirement is "start fresh when ID changes," use React's remount
semantics directly.

**Smell test:**
- Your effect's only job is to reset local state when an ID/prop changes
- You want the component to behave like a brand-new instance for each entity

```tsx
// BAD
useEffect(() => { loadVideo(videoId); }, [videoId]);

// GOOD
function VideoPlayer({ videoId }) {
  useMountEffect(() => loadVideo(videoId));
}
// Parent:
<VideoPlayer key={videoId} videoId={videoId} />
```

## Enforcement

### ESLint

Add to your ESLint config:

```json
{
  "rules": {
    "no-restricted-imports": ["error", {
      "paths": [{
        "name": "react",
        "importNames": ["useEffect"],
        "message": "Direct useEffect is banned. Use useMountEffect or a declarative pattern instead."
      }]
    }]
  }
}
```

### Agent guidance

Add to your `AGENTS.md` or equivalent:

```
Never use useEffect directly. Apply these patterns instead:
1. Derived state -> compute inline
2. Data fetching -> useQuery / useSWR
3. User actions -> event handlers
4. Mount-only sync -> useMountEffect
5. Reset on ID change -> key prop
```

## Full guide

See [AGENTS.md](./AGENTS.md) for the comprehensive reference with all edge
cases, migration examples, and architectural rationale.
