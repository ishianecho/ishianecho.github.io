# Post taxonomy cheat sheet

Existing `categories` (English, broad → narrow) and `tags` (lowercase English) harvested from `content/posts/`.
Per the style guide: reuse these before creating new ones; new entries only when nothing fits.
Snapshot taken 2026-10-17 — refresh with:

```bash
for f in content/posts/*.md; do sed -n 's/^categories: //p;s/^tags: //p' "$f"; done | sort -u
```

## Categories

- `[CS Fundamentals, ...]`: `Calculus`, `COA`, `Compilers`, `Concrete Math`, `DBMS`, `Discrete Math`, `Linear Algebra`, `Network`, `OS`, `Physics`, `Probability Math`, `Statistics`
- `[Algorithm, ...]`: `Trick`, `Data Structure`, `DP`, `Math`, `Graph Algo`, `String Algo`, `Tree Algo`, `Game Theory`
- `[Programming, ...]`:
  - `C/Cpp` (+ `Qt` as 3rd level)
  - `Go`: `Spec & Stdlib` for language/stdlib, `3P Framework` (+ `Gin`, `Zap`, `Viper`, `Database` as 3rd level)
  - `Java`: `Java SE`, `Java EE`, `3P Framework`, `Spring framework`
  - `Python`: `Spec & Stdlib`, `Data Science Utils`
  - plain: `JavaScript`, `Markup Languages`, `MATLAB`, `Rlang`, `Rust`, `SQL`
- `[Infra, ...]`:
  - `Linux` (all Linux/CLI/shell posts)
  - `Container` (+ `Docker`, `Helm`, `Kubernetes`), `CI/CD`, `Distributed System`, `Observability`, `Virtual Machine` (+ `WSL`)
- `[Tools & Utilities, ...]`:
  - `Editor` (+ `Vim`, `Neovim`, `VS Code`), `Version Control` (+ `Git`), `3P Tools` (+ `Regex`, `Cloudflare`, `ntfy` as 3rd level)
- `[ML/DL]`, `[Games]` stand alone

## Tags

`algorithm` `android` `claude-code` `cloudflare` `cpp` `cs` `database` `devops` `gaming` `go` `graphics` `https` `java` `linux` `math` `ml` `nginx` `ntfy` `physics` `push` `python` `regex` `reverse-proxy` `rust` `tls` `tools` `vps` `web` `wayland`

## Conventions observed

- Linux command/tool posts: `[Infra, Linux]` + `[linux]`, plus a topical tag when one exists (`wayland`, `graphics`)
- Standalone third-party tools (like `regex.md`): `[Tools & Utilities, 3P Tools, <Name>]` + `[tools, <name>]`
- Editors: `[Tools & Utilities, Editor, <Name>]` + `[tools]`
- Language posts: `[Programming, <Lang>, <Sub>]` + `[<lang>]`
