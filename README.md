# Frontend Architecture Skill

A portable, **framework-agnostic** architecture style for any React or React Native frontend — packaged as an [Agent Skill](https://www.anthropic.com/news/skills) (`SKILL.md`) that Claude Code, Cursor, OpenCode, Codex, Windsurf, and other agents can read and apply.

It organizes apps into **feature modules** with **page/screen directories**, a strict **server-state vs UI-state split**, **barrel-only** cross-module imports, **co-located styles**, and clear **component-promotion** rules.

- ✅ **State-management agnostic** — Zustand, Redux Toolkit, MobX, Jotai, Valtio, or Context.
- ✅ **Styling agnostic** — Tailwind, CSS Modules, Tamagui, React Native StyleSheet, styled-components.
- ✅ **Framework agnostic** — Next.js App Router, React + Vite, Remix, and Expo / React Native.
- ✅ **Naming conventions** — interfaces prefixed with `I` (e.g. `IFeatureUiState`).

> The skill describes a **structure and a set of rules**, not a component library or a visual design. Pair it with a design/component skill for the look-and-feel.

---

## Install

With the [`skills` CLI](https://skills.sh) (works across agents):

```bash
npx skills add stareezy-1/frontend-architecture-skill --skill "frontend-architecture"
```

Or with the GitHub CLI:

```bash
gh skill install stareezy-1/frontend-architecture-skill
```

Or copy it manually into your agent's skills directory:

```bash
# Claude Code (project scope)
mkdir -p .claude/skills && cp -r skills/frontend-architecture .claude/skills/

# Cursor / others that read .ai/skills
mkdir -p .ai/skills && cp -r skills/frontend-architecture .ai/skills/
```

---

## What's inside

```
skills/
└── frontend-architecture/
    └── SKILL.md     ← the skill (frontmatter + instructions)
```

The skill covers:

1. **Five core ideas** — feature modules, page directories, state-by-origin, barrel imports, promotion.
2. **Directory layout** — identical across frameworks.
3. **Feature modules** — the barrel as a contract.
4. **State split** — server (query layer) vs UI (client store), with examples for Zustand / Redux Toolkit / MobX / Jotai.
5. **Co-located styling** — Tailwind / CSS Modules / Tamagui / StyleSheet.
6. **Naming conventions** — `I`-prefixed interfaces and more.
7. **Framework adapters** — Next.js, Vite SPA, Remix, Expo / React Native.
8. **Review checklist** + **component promotion ladder**.

---

## When the agent should use it

- Scaffolding a new app.
- Adding a feature or module.
- Deciding where a component, hook, or piece of state belongs.
- Naming types and interfaces.
- Reviewing folder structure and import boundaries.

---

## License

[MIT](./LICENSE) — use, copy, and adapt freely.
