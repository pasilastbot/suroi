# Top 50 Questions a New Senior Developer Might Ask

> Onboarding reference for new team members joining the suroi project.
> Questions are organized by angle/concern area.

---

## Architecture & System Design

1. Why was Bun chosen over Node.js, and are there places where Bun-specific APIs are used that would make migration difficult?
2. How does the GameManager → Worker architecture work, and why is each game forked as a separate worker instead of running in-process?
3. What is the rationale behind the monorepo workspace layout (`client/`, `common/`, `server/`, `tests/`), and what are the dependency rules between packages?
4. Why does the project use a custom binary protocol over WebSocket instead of JSON or Protocol Buffers?
5. How is the `common/` package shared between client and server — is it compiled separately for each, or bundled differently?
6. What drove the decision to use PixiJS 8 for rendering instead of Canvas2D, Phaser, or a WebGL abstraction?
7. Where does the boundary between server-authoritative logic and client-side prediction sit, and is there any client-side prediction at all?

## Networking & Multiplayer

8. How large is a typical `UpdatePacket` in bytes, and what is the downstream bandwidth per player at 40 TPS?
9. What happens when a packet is lost or arrives out of order — is there any compensation or does WebSocket TCP handle it?
10. How does the dirty flag system (`partialDirtyObjects` / `fullDirtyObjects`) decide what to include in each tick's packet?
11. Why is visibility recalculated only every ~8 ticks instead of every tick, and what are the trade-offs?
12. How does the server handle a player disconnecting mid-game — is there a grace period or instant death?
13. What is the `PacketStream` multiplexing model, and can multiple packet types be batched into a single WebSocket frame?

## Game Loop & Performance

14. The server runs at 40 TPS with a 25ms tick budget — what is the typical tick time, and where does most CPU time go?
15. How does the spatial grid (32-unit cells) scale as player count increases, and has the cell size been tuned?
16. What profiling tools or metrics exist for measuring server tick performance in production?
17. Are there any known hot paths that could become bottlenecks with higher player counts (e.g., explosion raycasting)?
18. How is the game loop scheduled — `setInterval`, `setTimeout`, or a Bun-specific timer mechanism?

## Object System & Definitions

19. How does the `ObjectDefinitions<T>` registry work, and what is the contract for adding a new definition type?
20. Why are definitions serialized as single-byte indices — what happens when a category exceeds 255 items?
21. What is the `BaseGameObject.derive(ObjectCategory.X)` pattern, and why was mixins chosen over class inheritance?
22. How do I add a completely new weapon type that doesn't fit existing categories (e.g., a deployable turret)?

## Client Rendering & UI

23. How does the spritesheet packing pipeline work (4096×4096 hi-res / 2048×2048 lo-res), and what happens when assets overflow a single sheet?
24. What is the PixiJS container hierarchy, and how do the 3 layer containers map to the 5 logical game layers?
25. How does the layer transition animation (200ms sineOut) work when a player walks onto stairs?
26. Why is the UI built with a mix of raw DOM/jQuery and Svelte — is there a migration plan to unify on Svelte?
27. How does the minimap rendering work — is it a separate canvas, an off-screen PixiJS render, or DOM overlay?

## Combat & Gameplay Mechanics

28. How is bullet damage calculated end-to-end — from gun fire through armor reduction to final health change?
29. What is the actual armor reduction formula, and why is it additive (helmet + vest) instead of multiplicative?
30. How does the explosion raycasting work — how many rays are cast, and how does wall occlusion affect damage?
31. What determines kill credit in team modes when a downed player is finished off by a different attacker?
32. How do perks interact with each other — what is the modifier application order and are there known stacking bugs?

## Map & World Generation

33. How does the map generation algorithm ensure buildings don't overlap, and what is the retry/fallback strategy?
34. What is the loot table structure, and how are rarity weights balanced across different map regions?
35. How are rivers and trails generated — spline-based, noise-based, or predefined paths?
36. What is the gas phase progression, and how are the center position and radius interpolated between stages?

## Testing & Quality

37. What is the current test coverage, and which areas of the codebase have the most/least coverage?
38. Why is there both Jest (`tests/`) and Bun test infrastructure — which should I use for new tests?
39. What does `validateDefinitions` check, and how do I run it before submitting a PR?
40. Are there integration or end-to-end tests for the full client-server game loop, or only unit tests?
41. How do I set up a local development environment to run both client and server simultaneously?

## Build, Deploy & DevOps

42. What does `bun dev` actually start — does it run both client dev server and game server, or just one?
43. How does the Vite build pipeline work with the custom plugins (spritesheet packer, audio spritesheet, HJSON translations)?
44. What is the deployment architecture — how many game server instances run in production, and how are players routed?
45. How are configuration changes deployed — does the server need a restart, or can config be hot-reloaded?

## Code Style & Contribution

46. What are the Biome lint rules that most frequently trip up new contributors (e.g., no `!` assertions, `interface` over `type`)?
47. What is the PR review process — are there required reviewers, CI checks, or automated gates?
48. How is the translation/localization system maintained — who updates the HJSON files, and what is the fallback chain?
49. Is there an established pattern for feature flags, or are experimental features gated by game mode configuration?
50. What is the plugin system's intended use — are plugins meant for modding by external developers, or only internal server extensions?
