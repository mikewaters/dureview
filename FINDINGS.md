# Architecture findings

How dureview uses the pi-durable harness: persistence, event flow, control, and runtime shape.

## Scope and method

- Read: all of `src/service.ts`, `src/app/runtime.ts`, `src/review/{task,tools,state,prompt,files,history,usage,ledger}.ts`, `src/auth/session.ts`, `src/config.ts`, `src/server-entry.ts`, the HTTP/UI loaders, the archive write path, `test/support.ts`, `test/resume.test.ts`, `test/e2e.test.ts`, and the README.
- Library claims come from the pinned `@earendil-works/pi-durable@1.1.0`, `chord@1.1.0` and `pi-ai@1.1.0` tarballs (README, CHANGELOG, `.d.ts`). I did not read the library's JavaScript. Behaviors that depend on scheduler internals are marked as library-documented rather than verified.
- Not done: `npm ci`, build, typecheck and tests. This was a read-only walk, so nothing here is confirmed by running the code.
- Line numbers refer to the tree at `3e5a748`.

## Summary

1. **Persistence is a task state machine plus idempotent side effects.** One review is one pi-durable task (`dureview.review`) with three phases (`prepare` → `review` → `post`). Every external effect is bracketed by a checkpoint commit and an idempotency key, so a crash replays instead of duplicating. Everything lives under `DUREVIEW_DATA_DIR` on one volume.
2. **The app does not consume a harness event stream.** There is no `watch`, `viewState`, `watchEvents` or `subscribe` call anywhere in `src/`. The task itself pushes a terminal summary into the app's own SQLite archive through an `onFinished` callback, and the UI reads only that archive plus a live GitHub diff.
3. **Control is narrow: one command in, config out.** The app enqueues tasks, reads live settings once per review, and opens, resumes and closes the harness. It never cancels, inspects or steers a running task. `abortTask` appears only in a test.
4. **The runtime is deliberately minimal.** One extension carries one prompt section, three tools (`read`, `grep`, `submit_review`) and one task. There are **no skills, no MCP, no codemode and no shell or write tools**, in the app or in the pi-durable extension model.

---

## 1. How the harness is made persistent

### Stores

All paths are under `DUREVIEW_DATA_DIR` (`/data` in the image, declared as a `VOLUME` in the Dockerfile).

| File | Owner | Holds | Written by |
| --- | --- | --- | --- |
| `session.sqlite` | pi-durable | conversations, entries (transcripts), tasks and checkpoints, submissions, docs (`ReviewIndex`, `ReviewDraft`, `pi.usage`) | `openNodeSqliteStorage` (`service.ts:95`, `auth/session.ts:18`) |
| `reviews.sqlite` | app | `reviews`, `comments`, `findings`, `settings`, `settings_audit` | `node:sqlite` `DatabaseSync`, `journal_mode = DELETE` (`review/archive.ts:140`) |
| `reviews.json` | app | last 50 review outcomes (legacy log, kept in sync) | `ReviewLog` (`review/history.ts`) |
| `auth.json` | app | Codex OAuth credential, refreshed in place | `FileCredentialStore` (`auth/store.ts`) |
| `checkouts/<owner>/<repo>/<sha>` | app | shallow git checkout of the PR head, plus the base commit at `refs/dureview/base` | `GitCheckout` (`review/checkout.ts:48`) |

`session.sqlite` and `reviews.sqlite` are separate databases. Nothing makes a write to one atomic with a write to the other (see section 5).

### Boot and recovery

- `DureviewService.open` (`service.ts:57`) runs in this order: it opens the archive and settings, builds the extension, calls `registry.install(extension)` (`:90`), calls `Harness.open(storage, {...})` (`:94`), fetches the root conversation (`:117`), then calls `harness.resume()` (`:118`).
- The registry is not stored. Conversations store extension *names*, and tasks are resolved by name, so the extension must be re-installed on every boot before pending tasks can run. That is why `registry.install` precedes `Harness.open`.
- Recovery is eager. `server-entry.ts:14-21` queues `getRuntime()` as a microtask at module load, and a failure there calls `process.exit(1)`. Pending reviews therefore resume at process start and do not wait for the first request. `/health` also awaits the runtime (`routes/health.ts:8`), so the Docker `HEALTHCHECK` stays red until recovery has been started.
- Shutdown (`app/runtime.ts:140`) handles SIGINT and SIGTERM by calling `closeRuntime()`, which calls `harness.close()`. In-flight invocations get an abort signal. `failOrAbort` (`task.ts:623`) checks that signal and rethrows instead of recording a failure, so the checkpoint stays put for the next process.

