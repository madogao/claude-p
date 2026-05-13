# claude-p

A drop-in replacement for `claude -p` (Claude Code's print mode) that works by
driving the interactive `claude` UI inside an in-process [zmux][zmux] PTY
session. Use it when `-p` itself isn't an option (sandboxed environments,
features that only work in interactive mode, etc.) but you still want
non-interactive, scriptable output.

Output on stdout is byte-for-byte the same as what `claude -p` would print.

> **Status:** experimental. macOS and Linux only. Needs the `claude` CLI on
> your `$PATH`.

[zmux]: https://github.com/smithersai/zmux

## Install

### npm (recommended)

```bash
npx claude-p "what is 2 + 2?"
```

A small Node shim downloads the prebuilt Zig binary for your platform on first
run. No build toolchain required.

### From source

```bash
git clone https://github.com/williamcory/claude-p
cd claude-p
zig build -Doptimize=ReleaseSafe
./zig-out/bin/claude-p "hello"
```

Requires Zig **0.15.2**. Dependencies are fetched by `zig build`.

## Usage

```
claude-p [OPTIONS] [PROMPT]
```

If `PROMPT` is omitted, reads from stdin.

### Examples

```bash
# Plain text (default) — exactly what `claude -p` prints.
claude-p "summarize this commit" < commit.diff

# Single-shot JSON result.
claude-p --output-format json "write a haiku"

# Streamed JSONL transcript — matches `claude -p --output-format stream-json --verbose`.
claude-p --output-format stream-json --verbose "audit src/" | jq .

# Pick a model.
claude-p --model opus "explain quicksort to a 10-year-old"

# Multiline prompt from a file.
claude-p --input-file ./prompt.md

# Pipe in.
echo "describe yourself in one sentence" | claude-p
```

### Output formats

| `--output-format` | Stdout shape                                                       |
| ----------------- | ------------------------------------------------------------------ |
| `text` (default)  | Final assistant message text + trailing `\n`.                      |
| `json`            | One JSON object: `{type, subtype, session_id, result, ..., usage}`.|
| `stream-json`     | JSONL replay of the full session, ending with a `result` event.    |

`json` and `stream-json` shapes are the same shapes documented for
`claude -p --output-format json | stream-json`. If `claude` adds a field, it
will appear here too (we just forward whatever ends up in the session JSONL).

### Flags

| Flag | Default | Forwarded to `claude`? | Notes |
| ---- | ------- | ---------------------- | ----- |
| `--output-format <fmt>` | `text` | no | `text` \| `json` \| `stream-json` |
| `--model <name>` | (claude default) | yes | Same names as `claude --model`. |
| `--max-turns <N>` | unlimited | yes | Aborts with exit `124`. |
| `--allowedTools <list>` | — | yes | Permission-rule syntax (`"Bash(git diff *)" Read Edit`). |
| `--dangerously-skip-permissions` | off | yes | Skip the permission prompts. |
| `--resume <id>` / `--continue` | — | yes | Reuse a session. |
| `--session-id <uuid>` | (new) | yes | Force a UUID. |
| `--cwd <path>` | `$PWD` | no | Working directory for the child. |
| `--input-file <path>` | — | no | Read prompt from file. |
| `--verbose` | off | yes | Required for full `stream-json` output. |
| `--timeout <seconds>` | `300` | no | Wrapper-level wall-time cap. |
| `--debug` | off | no | Verbose wrapper logs to stderr. |

Any flag claude-p doesn't recognize is forwarded verbatim to `claude` (so it
stays useful as Claude Code evolves).

### Exit codes

| Code | Meaning |
| ---- | ------- |
| `0`  | Success (assistant returned a final message; no error). |
| `1`  | The assistant returned an error (`is_error: true`) or transcript missing. |
| `2`  | Wrapper internal error (couldn't spawn `claude`, ghostty init failed, etc.). |
| `124`| Timed out, or `--max-turns` exceeded. |
| `130`| Interrupted (SIGINT). |

## How it works (one paragraph)

`claude-p` spawns `claude` interactively inside a [zmux][zmux]
`NativeSession` — a tmux-style PTY wrapper that owns the child process, a
reader thread, and a bounded scrollback ring. A small ANSI scanner on our
side answers the DA1 / DA2 / DSR / XTVERSION / window-size queries Ink
(the React-for-terminals runtime Claude Code is built on) issues at
startup so the UI doesn't hang. We register two hooks via the `--settings`
flag: `SessionStart` (so we know when Ink is ready for typing) and `Stop`
(so we know the model is done). When `Stop` fires we read the session
transcript JSONL, extract the final assistant message + usage, and print
it in the exact format `claude -p` would have. No global config or
filesystem state is left behind.

## Use as a Zig library

```zig
const std = @import("std");
const claude_p = @import("claude_p");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const alloc = gpa.allocator();

    var result = try claude_p.run(alloc, .{
        .prompt = "what is the capital of France?",
        .output_format = .text,
    });
    defer result.deinit(alloc);

    std.debug.print("{s}\n", .{result.text});
}
```

See [`SPEC.md`](./SPEC.md) for the full API surface.

## Caveats

- macOS / Linux only (no Windows; ConPTY shim TBD).
- Adds ~50-200 ms over `claude -p` due to PTY + Ink startup overhead.
- Multiline prompts must come via `--input-file` or stdin (single-line on the
  CLI to keep escaping sane).
- Streaming is **buffered**: tokens are not streamed live — `claude-p` waits
  for the model's turn to finish, then prints. (Use the real `claude -p
  --output-format stream-json` if you need true streaming.)

## License

MIT.
