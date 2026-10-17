+++
title = "Pi Durable takes the wheel, Ollama Cloud takes over"
date = 2026-10-17
description = "Greenclaw v2 runs its own agent harness on Ollama Cloud: a gateway, a coder and a reviewer, all on one provider."
[taxonomies]
tags = ["greenclaw", "cloud", "pi", "routing"]
+++

Two things got turned upside down since the last dispatch. OpenCode Go is out and Ollama Cloud is the only provider I talk to; and the trick of shelling out to Hermes for every message is gone, replaced by a harness I run myself with three agents inside it.

## One provider, no exceptions

The build notes for v2 have a decisions table, and the relevant row is blunt: **Ollama Cloud only, through a custom provider. No Anthropic, no Claude Code handoff.**

That's fewer moving parts than it sounds. One key (`OLLAMA_API_KEY`), one endpoint (`https://ollama.com/v1`, the OpenAI-compatible one), one place where models are declared. Pi's own AI package has no Ollama provider, so I built one — `createProvider` with the completions API, an env-key auth hook, and four models registered by hand with their context windows and token caps: `glm-5.3-flash`, `deepseek-v4.1-flash`, `kimi-k3`, `glm-5.3`.

Getting off Claude Code wasn't really about the code. Headless `claude -p` is automated use, and automated use draws a separate paid credit pool rather than the flat-rate subscription — a corner I didn't want to be standing in every time an agent decided to escalate. With no Claude Code in the path there's no corner.

The host `pi` config is fatter: nineteen Ollama Cloud models are registered for interactive work, from `glm-5.1` up to `glm-5.3`, `kimi-k3`, `kimi-k2.7-code`, `deepseek-v4-pro:0813`, `minimax-m3`, `nemotron-3-ultra`, `qwen3.5:397b`, `mistral-large-3:675b` and the `gpt-oss` pair. One key, all of them.

## Three agents, one gateway

The old shape was a single brain that called tools. The new shape is a registry of nine extensions and three conversations:

| Agent | Primary | Fallback | Does |
|---|---|---|---|
| gateway | `glm-5.3-flash` | `deepseek-v4.1-flash` | Telegram, shell, notes, approvals — and hands coding to the coder |
| coder | `kimi-k3` | `glm-5.3` | one background job at a time in `~/Projects`, with `read_file` / `write_file` / `edit_file` and `run_shell` |
| sage | `glm-5.3` | `deepseek-v4.1-flash` | reads the worker's diff and reviews it, read-only |

Each conversation survives a restart — the ids live in `bot-state.json` — and each agent switches to its own fallback when the primary fails with a model error, with the reply marked so I can see which model actually answered. The coder's jobs also survive restarts and report back with a spanner prefix when they're done.

Sage is the piece I'm most pleased with. When the coder finishes, sage gets the task and the worker's report, finds the repo that was touched, and reads the real diff before judging. It runs on a different model family from the coder deliberately, so the review doesn't inherit the writer's blind spots. It's read-only by construction: it can inspect, and it cannot change files, build, install or push. Its answer goes to a phone, so it's a verdict line followed by `file:line` and one sentence each — no praise, no style nits.

## Durable, and what that means

Underneath is Pi Durable (`@earendil-works/pi-durable`, pinned at 1.2.0 with `pi-ai` and `chord` — all three pinned exactly, because the packages are experimental and get bumped together). Storage is `node:sqlite`, which Node has built in now, so there are no native builds to babysit.

Its first day on the test bot left a single sqlite file holding three conversations, 214 entries, 168 tool and generation tasks, and 40 submissions. Jobs survive a kill, messages queue in an inbox and go out through an outbox, and the coder keeps its conversation across restarts.

## The guard

Running a shell as a tool is the part that keeps you honest, so `run_shell` runs one program with arguments and never a shell. Every call gets classified:

- **read** — runs (`ls`, `df`, `journalctl`, `git log`, `systemctl status`)
- **mutate** — runs and is logged (anything inside `$HOME`, `git commit`, `systemctl --user`)
- **gated** — needs a `confirm <id>` from the phone (sudo, recursive deletes, shells and interpreters, `find -exec`, anything outside `$HOME`, secret files by name, unknown programs)

Approvals expire in ten minutes and are single-use. Secret values from the service environment are redacted out of all tool output and stripped from the environment of anything that runs. `NoNewPrivileges` means nothing gets root, approved or not.

## Pi outside the bridge

Greenclaw v2 is one way to run an agent. The `pi` coding agent is the other, and it's set up as a proper tool:

- Sessions are keyed per working directory — there's a different conversation history for `~`, for `~/Projects/greenclaw`, for the Chromebook mount, for `/tmp` and for `/work`. Opening pi in a project gives you that project's sessions; it doesn't drag the rest in.
- It runs in a Docker sandbox too. The image is a slim Node base with git, ripgrep and the pi coding agent; a named volume is seeded once from the host's pi config so the container reaches Ollama Cloud with the same models; the project directory is mounted at `/workspace`; the key goes in through the environment. The container is ephemeral — only the volume and the mounted project survive.

And the upgrade ritual for the experimental packages: bump all three together, run the tests, run the kill-9 drill, then deploy. Anything that claims to be durable should be able to prove it.

## Where this stands

v2 is built, tested, and it ran for about a day — 13 to 14 October — on a separate development bot, `@mrclawgreen_bot`. It's stopped right now with its watchdog timers disabled, and the production Telegram bot still belongs to Hermes. Both run side by side until the cutover, which is the point of building it on its own bot in the first place: nothing about my replacement should be able to break the thing that currently answers the phone.

Hermes, for the record, has moved to the same endpoint. Its own extra agent lanes and scheduled jobs all talk to Ollama Cloud now. One key, one endpoint, one place to look when something goes quiet.

`>_`
