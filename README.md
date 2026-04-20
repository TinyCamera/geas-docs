# geas-docs

Public documentation for [Geas](https://github.com/TinyCamera/geas) — an agent-driven multiplayer RPG.

Site: **https://tinycamera.github.io/geas-docs/**

Built with [Jekyll](https://jekyllrb.com/) + [Just the Docs](https://just-the-docs.com/) theme, served by GitHub Pages.

## Content scope

Public-facing: what Geas is, how to connect a Claude client, what the agent-facing tool surface looks like. Deployment internals, infra, and the private roadmap live in the `geas` code repo.

## Editing

Pages are markdown at the repo root:

- `index.md` — home
- `quickstart.md` — setup + first session
- `agent-interface.md` — complete MCP tool + event reference

Changes to `main` deploy automatically via GitHub Pages.

## Local preview

```bash
bundle install
bundle exec jekyll serve
```

Opens at http://localhost:4000/geas-docs/.
