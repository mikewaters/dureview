# Disabling the core coding tools for safety

How dureview guarantees that the model cannot run commands or change files, written so another project can copy the approach.

All source links are permalinks to commit `3e5a748dc8902d73c4126277fdfc56b8bc13803f` of `mikewaters/dureview`.

## What the core tools are

pi-durable ships `read`, `write`, `edit` and `bash` as the `CodingTools` extension, and `createPowerShellTool()` adds `powershell`. The package's own type comment says it all: "`read`, `write`, `edit`, and `bash`; **nothing installs it automatically**" (`dist/tools/index.d.ts` in `@earendil-works/pi-durable@1.1.0`).

That is the basis of the whole approach. The model can only call tools that some installed extension provides. The package README describes tools as coming only from extensions, and I found no built-in tool outside them. Not installing `CodingTools` is therefore enough to make `bash`, `write` and `edit` nonexistent for the model. They are not hidden or blocked, they are just not there.

## What dureview does

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

## The three tools

**`read`** ([tools.ts#L13-L29](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/tools.ts#L13-L29)) returns line-numbered text from the checkout. It is capped at 400 lines or 64,000 characters per call, and it refuses files over 2 MiB and files containing NUL bytes ([files.ts#L14-L36](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/files.ts#L14-L36)).

**`grep`** ([tools.ts#L31-L46](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/tools.ts#L31-L46)) searches with a case-insensitive regex, or a literal if the pattern is invalid. It stops at 40 matches or 4,000 files, skips files over 512 KiB, skips symlinks, and skips `.git`, `node_modules`, `dist`, `coverage` and `build` ([files.ts#L38-L95](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/files.ts#L38-L95)).

**`submit_review`** ([tools.ts#L48-L70](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/tools.ts#L48-L70)) validates a TypeBox schema, writes one durable document, and returns `control: { terminate: true }` so the run ends without another model request.

All three are declared `replay: "safe"`. The default for a tool is `unsafe`, meaning an interrupted call is not re-run after a crash. These are safe because two are pure reads of an immutable checkout and the third overwrites one document with the same payload. Decide `replay` explicitly for every tool you add ([tools.ts#L7-L12](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/tools.ts#L7-L12)).

## Path confinement

`resolveInside` ([files.ts#L97-L105](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/files.ts#L97-L105)) takes the real path of the root, resolves the requested path against it, takes the real path of the result, and accepts it only if the relative path does not start with `..` and is not absolute. Resolving through `realpath` before the check is what stops a symlink inside the repository from pointing out of it. Null bytes are rejected up front. `grep` additionally never follows symlinks while walking ([files.ts#L51](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/files.ts#L51)).

[test/files.test.ts#L17](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/test/files.test.ts#L17) ("refuses paths that leave the checkout") covers traversal.

The tools reach the filesystem through the agent's `cwd` and Node `fs` ([tools.ts#L24-L26](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/review/tools.ts#L24-L26)). They do not use the harness `ExecutionEnv`.

## Mechanisms the library offers

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

## Checklist for another project

1. Do not install `CodingTools`. Provide only the tools the task needs, in your own extension.
2. Give each conversation an explicit `tools` allowlist, even when the registry already limits it.
3. Make every tool either read-only or write to the app's own state. A tool that touches the network, the shell or arbitrary files is a new decision, not a default.
4. Confine every path argument with `realpath` plus a containment check. Skip symlinks when walking.
5. Cap output sizes, file sizes and result counts.
6. Set `replay` on each tool deliberately.
7. Keep side effects outside the model's reach. The model returns structured data, and server code decides what to post.
8. Add the tests below.

## Tests you should add (dureview has a gap here)

dureview has no test asserting that `bash`, `write` and `edit` are absent. The only related tests are the path tests in `test/files.test.ts`. The safety is structural but unchecked, so a future change could add a tool without any test failing. A cheap guard, using a registry API verified in the 1.1.0 type definitions. dureview's `registry` is local to `DureviewService.open`, so you would need to expose it or build the registry in a helper that the test can also call:

```ts
const names = registry.snapshot().tools().map((entry) => entry.tool.name).sort();
expect(names).toEqual(["grep", "read", "submit_review"]);
```

Also worth having: a test that drives the model to call `bash` (the faux provider in [test/support.ts#L160-L175](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/test/support.ts#L160-L175) shows how to script tool calls) and asserts that the result is an error and nothing ran. What the harness returns for an unknown tool name is not verified here.

## Residual risk

- **Prompt injection still works on the model.** The diff and repository files are untrusted input. The model can be steered into wrong findings or into ignoring real ones. It cannot post, edit or execute anything directly, because the server applies a confidence floor, ignore globs, priority limits, `max_inline` and a diff-range check before anything reaches GitHub ([README.md#L194](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/README.md#L194)). The `.dureview.md` rules come from the base branch, not the PR.
- **`read` can read anything inside the checkout.** That includes files the repository owner may consider secret if they were committed, and files under `.git` such as its config. The remote URL is stored without credentials ([types.ts#L15-L16](https://github.com/mikewaters/dureview/blob/3e5a748dc8902d73c4126277fdfc56b8bc13803f/src/types.ts#L15-L16)). Confirm in your project that the checkout holds no token, and consider adding `.git` to the skipped paths for `read`, not just `grep`.
- **Findings text goes to GitHub.** Whatever the model quotes from a file can end up in a comment on the PR.
- **No network or shell from tools.** True for the three tools as written. Adding any tool that calls out changes this.
