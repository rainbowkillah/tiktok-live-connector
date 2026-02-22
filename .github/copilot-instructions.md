# Copilot Instructions

## Build Commands

```bash
npm install          # Node >=20 required
npm run build        # Updates src/version.ts, copies package.json/LICENSE to dist/, compiles TS, resolves path aliases
npm pack --dry-run   # Verify publish artifacts
```

There is no `npm test` script. Validate changes with `npm run build` and a manual smoke test of the affected path.

**Protobuf regeneration** (only when `.proto/src/*.proto` files change):
```bash
npx tsx .proto/build-node-compatible.ts   # Regenerates src/types/tiktok-schema.ts — do not edit by hand
npx tsx .proto/build-web-compatible.ts    # Regenerates .proto/web_dist/
```

## Architecture

`TikTokLiveConnection` (`src/lib/client.ts`) is the main class. It orchestrates:

1. **Room ID resolution** — three methods tried in order: HTML scrape → TikTok API → Euler fallback. Set `disableEulerFallbacks: true` to skip the third.
2. **Signing** — `EulerSigner` (wraps `@eulerstream/euler-api-sdk`) obtains a signed WebSocket URL. Override by passing `signedWebSocketProvider` in options.
3. **WebSocket** — `TikTokWsClient` (`src/lib/ws/lib/ws-client.ts`) sends `im_enter_room` on connect, maintains heartbeat, and decodes protobuf frames.
4. **Decoding pipeline** — `deserializeWebSocketMessage` (in `utilities.ts`): `WebcastPushFrame` → gunzip → `ProtoMessageFetchResult` → decode each nested message via schema in `tiktok-schema.ts`.
5. **Event emission** — `processDecodedData` maps proto type names to `WebcastEvent`/`ControlEvent` values using `WebcastEventMap` (`src/types/events.ts`) and emits on the connection instance.

### Key modules

| Path | Role |
|---|---|
| `src/lib/client.ts` | `TikTokLiveConnection` — connect/disconnect/fetchRoomId/sendMessage |
| `src/lib/web/index.ts` | `TikTokWebClient` — assembles all route instances on top of `WebcastHttpClient` |
| `src/lib/web/lib/http-client.ts` | `WebcastHttpClient` — axios wrapper with signing, cookie jar, JSON/protobuf/HTML fetchers |
| `src/lib/web/lib/tiktok-signer.ts` | `EulerSigner` — strips forbidden params, calls `signWebcastUrl` |
| `src/lib/web/lib/cookie-jar.ts` | Manages `sessionid` + `tt-target-idc` cookies |
| `src/lib/web/routes/` | HTTP route classes; filenames use `kebab-case` |
| `src/lib/ws/lib/ws-client.ts` | WebSocket with heartbeat, ACK, room-switch, frame deserialization |
| `src/lib/utilities.ts` | Protobuf serialization helpers, `validateAndNormalizeUniqueId` |
| `src/lib/config.ts` | `Config` (HTTP/WS defaults), `SignConfig` (module-level Euler API config), device/location/screen presets |
| `src/types/tiktok-schema.ts` | **Generated** — do not edit directly |
| `src/types/events.ts` | `ControlEvent`, `WebcastEvent` enums; `WebcastEventMap`; typed `ClientEventMap` |
| `src/lib/_legacy/` | `WebcastPushConnection` — deprecated shim; do not remove without a major version bump |

### Signing & Euler Stream

`SignConfig` in `src/lib/config.ts` is a module-level singleton that consumers mutate to set an API key or custom base path. `EulerSigner` merges `SignConfig` with any per-instance config. Environment variables `SIGN_API_KEY` and `SIGN_API_URL` set these at startup.

## Key Conventions

### Route pattern
All HTTP routes are callable class instances using `callable-instance`. Each extends `Route<Args, Response>` from `src/types/route.ts` and implements `call(args)`. They are instantiated in `TikTokWebClient`'s constructor and attached as public properties.

### Adding a new event
1. Add a proto message type to `.proto/src/*.proto` and regenerate `tiktok-schema.ts`.
2. Add a `WebcastEvent` enum value in `src/types/events.ts`.
3. Add the proto-type-name → event mapping to `WebcastEventMap`.
4. Add the typed handler signature to `ClientEventMap`.
5. Handle any special dispatch logic in `processDecodedData` in `client.ts` (only needed if the event requires branching on message fields, like `FOLLOW`/`SHARE`).

### Import alias
Use `@/*` (maps to `./src/*`) for all internal imports — never use relative paths crossing module boundaries.

### Code style
- 4 spaces for `.ts`, 2 spaces for `.json` (`.editorconfig`)
- Prettier: single quotes, `printWidth: 180` (`.prettierrc.json`)
- `PascalCase` classes/types, `camelCase` variables/functions, `kebab-case` route filenames

### Commits
Use Conventional Commit prefixes: `feat:`, `fix:`, `bump:`. Update `README.md` in the same PR when public behavior or options change.

## Environment Variables

| Variable | Effect |
|---|---|
| `SIGN_API_KEY` | Euler API key (same as `SignConfig.apiKey`) |
| `SIGN_API_URL` | Euler base URL override (default: `https://tiktok.eulerstream.com`) |
| `TIKTOK_CLIENT_TIMEOUT` | axios timeout in ms (default: `10000`) |
| `RANDOMIZE_TIKTOK_DEVICE` | `"true"` picks a random user-agent preset |
| `RANDOMIZE_TIKTOK_LOCATION` | `"true"` picks a random locale preset |
| `RANDOMIZE_TIKTOK_SCREEN` | `"true"` picks a random screen resolution preset |
| `DEBUG_DESERIALIZE_XD` | Logs base64 payloads for unknown proto message types |
