# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm install          # Install dependencies (Node >=20 required)
npm run build        # Full build: updates version, copies package.json/LICENSE to dist/, compiles TypeScript, resolves path aliases
npm pack --dry-run   # Verify publish artifacts before release
```

**Protobuf regeneration** (when `.proto/src/*.proto` files change):
```bash
npx tsx .proto/build-node-compatible.ts   # Regenerate src/types/tiktok-schema.ts (Node-oriented)
npx tsx .proto/build-web-compatible.ts    # Regenerate .proto/web_dist/ (browser-compatible stubs)
```

There is no `npm test` script. Validate changes with `npm run build` and a manual smoke test of the affected connection path.

## Architecture

This is a TypeScript library that connects to TikTok's internal Webcast push service to receive live stream events. The public entry point is `src/index.ts`, which re-exports from `src/lib/`, `src/types/`, and `src/version.ts`.

### Connection Flow

`TikTokLiveConnection` (`src/lib/client.ts`) orchestrates the full lifecycle:

1. **Room ID resolution** — tries three methods in order: HTML scrape → TikTok API → Euler fallback (unless `disableEulerFallbacks: true`)
2. **Signing** — calls `EulerSigner` (wrapping `@eulerstream/euler-api-sdk`) to obtain a signed WebSocket URL via the Euler Stream service
3. **WebSocket** — `TikTokWsClient` (`src/lib/ws/lib/ws-client.ts`) extends the `ws` WebSocket, sends `im_enter_room` on connect, maintains a heartbeat ping, and decodes incoming protobuf frames
4. **Decoding** — `deserializeWebSocketMessage` in `src/lib/utilities.ts` unwraps `WebcastPushFrame` → decompresses gzip → decodes `ProtoMessageFetchResult` → decodes each nested message against the schema in `src/types/tiktok-schema.ts`
5. **Event emission** — `processDecodedData` in `client.ts` maps decoded proto type names to `WebcastEvent`/`ControlEvent` enum values (see `WebcastEventMap` in `src/types/events.ts`) and emits them on the `TikTokLiveConnection` instance

### Key Modules

| Path | Role |
|------|------|
| `src/lib/client.ts` | Main `TikTokLiveConnection` class: connect/disconnect/fetchRoomId/sendMessage logic |
| `src/lib/web/index.ts` | `TikTokWebClient` — assembles all route instances on top of `WebcastHttpClient` |
| `src/lib/web/lib/http-client.ts` | `WebcastHttpClient` — axios wrapper with optional URL signing, cookie jar, JSON/protobuf/HTML fetchers |
| `src/lib/web/lib/tiktok-signer.ts` | `EulerSigner` — thin wrapper around `EulerStreamApiClient` that strips forbidden params and calls `signWebcastUrl` |
| `src/lib/web/lib/cookie-jar.ts` | Manages `sessionid` + `tt-target-idc` cookies for authenticated requests |
| `src/lib/web/routes/` | Callable route classes (extend `Route<Args, Response>` from `src/types/route.ts`); filenames use `kebab-case` |
| `src/lib/ws/lib/ws-client.ts` | `TikTokWsClient` — WebSocket with heartbeat, ACK, room-switch, and frame deserialization |
| `src/lib/utilities.ts` | Protobuf serialization/deserialization helpers, `validateAndNormalizeUniqueId`, `createBaseWebcastPushFrame` |
| `src/lib/config.ts` | `Config` (HTTP/WS defaults), `SignConfig` (Euler API base path/key), device/location/screen presets |
| `src/types/tiktok-schema.ts` | Generated protobuf types — do not edit by hand; regenerate via `.proto/` scripts |
| `src/types/events.ts` | `ControlEvent`, `WebcastEvent` enums; `WebcastEventMap` (proto type name → event); `ClientEventMap` typed event signatures |
| `src/lib/_legacy/` | `WebcastPushConnection` — deprecated backwards-compat shim over `TikTokLiveConnection` |

### Route Pattern

All HTTP routes are callable class instances (via `callable-instance`). Each extends `Route<Args, Response>` and implements `call(args)`. They are instantiated inside `TikTokWebClient`'s constructor and attached as public properties (e.g. `webClient.fetchRoomInfoFromHtml(...)`).

### Signing & Euler Stream

`SignConfig` in `src/lib/config.ts` is a module-level singleton (`Partial<ClientConfiguration>`) that users can mutate directly to set an API key or custom base path. The `EulerSigner` merges `SignConfig` with any per-instance config passed to it. To bypass Euler entirely, set `options.signedWebSocketProvider` to a custom async function.

### Environment Variables

| Variable | Effect |
|----------|--------|
| `SIGN_API_KEY` | Sets the default Euler API key (same as `SignConfig.apiKey`) |
| `SIGN_API_URL` | Overrides Euler base URL (default: `https://tiktok.eulerstream.com`) |
| `TIKTOK_CLIENT_TIMEOUT` | axios timeout in ms (default: `10000`) |
| `RANDOMIZE_TIKTOK_DEVICE` | `"true"` picks a random user-agent from the preset list |
| `RANDOMIZE_TIKTOK_LOCATION` | `"true"` picks a random locale preset |
| `RANDOMIZE_TIKTOK_SCREEN` | `"true"` picks a random screen resolution preset |
| `DEBUG_DESERIALIZE_XD` | Logs base64 payloads for unknown proto message types |

## Code Style

- 4 spaces for `.ts` source files, 2 spaces for `.json` (see `.editorconfig`)
- Prettier: single quotes, `printWidth: 180` (see `.prettierrc.json`)
- Use the `@/*` alias (maps to `./src/*`) for all internal imports
- Route module filenames: `kebab-case` (e.g., `fetch-room-info-euler.ts`)
- Classes/types: `PascalCase`; variables/functions: `camelCase`
- Commits use Conventional Commit prefixes: `feat:`, `fix:`, `bump:`
- When changing public behavior or options, update `README.md` in the same PR

## Legacy Compatibility

`WebcastPushConnection` in `src/lib/_legacy/legacy-client.ts` is a deprecated subclass of `TikTokLiveConnection` that emits simplified/flattened event payloads (via `simplifyObject` in `data-converter.ts`). New code should use `TikTokLiveConnection` directly. Do not remove the legacy class without a major version bump.
