# Eager recovery of interrupted durable work

How dureview resumes reviews that a previous process left unfinished, at process start and not on the first request. Written so another project can copy the approach.

All source links are permalinks to commit `3e5a748dc8902d73c4126277fdfc56b8bc13803f` of `mikewaters/dureview`.

## What "eager" means here

pi-durable stores task checkpoints in SQLite, but nothing runs them until the scheduler is enabled. The library enables it lazily: `submit()`, `wait()` and similar calls start it, and so does `harness.resume()` ([`Harness.resume`](https://github.com/earendil-works/pi/blob/main/packages/durable/README.md#persist-and-resume) in the package README). If the app only opened the harness when the first request arrived, a review interrupted by a deploy would sit idle until some webhook or page view happened to trigger it.

dureview avoids that in three steps:

1. **At module load**, the server entry asks for the runtime in a microtask. No request is needed.
2. **While opening the runtime**, the service installs its extension, opens storage, and calls `harness.resume()` explicitly.
3. **Every phase of the task is written to be re-entered** after a crash, so resuming is safe.

## Step 1: the boot trigger

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

## Step 2: a singleton runtime that resumes while opening

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
- Resumed work starts concurrently with HTTP startup. Anything a resumed task needs (GitHub client, models, credentials, settings) must be ready before `resume()`. dureview builds all of them before calling `DureviewService.open`. Credential trouble does not block boot, only the affected review ([runtime.ts#L65-L70](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/app/runtime.ts#L65-L70)). See the [Codex auth guide](https://github.com/mikewaters/dureview/blob/claude/architecture-findings/docs/codex-auth-persistence.md).

## Step 3: tasks that are safe to resume

Eager resume only helps if re-running a half-finished task is safe. dureview does four things.

**Checkpoint after every step.** Each phase handler does one step, commits a new checkpoint, and returns ([src/review/task.ts#L124-L464](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/task.ts#L124-L464)). The code relies on the scheduler calling the handler again with the committed checkpoint.

**Idempotency key for every outside effect.** The harness cannot undo a GitHub write, so each write has its own key: a check run matched by `external_id`, comments matched by hidden markers read before writing, and prompts submitted with a fixed `requestId`. The table is in [FINDINGS.md](https://github.com/mikewaters/dureview/blob/claude/architecture-findings/FINDINGS.md#crash-safety-comes-from-idempotency-keys-not-from-the-harness-alone).

**Record the result before the final step.** The `post` phase commits `published` and `checkOutput` before it completes the check run ([task.ts#L349-L381](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/task.ts#L349-L381)). A crash between those steps re-enters `post`, skips publishing, and only finishes the check.

**Tell shutdown apart from failure.** On SIGINT or SIGTERM the harness aborts the running invocation's signal. `failOrAbort` checks it and rethrows instead of writing a failed outcome ([task.ts#L623-L659](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/task.ts#L623-L659)), and each phase's `catch` does the same ([e.g. #L236](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/task.ts#L236)). Without this, every deploy would permanently fail the reviews it interrupted. The shutdown handlers are installed once ([runtime.ts#L140-L148](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/app/runtime.ts#L140-L148)) and call `closeRuntime`, which closes the harness, settings and archive.

## Checklist for another project

1. Define durable tasks with a checkpointed state machine. Give every external side effect an idempotency key, and make "already done" a normal path.
2. Install every extension before `Harness.open`, then call `resume()` explicitly. Do not rely on a later `submit()` or `wait()` to start the scheduler.
3. Trigger runtime creation at process start, not from the first request. Memoize the promise so that boot and requests share one open.
4. Exit non-zero if the boot open fails.
5. Wire `/health` to the same promise so the orchestrator sees "not ready" until recovery has started.
6. Handle SIGINT and SIGTERM by closing the harness. In task code, rethrow when `context.abortSignal?.aborted` instead of recording failure.
7. Run exactly one process per storage directory. The library has no cross-process lock ([README.md#L65](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/README.md#L65)), and a rolling deploy with overlap would run the same tasks twice.
8. Guard the boot trigger so builds and tests do not start the runtime.

## How to test recovery

[test/resume.test.ts](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/test/resume.test.ts) is the pattern. Open a service on a temp data directory, install a test hook that waits until the abort signal fires (`hangUntilAbort` in [test/support.ts#L492-L499](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/test/support.ts#L492-L499)), call `close()`, then open a second service on the same directory and `wait()` for the task. Assert the number of external writes. It covers four windows:

- before any write ([#L25](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/test/resume.test.ts#L25));
- after the write was accepted but before it was checkpointed ([#L69](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/test/resume.test.ts#L69));
- during checkout ([#L106](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/test/resume.test.ts#L106));
- the inline-comment mode ([#L141](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/test/resume.test.ts#L141)).

The first test also reopens a third time and checks that nothing re-posts.

## Pitfalls seen in the code

- **A failed open is cached.** `getRuntime` keeps `opening` after a rejection ([runtime.ts#L26-L30](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/app/runtime.ts#L26-L30)), so a later request gets the same error. This is acceptable only because the boot trigger exits the process. If your project retries instead of exiting, clear the promise on failure.
- **Shutdown is not time-bounded.** `closeRuntime().finally(() => process.exit(0))` waits as long as `harness.close` takes, because it runs under a context that cannot be cancelled. The orchestrator's SIGKILL is the backstop, and that looks like a crash, which recovery handles. How long `close` takes with a review mid-flight is not verified.
- **A replayed `post` repeats the completion callback.** The archive write is an idempotent upsert, but it overwrites `finished_at`, so durations after a crash include the downtime.
- **Changing the checkpoint shape strands in-flight tasks.** The review task is `version: 1` with no `migrate`, and the library reports tasks whose definition cannot take them as blocked. Add `migrate` before changing `ReviewState`.
- **Dev mode.** Whether `vite dev` evaluates the server entry at startup, and so boots the runtime without a request, is not verified.
