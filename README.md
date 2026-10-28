# useeffect-ban

A skill for AI agents that enforces a no-direct-`useEffect` policy in React
codebases. Based on [Factory's useEffect ban](https://factoryai.com/blog/why-we-banned-useeffect)
and the React team's [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect) guide.

## Installation

```bash
npx skills add tylergibbs1/useeffectban
```

## What it does

The skill replaces `useEffect` with five declarative patterns:

1. **Derive state inline**: compute values from state and props directly.
2. **Data-fetching libraries**: TanStack Query, SWR, or server components.
3. **Event handlers**: handle user actions at the source.
4. **`useMountEffect`**: one-time external system sync on mount.
5. **Key-based remounting**: reset components with React's `key` prop.

It also covers enforcing the ban with ESLint.

## Skills

| Skill | Description |
|-------|-------------|
| [useeffect-ban](./skills/useeffect-ban/) | Replace useEffect with five declarative patterns |

[`SKILL.md`](./skills/useeffect-ban/SKILL.md) is the main definition;
[`AGENTS.md`](./skills/useeffect-ban/AGENTS.md) is the full guide with every pattern and edge case.

## License

MIT
