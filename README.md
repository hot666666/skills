# Agent Skills

Minimal Agent Skills repository for testing install and discovery across agentic coding tools.

## Skills

- `hello-world`: a tiny example skill that confirms skill loading behavior.

## Install

Interactive install:

```bash
npx skills add hot666666/skills
```

Install only the hello-world skill:

```bash
npx skills add hot666666/skills --skill hello-world
```

Install globally for Codex:

```bash
npx skills add hot666666/skills --skill hello-world -g -a codex
```

After installing, restart the target agent if it does not pick up new skills immediately.
