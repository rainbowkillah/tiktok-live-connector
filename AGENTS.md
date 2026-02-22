# Repository Guidelines

## Project Structure & Module Organization
`src/index.ts` is the public package entrypoint. Core connection logic lives in `src/lib/client.ts`, with HTTP/signing flow in `src/lib/web/**` and WebSocket transport in `src/lib/ws/**`. Shared contracts are in `src/types/**`, including generated schema types (`src/types/tiktok-schema.ts`).  
Protobuf sources are in `.proto/src/*.proto`; regeneration scripts live at `.proto/build-node-compatible.ts` and `.proto/build-web-compatible.ts`.  
Build output is written to `dist/` and should not be edited by hand.

## Build, Test, and Development Commands
- `npm install` installs dependencies (Node `>=20` required).
- `npm run build` updates `src/version.ts`, copies `package.json`/`LICENSE` into `dist/`, compiles TypeScript, and resolves path aliases.
- `npx tsx .proto/build-node-compatible.ts` regenerates Node-oriented protobuf typings into `src/types/`.
- `npx tsx .proto/build-web-compatible.ts` regenerates browser-compatible schema output in `.proto/web_dist/`.
- `npm pack --dry-run` verifies publish artifacts before release.

There is currently no `npm test` script in this repository.

## Coding Style & Naming Conventions
Follow `.editorconfig`: 4 spaces for source files, 2 spaces for JSON. Formatting is governed by Prettier (`.prettierrc.json`): single quotes and `printWidth: 180`.  
Use TypeScript with explicit named exports. Prefer `PascalCase` for classes/types, `camelCase` for variables/functions, and `kebab-case` for route module filenames (for example, `fetch-room-info-euler.ts`).  
Use the `@/*` import alias from `tsconfig.json` for internal modules.

## Testing Guidelines
No automated suite is wired yet. At minimum, validate every change with `npm run build` and a manual smoke test of the affected connection path (WebSocket or polling).  
If adding tests, place them under `src/**/__tests__` or `tests/` and add a corresponding `package.json` script in the same PR.

## Commit & Pull Request Guidelines
Recent history uses Conventional Commit-style prefixes (`feat:`, `fix:`, `bump:`); keep using that pattern. Keep each commit focused on one concern.  
PRs should include a short problem/solution summary, changed paths (for example `src/lib/web/routes/*`), manual verification steps, and linked issues.  
When behavior or options change, update `README.md` in the same PR.
