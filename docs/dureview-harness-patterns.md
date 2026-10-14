# dureview harness patterns

Three patterns from dureview that another project needs to reproduce, each written so a team can implement it from this page alone. All source links are permalinks to commit `3e5a748dc8902d73c4126277fdfc56b8bc13803f` of `mikewaters/dureview`.

## At a glance

| Part | What to build | Biggest thing to get right | Main risk |
| --- | --- | --- | --- |
| [1. Codex auth persistence](#part-1-persisting-chatgpt-codex-oauth-credentials) | A file-backed credential store with a lock, wired into pi-ai, plus a one-time login command and a persistent volume | Every refresh rewrites the stored token, because the refresh token rotates | A lost write means an interactive re-login. Two writers invalidate each other |
| [2. Eager recovery](#part-2-eager-recovery-of-interrupted-durable-work) | A boot trigger that opens the harness and calls `resume()` before any request | Install extensions before `resume()`, and make every task step safe to re-run | Running two processes on one storage directory |
| [3. Core tool lockdown](#part-3-disabling-the-core-coding-tools-for-safety) | Do not install the coding tools, and give each conversation an explicit tool allowlist | Default-deny: the model only has tools you wrote | Nothing tests that the dangerous tools are absent |

## Before you adopt this

- **Terms of use.** Part 1 signs in with a ChatGPT Plus/Pro subscription. OpenAI's terms may restrict automated use of subscription accounts. Settle that with your policy owners first. With an API key, Part 1 does not apply.
- **Nothing here was run live.** No live Codex login, token refresh, rolling restart, or tool-abuse attempt was performed. Statements come from dureview's source, its tests, and the pinned 1.1.0 library packages. Each part ends with what is unverified.
- **Pi Durable is experimental.** Its API can change between releases. dureview pins `@earendil-works/pi-durable`, `pi-ai` and `chord` at 1.1.0.
- **One process per storage directory.** This applies to the session database, the credential file, and the recovery design.

## Part 1: Persisting ChatGPT (Codex) OAuth credentials

How dureview keeps a ChatGPT-subscription login alive across refreshes and restarts, written so another project can copy the approach.


> **Terms-of-use risk.** The Codex OAuth path signs in with a ChatGPT Plus/Pro subscription. OpenAI's terms may restrict automated use of subscription accounts. Decide that with your own policy owners before you adopt this. dureview's README says the same: [README.md#L7](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/README.md#L7). For an API key, nothing here applies. The key is read from the environment and never written to disk.

### Why it has to be persisted

The Codex access token is short-lived. When pi-ai refreshes it, the provider rotates the refresh token as well. The old refresh token can no longer be used, so the new one must be stored before the next request needs it. If the new token is lost, the only recovery is an interactive re-login.

That rules out an ephemeral worker per job, which cannot write the new token back. dureview is a long-running process with a data directory on a persistent volume ([README.md#L5](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/README.md#L5), [README.md#L95](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/README.md#L95)).

### What to build (four pieces)

| # | Piece | dureview source | Size |
| --- | --- | --- | --- |
| 1 | A `CredentialStore` backed by one JSON file, with a lock | [src/auth/store.ts#L17-L95](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/auth/store.ts#L17-L95) | about 150 lines, depends only on `node:fs` and pi-ai types |
| 2 | A pi-ai `Models` object wired to that store | [src/auth/session.ts#L23-L28](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/auth/session.ts#L23-L28) | 6 lines |
| 3 | A one-time `login` command | [src/auth/session.ts#L42-L64](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/auth/session.ts#L42-L64) and the terminal prompts at [#L72-L116](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/auth/session.ts#L72-L116) | about 75 lines |
| 4 | A persistent volume, plus a build that keeps pi-ai external | [Dockerfile#L21](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/Dockerfile#L21), [vite.config.ts#L21-L51](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/vite.config.ts#L21-L51) | config |

`store.ts` has no dureview-specific imports, so it can be copied as-is.

### How the pieces fit

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

### The store contract

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

### Minimal wiring

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

### Choosing the provider at startup

`models.checkAuth(providerId)` reports whether complete auth exists **without refreshing**. dureview uses it to prefer Codex when an OAuth credential is stored, then the API key, then Codex as the default that `login` fills in ([session.ts#L34-L40](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/auth/session.ts#L34-L40)). At boot it only warns when no credential exists ([runtime.ts#L65-L70](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/app/runtime.ts#L65-L70)), so the service can start and be logged into afterwards.

For a status display, dureview reads `auth.json` directly and copies only "present" and "expires at", never the tokens ([src/ui/status.ts#L49-L71](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/ui/status.ts#L49-L71)).

### Deployment requirements

- **Persistent volume** for the data directory. Never bake `auth.json` into an image ([Dockerfile#L21](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/Dockerfile#L21), [docker-compose.yml#L6-L7](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/docker-compose.yml#L6-L7)).
- **One writer.** Two replicas with separate copies of the file will each rotate the token and invalidate the other. The file lock only protects processes sharing one filesystem. It is not a distributed lock. Treat this as the same single-process rule dureview states for its session storage ([README.md#L65](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/README.md#L65)).
- **Keep pi-ai external in the server bundle.** pi-ai loads the Codex OAuth module through a variable dynamic import so bundlers cannot follow it. If a bundler inlines pi-ai, the chunk is missing at runtime. dureview marks the packages external and traces them ([vite.config.ts#L21-L51](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/vite.config.ts#L21-L51)). It also runs a build-time smoke check that loads the provider and exercises `toAuth` from the built output ([scripts/smoke-pi-ai-external.mjs#L24-L71](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/scripts/smoke-pi-ai-external.mjs#L24-L71)), so a bundling regression fails the Docker build instead of the first review. Copy that check if you use a bundler.
- **Container login.** Run it once against the same volume, for example `docker run --rm -it -v dureview-data:/data dureview login` ([README.md#L18](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/README.md#L18)).

### Test to copy

[test/auth-store.test.ts#L8-L36](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/test/auth-store.test.ts#L8-L36) fires five concurrent `modify` calls that each read the current value, wait, and write `current + 1`. It then reopens the store from disk and asserts the result is `5`. A store without real mutual exclusion would lose updates and fail it. It also checks that `list()` returns metadata only.

### Failure modes

| Symptom | Cause | Behavior |
| --- | --- | --- |
| `ModelsError` code `oauth` | refresh rejected (revoked or already-used refresh token) | stored credential is kept. Fix is a re-login |
| `ModelsError` code `auth` | store threw: unreadable file, `Invalid credential file`, or a lock timeout after 180 s | the request fails. The file is not modified |
| Lost login after a host crash | the write is not `fsync`ed before the rename | a power loss at the wrong moment can leave the old or an empty file. This is read from the code, not tested. Add `fsync` on the temp file and its directory if that matters |
| Stale `auth.json.lock` | process killed while holding the lock | stolen after 180 s. Until then calls wait |

### Not verified

I did not run a live Codex login or a refresh. The constants and flow above are read from the pinned pi-ai 1.1.0 package and dureview's source. The README says the same: "Live Codex OAuth and a real GitHub App are not exercised here."

## Part 2: Eager recovery of interrupted durable work

How dureview resumes reviews that a previous process left unfinished, at process start and not on the first request. Written so another project can copy the approach.


### What "eager" means here

pi-durable stores task checkpoints in SQLite, but nothing runs them until the scheduler is enabled. The library enables it lazily: `submit()`, `wait()` and similar calls start it, and so does `harness.resume()` ([`Harness.resume`](https://github.com/earendil-works/pi/blob/main/packages/durable/README.md#persist-and-resume) in the package README). If the app only opened the harness when the first request arrived, a review interrupted by a deploy would sit idle until some webhook or page view happened to trigger it.

dureview avoids that in three steps:

1. **At module load**, the server entry asks for the runtime in a microtask. No request is needed.
2. **While opening the runtime**, the service installs its extension, opens storage, and calls `harness.resume()` explicitly.
3. **Every phase of the task is written to be re-entered** after a crash, so resuming is safe.

### Step 1: the boot trigger

[src/server-entry.ts#L13-L21](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/server-entry.ts#L13-L21):

```ts
const building = process.argv.includes("build") || process.env.npm_lifecycle_event === "build";
if (!building && !process.env.VITEST) {
  queueMicrotask(() => {
    void getRuntime().catch((error: unknown) => {
      console.error(error instanceof Error ? error.message : error);
      process.exit(1);
    });
  });
}
```

- It runs when the Nitro server module is evaluated. In production, `node dist/cli.js serve` imports the built server ([src/prod.ts#L19](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/prod.ts#L19)), which evaluates this file.
- It is skipped during `build` and under Vitest, so those do not open the database or start work.
- **It fails fast.** If the runtime cannot open (bad config, unreadable storage), the process exits with code 1 and the orchestrator restarts or alerts. Without this, the process would stay up and answer requests with errors.
- Requests also go through `await getRuntime()` ([server-entry.ts#L23-L26](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/server-entry.ts#L23-L26)), so a request that races the boot waits for it instead of starting a second open.
- `/health` also awaits the runtime ([src/routes/health.ts#L7-L8](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/routes/health.ts#L7-L8)). The Docker `HEALTHCHECK` stays red until the runtime is open, and it has a 20-second start period ([Dockerfile#L24-L25](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/Dockerfile#L24-L25)).

### Step 2: a singleton runtime that resumes while opening

[src/app/runtime.ts#L26-L30](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/app/runtime.ts#L26-L30) memoizes the open as one shared promise (`current`, `opening`). The open itself ([runtime.ts#L54-L110](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/app/runtime.ts#L54-L110)) builds config, models, the GitHub client and the checkout, then calls `DureviewService.open`.

The order inside [service.ts#L57-L120](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/service.ts#L57-L120) matters:

| Order | Call | Why |
| --- | --- | --- |
| 1 | `registry.install(extension)` ([#L90-L91](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/service.ts#L90-L91)) | The registry is not stored. Tasks are resolved by name, so the code must be installed before the scheduler can run a stored task. |
| 2 | `Harness.open(storage, {...})` ([#L94-L111](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/service.ts#L94-L111)) | Opens `session.sqlite`. |
| 3 | `harness.root(...)` ([#L117](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/service.ts#L117)) | Returns the same root conversation as last time. |
| 4 | `harness.resume()` ([#L118](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/service.ts#L118)) | Enables the scheduler, which picks up tasks left `running` or `pending`. |

Two details about timing:

- After `service.open` returns, the app also sweeps old checkouts and arms a timer, but only once the harness is already resuming ([runtime.ts#L99-L100](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/app/runtime.ts#L99-L100)). The sweep protects checkouts of reviews still marked running.
- Resumed work starts concurrently with HTTP startup. Anything a resumed task needs (GitHub client, models, credentials, settings) must be ready before `resume()`. dureview builds all of them before calling `DureviewService.open`. Credential trouble does not block boot, only the affected review ([runtime.ts#L65-L70](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/app/runtime.ts#L65-L70)). See the [Codex auth guide](#part-1-persisting-chatgpt-codex-oauth-credentials).

### Step 3: tasks that are safe to resume

Eager resume only helps if re-running a half-finished task is safe. dureview does four things.

**Checkpoint after every step.** Each phase handler does one step, commits a new checkpoint, and returns ([src/review/task.ts#L124-L464](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/task.ts#L124-L464)). The code relies on the scheduler calling the handler again with the committed checkpoint.

**Idempotency key for every outside effect.** The harness cannot undo a GitHub write, so each write has its own key: a check run matched by `external_id`, comments matched by hidden markers read before writing, and prompts submitted with a fixed `requestId`. The table is in [FINDINGS.md](https://github.com/mikewaters/dureview/blob/claude/architecture-findings/FINDINGS.md#crash-safety-comes-from-idempotency-keys-not-from-the-harness-alone).

**Record the result before the final step.** The `post` phase commits `published` and `checkOutput` before it completes the check run ([task.ts#L349-L381](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/task.ts#L349-L381)). A crash between those steps re-enters `post`, skips publishing, and only finishes the check.

**Tell shutdown apart from failure.** On SIGINT or SIGTERM the harness aborts the running invocation's signal. `failOrAbort` checks it and rethrows instead of writing a failed outcome ([task.ts#L623-L659](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/task.ts#L623-L659)), and each phase's `catch` does the same ([e.g. #L236](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/task.ts#L236)). Without this, every deploy would permanently fail the reviews it interrupted. The shutdown handlers are installed once ([runtime.ts#L140-L148](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/app/runtime.ts#L140-L148)) and call `closeRuntime`, which closes the harness, settings and archive.

### Checklist for another project

1. Define durable tasks with a checkpointed state machine. Give every external side effect an idempotency key, and make "already done" a normal path.
2. Install every extension before `Harness.open`, then call `resume()` explicitly. Do not rely on a later `submit()` or `wait()` to start the scheduler.
3. Trigger runtime creation at process start, not from the first request. Memoize the promise so that boot and requests share one open.
4. Exit non-zero if the boot open fails.
5. Wire `/health` to the same promise so the orchestrator sees "not ready" until recovery has started.
6. Handle SIGINT and SIGTERM by closing the harness. In task code, rethrow when `context.abortSignal?.aborted` instead of recording failure.
7. Run exactly one process per storage directory. The library has no cross-process lock ([README.md#L65](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/README.md#L65)), and a rolling deploy with overlap would run the same tasks twice.
8. Guard the boot trigger so builds and tests do not start the runtime.

### How to test recovery

[test/resume.test.ts](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/test/resume.test.ts) is the pattern. Open a service on a temp data directory, install a test hook that waits until the abort signal fires (`hangUntilAbort` in [test/support.ts#L492-L499](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/test/support.ts#L492-L499)), call `close()`, then open a second service on the same directory and `wait()` for the task. Assert the number of external writes. It covers four windows:

- before any write ([#L25](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/test/resume.test.ts#L25));
- after the write was accepted but before it was checkpointed ([#L69](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/test/resume.test.ts#L69));
- during checkout ([#L106](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/test/resume.test.ts#L106));
- the inline-comment mode ([#L141](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/test/resume.test.ts#L141)).

The first test also reopens a third time and checks that nothing re-posts.

### Pitfalls seen in the code

- **A failed open is cached.** `getRuntime` keeps `opening` after a rejection ([runtime.ts#L26-L30](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/app/runtime.ts#L26-L30)), so a later request gets the same error. This is acceptable only because the boot trigger exits the process. If your project retries instead of exiting, clear the promise on failure.
- **Shutdown is not time-bounded.** `closeRuntime().finally(() => process.exit(0))` waits as long as `harness.close` takes, because it runs under a context that cannot be cancelled. The orchestrator's SIGKILL is the backstop, and that looks like a crash, which recovery handles. How long `close` takes with a review mid-flight is not verified.
- **A replayed `post` repeats the completion callback.** The archive write is an idempotent upsert, but it overwrites `finished_at`, so durations after a crash include the downtime.
- **Changing the checkpoint shape strands in-flight tasks.** The review task is `version: 1` with no `migrate`, and the library reports tasks whose definition cannot take them as blocked. Add `migrate` before changing `ReviewState`.
- **Dev mode.** Whether `vite dev` evaluates the server entry at startup, and so boots the runtime without a request, is not verified.

## Part 3: Disabling the core coding tools for safety

How dureview guarantees that the model cannot run commands or change files, written so another project can copy the approach.


### What the core tools are

pi-durable ships `read`, `write`, `edit` and `bash` as the `CodingTools` extension, and `createPowerShellTool()` adds `powershell`. The package's own type comment says it all: "`read`, `write`, `edit`, and `bash`; **nothing installs it automatically**" (`dist/tools/index.d.ts` in `@earendil-works/pi-durable@1.1.0`).

That is the basis of the whole approach. The model can only call tools that some installed extension provides. The package README describes tools as coming only from extensions, and I found no built-in tool outside them. Not installing `CodingTools` is therefore enough to make `bash`, `write` and `edit` nonexistent for the model. They are not hidden or blocked, they are just not there.

### What dureview does

The model gets three tools and no others.

| Layer | What | Source |
| --- | --- | --- |
| 1. Never install the core tools | Nothing in `src/` imports `@earendil-works/pi-durable/tools` | whole tree |
| 2. One extension in the registry | `registry.install(extension)` is the only install | [service.ts#L90-L91](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/service.ts#L90-L91) |
| 3. Default selection is that extension only | `settings.extensions: [extension]` | [service.ts#L99-L101](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/service.ts#L99-L101) |
| 4. Per-conversation allowlist | `configure(tx, id, { tools: [...reviewTools], extensions: [extension] })` | [task.ts#L213-L219](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/task.ts#L213-L219) |
| 5. The three tools are read-only or output-only | `read`, `grep`, `submit_review` | [tools.ts#L13-L72](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/tools.ts#L13-L72) |
| 6. Filesystem access is confined | `resolveInside` plus size and result limits | [files.ts#L97-L105](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/files.ts#L97-L105) |
| 7. The only output channel is structured | `submit_review` is schema-validated, and the server decides what reaches GitHub | [schema.ts#L74-L86](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/schema.ts#L74-L86) |

Layers 3 and 4 are redundant with layer 2, since the registry holds nothing else to select. They are a second wall: if someone later installs another extension, the per-conversation allowlist still limits what the review conversation is offered. The extension itself also repeats the same three tools ([task.ts#L497-L502](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/task.ts#L497-L502)).

### The three tools

**`read`** ([tools.ts#L13-L29](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/tools.ts#L13-L29)) returns line-numbered text from the checkout. It is capped at 400 lines or 64,000 characters per call, and it refuses files over 2 MiB and files containing NUL bytes ([files.ts#L14-L36](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/files.ts#L14-L36)).

**`grep`** ([tools.ts#L31-L46](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/tools.ts#L31-L46)) searches with a case-insensitive regex, or a literal if the pattern is invalid. It stops at 40 matches or 4,000 files, skips files over 512 KiB, skips symlinks, and skips `.git`, `node_modules`, `dist`, `coverage` and `build` ([files.ts#L38-L95](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/files.ts#L38-L95)).

**`submit_review`** ([tools.ts#L48-L70](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/tools.ts#L48-L70)) validates a TypeBox schema, writes one durable document, and returns `control: { terminate: true }` so the run ends without another model request.

All three are declared `replay: "safe"`. The default for a tool is `unsafe`, meaning an interrupted call is not re-run after a crash. These are safe because two are pure reads of an immutable checkout and the third overwrites one document with the same payload. Decide `replay` explicitly for every tool you add ([tools.ts#L7-L12](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/tools.ts#L7-L12)).

### Path confinement

`resolveInside` ([files.ts#L97-L105](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/files.ts#L97-L105)) takes the real path of the root, resolves the requested path against it, takes the real path of the result, and accepts it only if the relative path does not start with `..` and is not absolute. Resolving through `realpath` before the check is what stops a symlink inside the repository from pointing out of it. Null bytes are rejected up front. `grep` additionally never follows symlinks while walking ([files.ts#L51](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/files.ts#L51)).

[test/files.test.ts#L17](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/test/files.test.ts#L17) ("refuses paths that leave the checkout") covers traversal.

The tools reach the filesystem through the agent's `cwd` and Node `fs` ([tools.ts#L24-L26](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/tools.ts#L24-L26)). They do not use the harness `ExecutionEnv`.

### Mechanisms the library offers

Documented in the package README, for projects that need a different split:

| Goal | Mechanism |
| --- | --- |
| Offer only these tools to a conversation | `configure(..., { tools: [a, b] })` |
| Keep an extension but drop some of its tools | `configure(..., { tools: { remove: ["bash"] } })` |
| Select which extensions a conversation sees | `configure(..., { extensions: [...] })` or `{ add, remove }` |
| Block or rewrite a call at runtime | `hook(ToolTask, { beforeTool: (call) => ({ block: "reason" }) })` |
| Replace or decorate a tool | `wrapTool(tool, wrapper)`. A later extension's tool with the same name also replaces an earlier one |

Source: the "Tools", "Per-Conversation Agent" and "Hooks" sections of the package README ([README.md](https://github.com/earendil-works/pi/blob/main/packages/durable/README.md)). `beforeTool` and `AgentState.tools` also appear in the 1.1.0 type definitions.

If your project needs the real coding tools in some conversations, prefer installing them and dropping them from the conversations that must not have them. Back that with a `beforeTool` hook as a second layer. That is the opposite of dureview's default-deny, so it needs the tests below even more.

### Checklist for another project

1. Do not install `CodingTools`. Provide only the tools the task needs, in your own extension.
2. Give each conversation an explicit `tools` allowlist, even when the registry already limits it.
3. Make every tool either read-only or write to the app's own state. A tool that touches the network, the shell or arbitrary files is a new decision, not a default.
4. Confine every path argument with `realpath` plus a containment check. Skip symlinks when walking.
5. Cap output sizes, file sizes and result counts.
6. Set `replay` on each tool deliberately.
7. Keep side effects outside the model's reach. The model returns structured data, and server code decides what to post.
8. Add the tests below.

### Tests you should add (dureview has a gap here)

dureview has no test asserting that `bash`, `write` and `edit` are absent. The only related tests are the path tests in `test/files.test.ts`. The safety is structural but unchecked, so a future change could add a tool without any test failing. A cheap guard, using a registry API verified in the 1.1.0 type definitions. dureview's `registry` is local to `DureviewService.open`, so you would need to expose it or build the registry in a helper that the test can also call:

```ts
const names = registry.snapshot().tools().map((entry) => entry.tool.name).sort();
expect(names).toEqual(["grep", "read", "submit_review"]);
```

Also worth having: a test that drives the model to call `bash` (the faux provider in [test/support.ts#L160-L175](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/test/support.ts#L160-L175) shows how to script tool calls) and asserts that the result is an error and nothing ran. What the harness returns for an unknown tool name is not verified here.

### Residual risk

- **Prompt injection still works on the model.** The diff and repository files are untrusted input. The model can be steered into wrong findings or into ignoring real ones. It cannot post, edit or execute anything directly, because the server applies a confidence floor, ignore globs, priority limits, `max_inline` and a diff-range check before anything reaches GitHub ([README.md#L194](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/README.md#L194)). The `.dureview.md` rules come from the base branch, not the PR.
- **`read` can read anything inside the checkout.** That includes files the repository owner may consider secret if they were committed, and files under `.git` such as its config. The remote URL is stored without credentials ([types.ts#L15-L16](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/types.ts#L15-L16)). Confirm in your project that the checkout holds no token, and consider adding `.git` to the skipped paths for `read`, not just `grep`.
- **Findings text goes to GitHub.** Whatever the model quotes from a file can end up in a comment on the PR.
- **No network or shell from tools.** True for the three tools as written. Adding any tool that calls out changes this.
