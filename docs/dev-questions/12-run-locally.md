# Q: How do I set up and run the game locally for development?

<!-- @tags: development, setup, bun, server, client -->
<!-- @related: docs/development.md -->

## Prerequisites

- [Bun](https://bun.sh) (latest stable) — runtime + package manager

No database, Docker, or external services required.

## Setup (one time)

```bash
# 1. Clone
git clone https://github.com/HasangerGames/suroi.git
cd suroi

# 2. Install dependencies
bun install

# 3. Create server config (required — server won't start without it)
cp server/config.example.json server/config.json
```

Default config works for local dev. You don't need to edit it unless you want
to change port, team mode, map, or plugins.

## Run the Game

```bash
bun dev
```

This starts:
- **Client** (Vite dev server) on `http://localhost:3000`
- **Server** (Bun) on `ws://localhost:8000`

The client hot-reloads on file changes. The server uses `bun --hot`.

## Run Just One Side

```bash
bun dev:client   # client only (if server already running)
bun dev:server   # server only
```

## Common Tasks

```bash
bun lint                  # auto-fix linting (Biome)
bun validateDefinitions   # validate game definitions after changes
bun validateSvgs          # validate SVG assets
bun watch:server          # TypeScript type checking (watch mode)
```

Always run `bun validateDefinitions` after modifying anything in
`common/src/definitions/`.

## Troubleshooting

**Client shows "version mismatch" / can't connect:**
Hard-refresh the browser (`Cmd+Shift+R` / `Ctrl+Shift+R`) or restart `bun dev`.

**"Cannot find config.json":**
Run `cp server/config.example.json server/config.json`.

**`@common/*` import not resolving:**
Run commands from the workspace root, not from a package subdirectory.

## Related

- [Development Guide](../development.md) — full command reference, code standards, CI