### The task state machine

`createReviewRuntime` (`task.ts:118`) defines the task. Its checkpoint is a discriminated union, `ReviewState` (`task.ts:97`). Each phase handler does *one* step, commits a new checkpoint, and returns. The code relies on the scheduler re-invoking the handler against the committed checkpoint, which the library documents ("a durable state machine that saves a checkpoint at every step").

```
prepare  (several re-entries of the same phase)
  1. freeze model/provider/thinking from live settings → commit          task.ts:145-168
  2. open the GitHub check run → commit its id                           task.ts:170-200
  3. git checkout + diff + base-branch .dureview.md
     create the task-owned conversation, configure() it → phase=review   task.ts:202-234
review
  submit prompt (requestId dureview:<task>:prompt) → wait                task.ts:272-280
  if no submit_review: one "nudge" submit (…:nudge) → wait               task.ts:283-298
  read ReviewDraft doc → commit review into checkpoint → phase=post     task.ts:303-314
post  (two re-entries)
  A. publishReview() → commit {published, checkOutput}                   task.ts:349-381
  B. finish check run → onFinished(...) → terminal commit "completed"    task.ts:383-436
```

### Crash safety comes from idempotency keys, not from the harness alone

The harness guarantees that checkpoints and conversation state survive. It cannot undo a GitHub write, so each external effect has its own key.

| Effect | Key | Where |
| --- | --- | --- |
| Enqueue the same head twice | `ReviewIndex` doc, written in the same `root.commit` as `tx.createTask` | `task.ts:515-555`, `state.ts:33` |
| Re-submit the prompt after restart | `requestId` `dureview:<task>:prompt` and `:nudge` return the existing submission | `task.ts:273`, `:286` |
| Re-run a tool call after a mid-call crash | `replay: "safe"` on `read`, `grep` and `submit_review` (all pure reads, or an overwrite of one doc with the same payload) | `review/tools.ts:22,39,53` |
| Duplicate GitHub check run | `external_id` = task id, looked up on retry | `review/checks.ts` (`findOwnCheckRun`), `task.ts:471` |
| Duplicate comment or review | hidden HTML markers (`dureview:task`, `:inline`, `:finding <fp> <task>`, `:fixed`) read before every write | `github/publish.ts:28-43`, `review/ledger.ts` |
| Record the publish before completing the check | `published` and `checkOutput` committed first, so a later commit cannot flip the conclusion | `task.ts:372-381` |
| Model changes mid-review | provider/model/thinking frozen in the first `prepare` checkpoint | `task.ts:145-168`, `:670` |

Crash windows and what the tests assert:

| Crash point | On restart | Test |
| --- | --- | --- |
| during checkout | `prepare` re-runs. The checkout is reused when HEAD already equals the SHA (`checkout.ts:52`) | `resume.test.ts:106` |
| after generation, before any GitHub write | `post` publishes once | `resume.test.ts:25` |
| after GitHub accepted the write, before `published` is checkpointed | `publishReview` finds the marker, returns `unchanged`, writes nothing | `resume.test.ts:69`, `:141` |
| after `published`, before the check is completed | `post` re-enters, skips publish, only completes the check | by construction (`task.ts:349`, `:383`) |
| a third boot after completion | nothing re-posts | `resume.test.ts:61-66` |

One consequence worth knowing: a task that is resumed after GitHub accepted its write records `summaryAction: "unchanged"` even though it created the comment (asserted in `resume.test.ts:101`). The result is correct for GitHub's state and slightly misleading as history.

### Provider credentials

`dureview login` writes `auth.json`. pi-ai refreshes the access token inside the credential store's lock and writes the rotated refresh token back. That is the stated reason the service is long-running with a persistent volume. With `DUREVIEW_PROVIDER=openai`, the key comes from the environment and is never written to disk.

---

## 2. How events from the harness reach the app

### There is no event consumption

pi-durable offers three ways to observe a run: `conversation.viewState()` / `watch()`, `watchEvents()` (agent-style events such as `message_update`), and `harness.taskGraph()` / `watchTaskGraph()` / `inspect()`. A grep over `src/` for `watchEvents|viewState|\.watch\(|taskGraph|inspect\(|subscribe|onEvent|for await` finds none of them in application code. The only harness observation outside the task is `harness.waitForTask`, wrapped as `service.wait` (`service.ts:301`). Only tests call it. Production routes return `202 {taskId}` immediately.

### What the app does instead: the task reports out

