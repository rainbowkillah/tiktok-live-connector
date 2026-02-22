# Gemini CLI Guidelines for TikTok-Live-Connector

## Project Overview

**TikTok-Live-Connector** is a Node.js library designed to receive live stream events (such as comments, gifts, likes, and member joins) in real-time from TikTok LIVE. It connects to TikTok's internal Webcast push service using just the broadcaster's username (`@uniqueId`).

### Architecture & Key Modules
- **Public API**: Exported from `src/index.ts`.
- **Connection Logic**: `TikTokLiveConnection` inside `src/lib/client.ts` orchestrates the connection, routing, and event emission.
- **Web Requests & Signing**: Located in `src/lib/web/**`. Handles HTTP requests, room ID resolution, and WebSocket URL signing via `EulerSigner`.
- **WebSocket Transport**: Located in `src/lib/ws/**`. Handles the WebSocket lifecycle, heartbeats, and frame deserialization.
- **Protobuf Schemas**: Raw files are in `.proto/src/*.proto`. Generated types reside in `src/types/tiktok-schema.ts`.
- **Events**: Defined in `src/types/events.ts` (`WebcastEvent`, `ControlEvent`).
- **Legacy Shim**: `src/lib/_legacy/legacy-client.ts` contains `WebcastPushConnection` for backwards compatibility. Do not remove this without a major version bump.

## Building and Running

### Prerequisites
- Node.js >= 20.0.0

### Key Commands
- **Install dependencies**: `npm install`
- **Build the project**: `npm run build` (This runs TypeScript compilation, updates `src/version.ts`, resolves path aliases, and copies package metadata to `dist/`).
- **Protobuf Regeneration**:
  - For Node-compatible schemas: `npx tsx .proto/build-node-compatible.ts`
  - For Browser-compatible schemas: `npx tsx .proto/build-web-compatible.ts`

### Testing
There is currently **no automated test suite** (`npm test` does not exist).
- **Validation**: All changes must be validated by running `npm run build` followed by a manual smoke test of the affected connection path (WebSocket or polling).
- **Adding Tests**: If adding tests, place them under `src/**/__tests__` or `tests/` and add a corresponding `test` script to `package.json`.

## Development Conventions

### Coding Style
- **Indentation**: 4 spaces for `.ts` source files, 2 spaces for `.json` files (per `.editorconfig`).
- **Formatting**: Governed by Prettier (`.prettierrc.json`) using single quotes and `printWidth: 180`.
- **Naming**:
  - `PascalCase` for classes and types.
  - `camelCase` for variables and functions.
  - `kebab-case` for route module filenames (e.g., `fetch-room-info-euler.ts`).
- **Imports**: Use the `@/*` import alias from `tsconfig.json` for internal modules. Use explicit named exports.

### Git & PR Guidelines
- **Commit Messages**: Follow Conventional Commits format (e.g., `feat:`, `fix:`, `bump:`). Keep each commit focused on a single concern.
- **Documentation**: Whenever changing public behaviors, events, or options, ensure `README.md` is updated in the same Pull Request.
