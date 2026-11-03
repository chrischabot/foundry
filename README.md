# Foundry

Foundry is a small coding agent for the terminal, written to study how tools like Claude Code work. It reads and edits files, runs shell commands, delegates to specialist subagents and compacts long sessions, all in about 12,000 lines of TypeScript that you can read through. It is for developers who want to understand the machinery around the model, or who want a compact harness to change and experiment with.

The model call is the simple part of a coding agent. The harder questions start once Claude replies: which of the tools it requested can safely run at the same time, which ones need a person's approval, what to discard when the conversation no longer fits, and how to notice that the agent is going in circles. Foundry answers each of those in plain code, and [Reading the code](#reading-the-code) says where to find each answer. It runs on Bun and talks to Claude through the Anthropic SDK, and at version 0.1.0 it is a study project rather than a finished product. The package and its command are still called `coding-agent`, which is the name you will see in its messages.

## How it works

Each turn streams a response from Claude, collects its tool calls, runs them and appends the results to the thread, and the loop repeats until a response asks for no tools or 50 turns have passed. The rest of Foundry deals with what that cycle runs into on a real project.

Claude can request several tools in one response. Running them one at a time is slower than it needs to be, and running them all at once lets an edit race a read of the same file, so a tool can declare which resources a call touches and whether it reads or writes them. The loop packs calls into batches that contain no conflict, runs each batch concurrently and runs the batches in order. `Read(a)`, `Read(a)` and `Write(b)` therefore go together, while a `Write(a)` from the same response waits for the next batch. The file tools declare their resolved paths. `Bash` declares nothing, because the loop can't know what a command will touch, so shell commands run alongside whatever else is in the batch.

Those tools can also change your machine, so each call is first matched against an ordered list of glob rules, and the first match chooses `allow`, `ask` or `reject`. Commands such as `git push`, `rm -rf` and `sudo` ask first, common read and build commands are allowed so that you aren't approving every `ls`, and any other shell command falls through to `ask`. The rules match the whole command string, so `cat a && b` passes as `cat *`, and `Edit` and `Write` are allowed on any path. That makes them a check against accidents rather than a sandbox, so point Foundry at work you can recover.

Everything the agent reads stays in its history, which makes exploration expensive. `Task`, `Finder`, `Oracle`, `Painter`, `Librarian` and `Kraken` move that work out of the main thread: each is exposed as an ordinary tool but runs a nested loop with its own prompt, message history, tool subset and turn limit, goes through the parent's permission checks and hands back only its final message. `Kraken` works in two phases, with a read-only pass that finds the files and an executor with editing tools and no shell that applies the change.

Long sessions still fill up, so the loop estimates tokens at four characters each and compacts a thread once it reaches about 135,000 of its 150,000-token limit. It keeps the most recent 18,000 or so verbatim, never splitting a tool call from its result, and has Claude summarize the rest along with the current todo list. Later compactions merge new messages into the previous summary rather than starting over. `Read` saves context before it is spent: a code file over 500 lines comes back as its imports and symbol signatures, plus any range requested with an offset, and logs, config, data and large Markdown get their own reductions.

An agent can also keep repeating a mistake, and a file can change while it works. Every third turn the loop inspects recent tool calls and tells the model to change approach if one tool has failed three times in the last ten calls, the last five calls have all failed, or the same sequence of three tools has just repeated. `Edit` needs an exact, unique match and refuses if the file changed on disk since the agent last read it, and both `Edit` and `Write` then run a formatter chosen by file extension when one applies.

## Getting started