```
GitHub ──POST /webhook──▶ routes/webhook.ts:10 ─▶ service.handleWebhook (service.ts:310)
                              HMAC check → parse → read .dureview.md from the BASE branch
                              → shouldReviewPull / command permission check
                              ▼
                          service.enqueue (service.ts:143)
                              root.commit{ ReviewIndex dedupe; tx.createTask }      ◀─ session.sqlite
                              then reviews.start + archive.start (row outcome='running')   ◀─ reviews.json / reviews.sqlite
                              ▼ 202 { taskId }

 harness scheduler (same process) runs prepare → review → post
                              ▼
 post phase: onFinished(event)                          task.ts:403 / :684
     event is assembled from the harness at that moment:
       • ReviewDraft doc + checkpoint  → verdict, findings, policy, model, thinking
       • runtime.snapshot(UsageDoc)    → token/cost buckets            task.ts:734
       • runtime.context() entries     → count of AssistantEntry       task.ts:736  (model_calls)
                              ▼
 service.ts:81-88  → ReviewLog.finish (reviews.json)  +  ReviewArchive.finish (reviews.sqlite)

 UI routes → service.reviewsForUi / reviewForUi / statsForUi → reviews.sqlite
             (review detail also calls GitHub listPullFiles live for the diff)
```

Details:

- **`onFinished` also fires on failure.** `failOrAbort` runs `failureNotice`, which builds a `failed` event from the checkpoint and whatever usage exists. The abort handler does not fire it (section 5).
- **`ReviewArchive.finish` is a transactional upsert** (`archive.ts:344-512`): it updates or inserts the `reviews` row, then deletes and re-inserts `comments` and `findings`. The callback runs *before* the task's terminal commit (`task.ts:403` then `:427`), so it is at-least-once. Replaying it is safe for that reason. The only visible effect is that `finished_at` is overwritten with the replay time.
- **Usage is read from the harness ledger, not from provider responses.** `reviewUsageFromLedger` (`review/usage.ts:27`) sums the `pi.usage` model buckets. `modelCalls` is counted separately because the ledger does not count generations. Null means "provider did not report"; zero means "reported zero".
- **Errors from the harness itself** reach the app through `onReport` (`service.ts:106`), which only logs `durable task error`.
- **The UI never reads `session.sqlite`.** Everything the dashboard shows about progress ("running", "疑似卡住" after 30 minutes) is derived from the app's own `reviews` row. The app cannot display harness-level states such as `blocked`, `waiting`, retry backoff or which phase a review is in.

---

## 3. How the harness is controlled by the app

What the app does:

| Action | Call | Where |
| --- | --- | --- |
| Construct | `createRegistry()`, `registry.install(extension)`, `Harness.open(storage, { models, registry, settings, env, onReport }, BACKGROUND_CONTEXT)` | `service.ts:90-111` |
| Start or recover | `harness.root()`, `harness.resume()` | `service.ts:117-118` |
| Submit work | `root.commit(tx => tx.createTask(task, request, { ownership: { kind: "conversation" } }))`, atomic with the `ReviewIndex` write | `task.ts:524-554` |
| Configure a run | inside a task commit: `tx.createConversation({ ownership: { kind: "task", taskId } })` then `configure(tx, id, { model, thinkingLevel, extensions, tools, cwd })` | `task.ts:209-219` |
| Drive a run | `conversation.submit({ type: "input", requestId })` then `.wait()` | `task.ts:272-292` |
| Read state | `runtime.snapshot(ReviewDraft \| UsageDoc)`, `runtime.context()` | `task.ts:281`, `:734-736` |
| End a run early | `submit_review` returns `control: { terminate: true }`, so no further model request is made | `review/tools.ts:67` |
| Stop | `harness.close()` on SIGINT/SIGTERM | `app/runtime.ts:140-148` |

Control inputs from the operator reach the harness only as data read at the start of a review.

- The Settings page (model, thinking, default trigger, draft skip, checkout TTL) writes overrides to `reviews.sqlite`.
- `ServerSettings.effective()` is a synchronous in-memory read (`settings/effective.ts:91`). The `prepare` phase calls it through `currentModel()` (`service.ts:76`), validates the model and thinking level against the pi-ai catalog, and freezes the result on the checkpoint. A change therefore applies to the *next* review and never retargets one in flight.
- The harness's own `settings` (`toolExecution: "sequential"`, `compaction.enabled: false`, `retry.maxRetries: 2`) are static literals (`service.ts:99-104`). The UI cannot change them, even though the library supports live getters there.

What the app never does (grep over `src/`):

- `abortTask`, `abortSubmission`, `Conversation.abort`: no call sites. `abortTask` appears once, in `test/checks.test.ts:497`.
- `inspect`, `waitForIdle`, `taskGraph`, `usage`, `watch`: no call sites.
- Steering or follow-up submits (`whenBusy`), `reset`, `compact`, forks, subagents, hooks: not used.

