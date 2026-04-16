# Q5: How is the common/ package shared between client and server — is it compiled separately for each, or bundled differently?

**Answer:** The `common/` package is **shared as source TypeScript, not compiled separately**. Both client and server import directly from `common/src/` via path aliases, and each runtime compiles it independently as part of their own build process.

---

## Compilation & Bundling Strategy

### How `common/` is Consumed

**No intermediate package build.** The `common/` package:
- Is marked `"private": true` in `common/package.json` (cannot be published to npm)
- Has **no build or compilation scripts** defined
- Contains only TypeScript source files in `common/src/`
- Is NOT listed as a dependency in `client/package.json` or `server/package.json`

Both client and server instead use **TypeScript path aliases** to import source directly.

### Client Build (Vite)

**Path alias resolution:**
- `client/tsconfig.json` defines: `"@common/*": ["../common/src/*"]`
- `client/vite/vite.common.ts` replicates: `alias: { "@common": path.resolve(__dirname, "../../common/src") }`

**During Vite build:**
1. Vite encounters imports like `import { GameConstants } from "@common/constants"`
2. The path alias resolves to `../common/src/constants.ts` (raw TypeScript source)
3. Vite's TypeScript loader (`esbuild`) processes both client code *and* common source together
4. Result: Common code is **bundled inline** into the final client bundle (e.g., `client/dist/scripts/main-[hash].js`)
5. The common code is **NOT minified or chunked separately**—it's merged with client code as a single Vite build

**Vite configuration details:**
- No special handling for `@common/*` imports
- No separate entry point for common
- Standard Vite tree-shaking applies to unused common exports

### Server Build (Bun Runtime)

**Path alias resolution:**
- `server/tsconfig.json` defines: `"@common/*": ["../common/src/*"]`

**At runtime:**
1. Dev mode: `bun --hot src/server.ts` runs TypeScript directly
   - Bun's native TypeScript loader respects the `tsconfig.json` path alias
   - Common source is **parsed and imported on-the-fly** as raw `.ts` files
   - No intermediate compilation step (Bun JIT-compiles both server and common code)

2. Prod mode: `bun .` (which runs `src/server.ts`)
   - Same behavior—Bun runs TypeScript directly without a build step
   - No bundling. Common code is loaded as separate modules in Bun's module cache

**No build step for server.** The server's `package.json` has no `build` script—only `dev` and `updateConfigSchema`.

### Monorepo Linker Configuration

`bunfig.toml` specifies:
```toml
[install]
linker = "hoisted"
```

The `hoisted` linker means:
- Dependencies are installed to the root `node_modules/` 
- Workspaces see a flat `node_modules/` structure
- This allows Bun to find `@types/node`, TypeScript, and other shared devDeps
- `@common/*` path alias resolution is **independent of this**—it uses filesystem paths, not npm packages

---

## Summary: Compilation Strategy

| Aspect | Details |
|--------|---------|
| **Common package format** | Raw TypeScript source in `common/src/`; no compilation, no npm bundling |
| **Client compilation** | Vite + esbuild: processes common source as part of main build; **inlines into one bundle** |
| **Server compilation** | Bun native: JIT-compiles common source on-the-fly; **no bundling, separate modules** |
| **Path resolution** | Both use `tsconfig.json` path alias `"@common/*": ["../common/src/*"]` |
| **Separate builds?** | No. Each consumer (client, server) handles common's TypeScript independently based on its own compilation settings |
| **Package manager integration** | Not used. No npm dependency graph for common; direct filesystem path aliases instead |

---

## Visual Flow

### Client Build Flow

```
client/src/scripts/game.ts
    ↓ import @common/packets
    ↓ (Vite resolves to ../common/src/packets)
common/src/packets/index.ts (source TS)
    ↓ esbuild transpiles both together
    ↓
client/dist/scripts/main-[hash].js (minified, bundled)
```

### Server Runtime Flow

```
server/src/game.ts
    ↓ import @common/packets
    ↓ (Bun resolves to ../common/src/packets)
common/src/packets/index.ts (source TS)
    ↓ Bun JIT-compiles on-the-fly
    ↓
Memory (no intermediate files)
```

---

## Why This Approach?

1. **Single source of truth** — Changes to `common/src/` immediately affect both client and server without any build step
2. **Flexible compilation** — Client and server can compile with different settings (client needs browser-compatible code, server needs Node.js compatible code)
3. **No circular dependencies** — Path aliases prevent npm package confusion (common is not a package, just a folder)
4. **Fast iteration** — During dev, changes to common are picked up instantly (Vite HMR, Bun hot reload)
5. **Tree-shaking efficiency** — Each consumer (client, server) only includes common code it actually uses

---

## References

- **Tier 1:** `docs/architecture.md` — Monorepo structure and package layout
- **Tier 1:** `docs/development.md` — Build commands and setup
- **Source:** `package.json` — Bun workspace declaration
- **Source:** `bunfig.toml` — Linker configuration
- **Source:** `client/tsconfig.json` — Path alias definition
- **Source:** `server/tsconfig.json` — Path alias definition
- **Source:** `client/vite/vite.common.ts` — Vite path alias configuration
