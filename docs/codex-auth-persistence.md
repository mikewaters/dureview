# Persisting ChatGPT (Codex) OAuth credentials

How dureview keeps a ChatGPT-subscription login alive across refreshes and restarts, written so another project can copy the approach.

All source links are permalinks to commit `3e5a748dc8902d73c4126277fdfc56b8bc13803f` of `mikewaters/dureview`.

> **Terms-of-use risk.** The Codex OAuth path signs in with a ChatGPT Plus/Pro subscription. OpenAI's terms may restrict automated use of subscription accounts. Decide that with your own policy owners before you adopt this. dureview's README says the same: [README.md#L7](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/README.md#L7). For an API key, nothing here applies. The key is read from the environment and never written to disk.

## Why it has to be persisted

The Codex access token is short-lived. When pi-ai refreshes it, the provider rotates the refresh token as well. The old refresh token can no longer be used, so the new one must be stored before the next request needs it. If the new token is lost, the only recovery is an interactive re-login.

That rules out an ephemeral worker per job, which cannot write the new token back. dureview is a long-running process with a data directory on a persistent volume ([README.md#L5](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/README.md#L5), [README.md#L95](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/README.md#L95)).

## What to build (four pieces)

| # | Piece | dureview source | Size |
| --- | --- | --- | --- |
| 1 | A `CredentialStore` backed by one JSON file, with a lock | [src/auth/store.ts#L17-L95](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/auth/store.ts#L17-L95) | about 150 lines, depends only on `node:fs` and pi-ai types |
| 2 | A pi-ai `Models` object wired to that store | [src/auth/session.ts#L23-L28](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/auth/session.ts#L23-L28) | 6 lines |
| 3 | A one-time `login` command | [src/auth/session.ts#L42-L64](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/auth/session.ts#L42-L64) and the terminal prompts at [#L72-L116](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/auth/session.ts#L72-L116) | about 75 lines |
| 4 | A persistent volume, plus a build that keeps pi-ai external | [Dockerfile#L21](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/Dockerfile#L21), [vite.config.ts#L21-L51](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/vite.config.ts#L21-L51) | config |

`store.ts` has no dureview-specific imports, so it can be copied as-is.

## How the pieces fit

```
dureview login ──▶ models.login("openai-codex","oauth", interaction)
                      │  provider runs the browser / device-code flow
                      └─▶ store.modify("openai-codex", async () => credential)   ──▶ auth.json

each model request ─▶ models.getAuth(...)
                      │  token has < 5 min left?  (checked without the lock first)
                      └─▶ store.modify("openai-codex", async (current) => {
                              re-check under the lock       // another request/process may have refreshed
                              return await oauth.refresh(current)   // 15 s timeout
                          })                                ──▶ auth.json (rotated refresh token)
```

The 5-minute window and the 15-second refresh timeout are constants in `dist/auth/resolve.js` of the pinned `@earendil-works/pi-ai@1.1.0` npm package. I could not link that file on GitHub, so check it in `node_modules` after install. The steps are double-checked locking. The cheap expiry check runs outside the lock. The authoritative check runs inside `modify`, so concurrent requests cannot refresh twice and burn the rotated token. If a refresh fails, pi-ai throws `ModelsError` with code `oauth` and leaves the stored credential unchanged. A store failure throws code `auth`.

## The store contract

pi-ai defines the interface, with the semantics in the doc comment on `CredentialStore` in its `auth/types.d.ts`.

| Method | Must do |
| --- | --- |
| `read(providerId)` | Return the credential (possibly expired) or `undefined` if there is none. Never throw for "missing". |
| `list()` | Return `{ providerId, type }` only. Never expose secrets. |
| `modify(providerId, fn)` | **The only write path.** Call `fn(current)` while holding mutual exclusion and persist its result. `undefined` from `fn` means "leave unchanged". Resolve with the stored credential. Let rejections from `fn` propagate. |
| `delete(providerId)` | Remove the entry, serialized against `modify`. |

dureview's implementation ([store.ts#L37-L53](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/auth/store.ts#L37-L53)) uses two layers of exclusion:

1. **An in-process promise chain** ([#L65-L72](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/auth/store.ts#L65-L72)). It serializes calls inside one process, even if an earlier call rejected.
2. **A lock file**, `auth.json.lock`, created with the `wx` flag ([#L101-L122](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/auth/store.ts#L101-L122)). It gives mutual exclusion across processes on the same host, so `dureview login` can run next to a stopped server. A lock older than 180 seconds is treated as stale and stolen ([#L124-L131](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/auth/store.ts#L124-L131)). That is comfortably longer than pi-ai's 15-second refresh timeout. Keep that ordering if you change either number.

Writes go to `auth.json.<pid>.tmp` with mode `0600` and are renamed over the target ([#L89-L94](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/auth/store.ts#L89-L94)). The directory is created with mode `0700`.

## Minimal wiring

```ts
import { join } from "node:path";
import { createModels } from "@earendil-works/pi-ai/models";
import { openaiCodexProvider } from "@earendil-works/pi-ai/providers/openai-codex";
import { FileCredentialStore } from "./auth/store.js";

const models = createModels({
  credentials: new FileCredentialStore(join(dataDir, "auth.json")),
});
models.setProvider(openaiCodexProvider());
```

This mirrors [session.ts#L23-L28](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/auth/session.ts#L23-L28). Pass this `models` object to whatever makes the model calls. In dureview that is the pi-durable harness ([service.ts#L94-L111](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/service.ts#L94-L111)). Refresh needs no other code.

Login is `models.login("openai-codex", "oauth", interaction, { agentName })`, and pi-ai persists the result through `store.modify` for you ([session.ts#L49-L54](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/auth/session.ts#L49-L54)). The `interaction` object supplies `prompt()` and `notify()`. Two details from dureview's version:

- With no TTY, it picks `device_code` automatically ([#L79-L85](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/auth/session.ts#L79-L85)). The browser flow listens for a callback on `localhost:1455` (or lets you paste the redirect URL when that port is taken), which is awkward inside a container. `DUREVIEW_LOGIN_METHOD=device_code|browser` forces a choice.
- Login is capped at 15 minutes with an `AbortController` ([#L46-L47](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/auth/session.ts#L46-L47)).

## Choosing the provider at startup

`models.checkAuth(providerId)` reports whether complete auth exists **without refreshing**. dureview uses it to prefer Codex when an OAuth credential is stored, then the API key, then Codex as the default that `login` fills in ([session.ts#L34-L40](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/auth/session.ts#L34-L40)). At boot it only warns when no credential exists ([runtime.ts#L65-L70](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/app/runtime.ts#L65-L70)), so the service can start and be logged into afterwards.

For a status display, dureview reads `auth.json` directly and copies only "present" and "expires at", never the tokens ([src/ui/status.ts#L49-L71](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/ui/status.ts#L49-L71)).

## Deployment requirements

- **Persistent volume** for the data directory. Never bake `auth.json` into an image ([Dockerfile#L21](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/Dockerfile#L21), [docker-compose.yml#L6-L7](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/docker-compose.yml#L6-L7)).
- **One writer.** Two replicas with separate copies of the file will each rotate the token and invalidate the other. The file lock only protects processes sharing one filesystem. It is not a distributed lock. Treat this as the same single-process rule dureview states for its session storage ([README.md#L65](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/README.md#L65)).
- **Keep pi-ai external in the server bundle.** pi-ai loads the Codex OAuth module through a variable dynamic import so bundlers cannot follow it. If a bundler inlines pi-ai, the chunk is missing at runtime. dureview marks the packages external and traces them ([vite.config.ts#L21-L51](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/vite.config.ts#L21-L51)). It also runs a build-time smoke check that loads the provider and exercises `toAuth` from the built output ([scripts/smoke-pi-ai-external.mjs#L24-L71](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/scripts/smoke-pi-ai-external.mjs#L24-L71)), so a bundling regression fails the Docker build instead of the first review. Copy that check if you use a bundler.
- **Container login.** Run it once against the same volume, for example `docker run --rm -it -v dureview-data:/data dureview login` ([README.md#L18](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/README.md#L18)).

## Test to copy

[test/auth-store.test.ts#L8-L36](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/test/auth-store.test.ts#L8-L36) fires five concurrent `modify` calls that each read the current value, wait, and write `current + 1`. It then reopens the store from disk and asserts the result is `5`. A store without real mutual exclusion would lose updates and fail it. It also checks that `list()` returns metadata only.

## Failure modes

| Symptom | Cause | Behavior |
| --- | --- | --- |
| `ModelsError` code `oauth` | refresh rejected (revoked or already-used refresh token) | stored credential is kept. Fix is a re-login |
| `ModelsError` code `auth` | store threw: unreadable file, `Invalid credential file`, or a lock timeout after 180 s | the request fails. The file is not modified |
| Lost login after a host crash | the write is not `fsync`ed before the rename | a power loss at the wrong moment can leave the old or an empty file. This is read from the code, not tested. Add `fsync` on the temp file and its directory if that matters |
| Stale `auth.json.lock` | process killed while holding the lock | stolen after 180 s. Until then calls wait |

## Not verified

I did not run a live Codex login or a refresh. The constants and flow above are read from the pinned pi-ai 1.1.0 package and dureview's source. The README says the same: "Live Codex OAuth and a real GitHub App are not exercised here."