So the only operator "control" over a running review is stopping the process.

---

## 4. What the harness runtime looks like

### Composition

```
Registry (one extension installed)
└─ extension "dureview"                                  task.ts:497
   ├─ sections: "role" → ADVERSARIAL_ROLE (tag:false)    review/prompt.ts:4
   ├─ tools:    read · grep · submit_review              review/tools.ts:72
   └─ tasks:    dureview.review (v1)                     task.ts:120

Harness settings   extensions:[dureview] · toolExecution:"sequential"
                   compaction:{enabled:false} · retry:{maxRetries:2}   service.ts:99-104
Models             pi-ai createModels(): openai-codex + openai providers,
                   FileCredentialStore(auth.json)                       auth/session.ts:23
Storage            openNodeSqliteStorage(session.sqlite)
Env                NodeExecutionEnv({ cwd })                            service.ts:105
```

Per review there is one *task-owned* conversation. The root conversation exists only to own tasks and to host the session-scoped `ReviewIndex` doc. Reviews are not turns on the root.

### Model and prompt

- Provider is `openai-codex` (ChatGPT OAuth) or `openai` (API key). The default model is `gpt-5.5` at `medium` thinking. The Settings page can override the model, restricted to the pi-ai catalog for the active provider.
- **System prompt** is the single `role` section: the adversarial rubric (severity scale, bug/flag/security types, categories, judgment rules) and the instruction to call `submit_review` once.
- **User prompt** (`buildUserPrompt`, `prompt.ts:53`) carries the PR title and SHA, the diff (truncated at 100,000 characters in `review/diff.ts`), the prose of `.dureview.md` from the *base* branch, the server's filter thresholds, the posting mode, and the output language. A PR therefore cannot change the rules it is reviewed under.
- **Instructions** (`configure({ instructions })`) are not used.

### Tools

| Tool | Behavior | Limits |
| --- | --- | --- |
| `read` | line-numbered text from the checkout | 400 lines or 64,000 characters per call; refuses files over 2 MiB or with NUL bytes |
| `grep` | regex over the checkout, case-insensitive; falls back to a literal match on an invalid regex | 40 matches; 4,000 files; skips files over 512 KiB, symlinks, and `.git`/`node_modules`/`dist`/`coverage`/`build` |
| `submit_review` | validated against a TypeBox schema (`review/schema.ts`), written to the `ReviewDraft` doc | up to 30 findings; terminates the run |

Path confinement is `resolveInside` (`review/files.ts:97`): `realpath` of both root and target, then a containment check. That blocks `..` traversal and symlinks that point outside the checkout.

The tools reach the filesystem through the agent's `cwd` and Node `fs`. They do not use the harness `ExecutionEnv`.

### Safety model

- The model has no shell and no write or edit tool. This is structural: `@earendil-works/pi-durable/tools` (`CodingTools`: `read`, `write`, `edit`, `bash`) is never imported or installed, so those tools do not exist in this registry. The `read` here is the app's own tool, not the library's.
- The model's only side-effect channel is the structured `submit_review` payload. Everything that reaches GitHub is server-side policy applied afterwards (confidence floor, ignore globs, priority and confidence thresholds, `max_inline`, line-in-diff check). A prompt injection in the diff or repository files can bias what the model reports. It cannot post, edit or run anything directly.
- The server computes each finding's fingerprint, and the model does not choose it.

### Skills, MCP, codemode: none

- **In the app:** no code or config references MCP, skills, or codemode. `git log -S` for those terms finds nothing relevant. The `.cursor/skills/*` files are instructions for coding agents working *on* this repo (see `AGENTS.md`). They are not a runtime feature.
- **In pi-durable 1.1.0:** an `Extension` has exactly `name`, `tools`, `sections`, `hooks`, `wraps` and `tasks` (`harness/types.d.ts:206`). There is no skills, MCP or codemode slot. Nothing in the package's README or type definitions mentions them.
- **In pi-ai 1.1.0:** the only mention is a doc comment on `NestedToolCallRecord`, "a tool call that another tool made while it ran, for example from a codemode script". That is a data shape for recording nested tool calls. It is not a feature the app uses or that pi-durable wires up.

If you want any of these, the supported extension points are:

- a tool exposed through `defineExtension({ tools })`, which could bridge to an MCP server;
- `section(...)` for skill-like prompt text;
- `hook(ToolTask, { beforeTool, afterTool })` for guards and logging.

Any new tool needs an explicit `replay` decision, because tools default to *not* replayable.

---

