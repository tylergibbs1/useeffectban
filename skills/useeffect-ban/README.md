# useEffect Ban

A skill that enforces a no-direct-`useEffect` policy in React codebases.

## What it does

Teaches AI agents and developers to replace `useEffect` with five declarative
patterns:

1. **Derive state inline** — compute values from state/props directly
2. **Data-fetching libraries** — use TanStack Query, SWR, or server components
3. **Event handlers** — handle user actions at the source
4. **useMountEffect** — one-time external system sync on mount
5. **Key-based remounting** — reset components with React's `key` prop

## Why

Direct `useEffect` usage leads to:

- Race conditions in data fetching
- Infinite render loops from dependency array mistakes
- Hard-to-debug implicit synchronization logic
- Brittle coupling hidden in dependency arrays

## Installation

```bash
npx skills add useeffect-ban
```

## References

- [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect) — React docs
- [Why We Banned useEffect](https://factoryai.com/blog/why-we-banned-useeffect) — Factory
- [Alvin Ng's thread](https://x.com/alvinsng/status/2033969062834045089) — UseFactory