You need [Bun](https://bun.sh), [ripgrep](https://github.com/BurntSushi/ripgrep), which the `Grep` and `Glob` tools call, and an Anthropic API key.

```bash
git clone https://github.com/chrischabot/foundry.git
cd foundry
bun install

export ANTHROPIC_API_KEY=your-key
bun run src/index.ts --use-anthropic-key
```

That opens the terminal UI in the Foundry checkout itself, which makes a reasonable first project: you can ask the agent how its own loop works, and a fresh clone is easy to restore. Pass `--workdir /path/to/project` to point it somewhere else. Type `/help` in the UI for its commands and keys. It lists Escape as an interrupt, but that only prints a message and the turn keeps running, so Ctrl+D is the way out. If the UI can't start, Foundry falls back to the readline session.

There are two other ways to run it:

```bash
# One task, then exit. --debug also prints streamed text, tool calls and token usage.
bun run src/index.ts --use-anthropic-key --prompt "Summarize src/agent/loop.ts" --debug

# A readline session. Type exit to quit.
bun run src/index.ts --use-anthropic-key --interactive
```

| Option | Effect |
| --- | --- |
| `-p`, `--prompt <text>` | Run one task and exit |
| `-i`, `--interactive` | Start a readline session |
| `-t`, `--tui` | Start the terminal UI, which is also the default |
| `-w`, `--workdir <path>` | Work in this directory instead of the current one |
| `-m`, `--model <id>` | Use this model for the main agent and its subagents; the default is `claude-opus-4-5-20251101` |
| `-d`, `--debug` | With `--prompt`, print the whole run rather than only the final answer |
| `-y`, `--yolo` | Approve every `ask` rule in `--prompt` and terminal UI runs |
| `--use-anthropic-key` | Authenticate with `ANTHROPIC_API_KEY` instead of OAuth |

An `ask` rule resolves differently in each mode. `--prompt` denies the call unless you pass `--yolo`, and `--interactive` prompts for y or n. The terminal UI prompts too and remembers approvals in `.coding-agent/permissions.json` under the working directory, by tool name and, for `Bash`, by the command's first word. Approving one `git push` therefore also approves the next `git` command that would have asked.

## Authentication

The commands above use `--use-anthropic-key`, which is the mode to choose if you are here to study Foundry, because it is the only one that sends the full system prompt by default: the core rules, your guidance file, skills and each subagent's role prompt. Without the flag Foundry refuses to start until you have signed in with OAuth:

```bash
bun run src/index.ts login    # opens claude.ai, then asks you to paste the code back
bun run src/index.ts status   # shows the account and token expiry
bun run src/index.ts logout   # removes the stored OAuth entries
```

This route is unofficial. [`claude-auth.md`](claude-auth.md) records how it was worked out from the Claude Code CLI bundle and two other clients. It reuses Claude Code's OAuth client, identifies itself as the Claude Code CLI and keeps tokens in `~/.claude.json` beside Claude Code's own settings, and `logout` deletes the OAuth entries from that file, including any that Claude Code itself put there. According to those notes, these tokens accept only a fixed one-line system prompt, so by default Foundry sends that line in place of every prompt it would otherwise use, and the subagents lose their role instructions along with the main agent's rules. Setting `CODING_AGENT_OAUTH_SYSTEM_MODE=full` sends the full prompt after that line, at the risk of the API rejecting it. Expect this route to break without notice, and check it against your Anthropic account's terms before relying on it.

## Tools and configuration

The main agent has 38 tools, registered in [`src/tools/index.ts`](src/tools/index.ts): file editing and search, `Bash`, the six subagents, code intelligence through language servers and ast-grep, planning and scratchpad tools, web fetch and search, and GitHub access. A few depend on programs or keys you supply:

| Tools | Need |
| --- | --- |
| `Grep`, `Glob` | `rg` |
| `GoToDefinition`, `FindReferences`, `GetTypeInfo` | A language server on your `PATH`; [`src/lsp/types.ts`](src/lsp/types.ts) lists each server and its install command |
| `AstGrep` | [ast-grep](https://ast-grep.github.io/) (`sg`) |
| `ReadGitHubFile` and the other GitHub tools | An authenticated [`gh`](https://cli.github.com) |
| `Mermaid` | [`mmdc`](https://github.com/mermaid-js/mermaid-cli) |
| `WebSearch` | `TAVILY_API_KEY` or `SERPER_API_KEY` if set; otherwise DuckDuckGo |

`LookAt` reports an image's or PDF's type and size but does not send it to a vision model.

Foundry builds its system prompt at startup from the environment, the tool list, an `AGENTS.md` or `CLAUDE.md` in the working directory and any skills it discovers. Skills are Markdown files, with optional frontmatter, in `.agents/skills/` and `.claude/skills/` under the working directory and in `~/.claude/skills/` and `~/.claude/plugins/cache/`. Each file's full text goes into the prompt unless its frontmatter sets `isolatedContext: true`, so a crowded plugin cache costs tokens on every request.

Permission rules live in [`src/permission/rules.ts`](src/permission/rules.ts) rather than a config file, and a tool with no matching rule is allowed. Sessions are saved as JSON under `~/.coding-agent/sessions/`. The `ReadThread` tool can read them back, but the CLI has no resume option, so every run starts a new thread.

## Reading the code

| To study | Start with |
| --- | --- |
| The turn cycle, batching and compaction | [`src/agent/loop.ts`](src/agent/loop.ts) |
| Course correction | [`src/agent/correction.ts`](src/agent/correction.ts) |
| Streaming and the summary prompt | [`src/provider/anthropic.ts`](src/provider/anthropic.ts) |
| Permission matching | [`src/permission/checker.ts`](src/permission/checker.ts) |
| Subagent prompts, tool subsets and the nested loop | [`src/subagent/configs.ts`](src/subagent/configs.ts), [`src/subagent/runner.ts`](src/subagent/runner.ts) |
| System prompt assembly | [`src/prompt/system.ts`](src/prompt/system.ts) |
| Large-file reads | [`src/context/truncation.ts`](src/context/truncation.ts) |
| The OAuth flow | [`claude-auth.md`](claude-auth.md), [`src/auth/flow.ts`](src/auth/flow.ts) |

[`AGENT_DESIGN.md`](AGENT_DESIGN.md) walks through the agent hierarchy and prints each subagent's system prompt, and [`AGENTS.md`](AGENTS.md) includes a walkthrough for adding a tool. Both were written before the code-intelligence tools and OAuth, so trust the source where they differ.

## Development

```bash
bun run typecheck   # tsc --noEmit
bun run build       # writes a Bun bundle to dist/
bun run test        # bun test tests/*.test.ts
```

`bun run test` includes `tests/e2e.test.ts`, which has the real agent write and run small programs in TypeScript, Python and Rust and asks Claude to check the code. Those cases need an OAuth login, `ANTHROPIC_API_KEY`, `python3` and `rustc`, and they spend tokens. `tests/tools.test.ts` also fetches from httpbin.org. To run without any of that, name the files you want:

```bash
bun test tests/permission.test.ts tests/execution-profiles.test.ts
```

`tests/compaction-test.ts` and `tests/subagent-test.ts` are standalone scripts that drive live sessions with an API key and write prompts and summaries to `tests/output/` for evaluation.

## Project background

- [Foundry project overview](https://chabot.dev/projects/#foundry) on Chris Chabot’s site.
- [The 57-character system prompt](https://chabot.dev/blog/claude-oauth-detective-story/): an investigation into Foundry’s early OAuth integration.

## Credits and license

Foundry uses the [Anthropic TypeScript SDK](https://github.com/anthropics/anthropic-sdk-typescript) and [OpenTUI](https://github.com/anomalyco/opentui). Its OAuth flow draws on analysis of the Claude Code CLI, the `opencode-anthropic-auth` package and [Link Assistant Agent](https://github.com/link-assistant/agent), as recorded in [`claude-auth.md`](claude-auth.md). OpenCode and OpenTUI's source served as local references during development; nothing in `src/` imports those copies, and the `opentui` entry in the tree isn't needed to build.

Foundry is released under the [MIT License](LICENSE).