## 5. Observations and risks

Ordered by how likely they are to matter. "Verified" means traced in the code; none of this was exercised at runtime.

| # | Finding | Status | Why it matters |
| --- | --- | --- | --- |
| 1 | **Aborting a task leaves its archive row `running` forever.** The `abort` handler (`task.ts:466-494`) completes the check run and commits `aborted`, but never calls `onFinished`. `runningCheckouts()` selects `outcome = 'running'` (`archive.ts:567`), so that checkout is also exempt from TTL cleanup. | Verified by reading. Not reachable from production code today (no caller of `abortTask`). | The gap appears the moment a cancel feature is added, or if an operator aborts through the harness directly. |
| 2 | **`session.sqlite` grows without bound and holds source content.** The PR diff (up to 100k characters) sits in the `Prepared.diff` checkpoint and in the prompt entry. Every `read` result is in a transcript entry. Only `checkouts/` has a retention sweep. Nothing prunes `session.sqlite`. | Verified: no delete, vacuum or retention code touches the harness storage. | The README's "The diff is not stored" is true of `reviews.sqlite` only. For capacity and for data-handling statements, `session.sqlite` is the larger store and holds more. |
| 3 | **Bumping `version` or changing the shape of `ReviewState` will strand in-flight tasks.** The task is at `version: 1` with no `migrate`. The library reports tasks with a missing, too-old or failed-migration definition as `blocked`. | Verified in `types.d.ts` (`TaskDefinition.migrate`, `TaskInspection` blocked reasons). | A deploy that changes the checkpoint shape mid-review leaves tasks that never finish, and finding 4 hides it. `ReviewIndex` already shows the right pattern (`version: 2` with `migrate`, `state.ts:33`). |
| 4 | **The app cannot see harness-level state.** Progress is derived from its own `reviews` row, so `blocked`, `waiting` and mid-retry tasks look the same as healthy ones until the 30-minute "stuck" label. | Verified (no `inspect`/`watch` calls). | A cheap `harness.inspect()` surface would catch findings 1 and 3. `watch()` / `watchEvents()` would give live phase and token progress if the UI ever wants it. |
| 5 | **No cross-process lock on `session.sqlite`.** The library says "one process owns a storage at a time"; the app adds no guard. The only lock in the app is on the `auth.json` refresh. | Library-documented; app adds nothing. | Two containers on one volume (a rolling deploy, or a second `docker compose up`) would corrupt or double-run work. The README already says to run one process; nothing enforces it. |
| 6 | **Two-database dual write at enqueue.** The task is created in `session.sqlite`, then `reviews.start` / `archive.start` run separately (`service.ts:147-175`). A crash between them leaves a task with no archive row. | Verified. Failures are only logged. | Low impact: `finish` inserts the missing row with trigger `unknown` and a zero-length duration. Until then the checkout is not protected from cleanup. |
| 7 | **`env` factory is inert.** `NodeExecutionEnv` is configured (`service.ts:105`), but no tool or section reads the environment. Also `extensions` and `tools` are given to both `Harness.open` settings and `configure()`, and `configure({ tools })` repeats the extension's own tools. | As far as I can see; not proven absent in library internals. | Candidate for removal under the repo's "subtract before you add" rule. |
| 8 | **Replayed `post` overwrites `finished_at`** (see section 2). | Verified. | Cosmetic. Durations after a crash include the downtime. |
| 9 | **Context overflow behavior is unverified.** With `compaction.enabled: false`, I could not confirm from the docs whether the library's overflow recovery (compact and retry once) still applies. | Unverified. | Large PRs with many `read` calls are the case to test. |

## Appendix: file map

| Concern | Files |
| --- | --- |
| Harness lifecycle, enqueue, webhook handling | `src/service.ts`, `src/app/runtime.ts`, `src/server-entry.ts` |
| Durable task and phases | `src/review/task.ts`, `src/review/state.ts` |
| Model-facing runtime | `src/review/prompt.ts`, `src/review/tools.ts`, `src/review/files.ts`, `src/review/schema.ts` |
| Post-model policy and GitHub writes | `src/review/policy.ts`, `src/review/findings.ts`, `src/review/render.ts`, `src/github/publish.ts`, `src/review/ledger.ts`, `src/review/checks.ts` |
| Outcome capture | `src/review/usage.ts`, `src/review/history.ts`, `src/review/archive.ts` |
| Operator config | `src/config.ts`, `src/settings/*` |
| Auth and models | `src/auth/session.ts`, `src/auth/store.ts` |
| Crash-recovery tests | `test/resume.test.ts`, `test/checks.test.ts`, `test/support.ts` |
