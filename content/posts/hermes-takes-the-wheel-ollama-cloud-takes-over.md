+++
title = "Hermes takes the wheel, Ollama Cloud takes over"
date = 2026-10-17
description = "The cloud switch is settled — everything here talks to Ollama Cloud. So is the multi-agent question, and the answer was Hermes, not the Pi Durable rewrite."
[taxonomies]
tags = ["greenclaw", "cloud", "hermes", "pi", "routing"]
+++

Two things settled since the last dispatch. The cloud is Ollama Cloud and nothing else. And the wheel went to Hermes — not to the rewrite I was building to replace it. I had that backwards in an earlier version of this post, so: the Pi Durable harness was built and parked; the thing actually running the show is Hermes.

## One provider, no exceptions

The build notes for the parked rewrite have a decisions table with a blunt row: **Ollama Cloud only, through a custom provider. No Anthropic, no Claude Code handoff.** That decision outlived the rewrite, and everything here now follows it.

One key (`OLLAMA_API_KEY`), one endpoint (`https://ollama.com/v1`, the OpenAI-compatible one), one place where models are declared. Pi's own AI package has no Ollama provider, so the rewrite hand-built one — `createProvider` with the completions API, an env-key auth hook, four models registered with their context windows and token caps. The host `pi` config is registered wider: nineteen Ollama Cloud models, from `glm-5.1` through `glm-5.3`, `kimi-k3`, `kimi-k2.7-code`, `deepseek-v4-pro:0813`, `minimax-m3`, `nemotron-3-ultra`, `qwen3.5:397b`, `mistral-large-3:675b` and the `gpt-oss` pair.

Getting off Claude Code wasn't really about the code. Headless `claude -p` is automated use, and automated use draws a separate paid credit pool rather than the flat-rate subscription — a corner I didn't want to be standing in every time an agent talked itself into escalating. No Claude Code in the path, no corner.

## Hermes took the wheel

The thing that answers the phone is Hermes itself now, running as a systemd user service on the home box, not a bridge sitting beside it. Its provider is `ollama-cloud`, pointed at the same endpoint, so the assistant on the other end of Telegram is a Hermes agent on Ollama Cloud models.

I should have called this months ago. The previous arrangement had me shelling out to Hermes for every message: a hand-rolled loop calling a better loop. The version that won was the better loop, doing its own routing, on the same provider as everything else.

## Agents are profiles now

This is the part that's actually new, and it's the reason the rewrite lost its purpose. Hermes runs named agents as worker lanes, and a lane is just a profile: its own model, its own memory, its own skills, its own sessions. Hal is the first one — its profile describes it as a general-purpose named agent running on Ollama Cloud `glm-5.3`, and it has its own memory directory, so what it learns on one job is there on the next.

Work goes through a job board. A card names an assignee, and the assignee string *is* the profile name:

- A dispatcher inside the gateway claims ready cards on a tick of about a minute and spawns `hermes -p <assignee> chat -q "work kanban task <id>"` as a real OS process in that task's workspace. A long job is a process that can outlive the conversation that asked for it — not a subagent call that dies with the turn.
- If the assignee doesn't resolve to a real profile, nothing is run and nothing falls back: the card parks in `ready` with a `skipped_nonspawnable` event. There's a card on my board assigned to `ghost` that exists purely to keep that honest.
- Blocked is the input path. A worker that needs a decision blocks and says why; the answer goes back as a comment, and the card is unblocked. The comment is the input — unblocking on its own just respawns the worker to fail the same way twice.
- Reviews are dispatched too, so finished work can be handed to another lane instead of being taken on trust.
- Outcomes reach a phone through a script-only cron job that reads the board read-only every five minutes and pushes done, blocked and failed events. It dedupes by event id and prints nothing when there's nothing new, which is the only behaviour I want from a notifier.
- A worker has no chat channel of its own. The board and the CLI are the ways in — there's no DMing a lane.

The board right now is five cards, and I'd rather say it plainly: every one of them is a test. Three of them exercise Hal's outcome paths — one done, one blocked on purpose, one timed out on purpose — one is the ghost that proves an unresolvable assignee is parked rather than guessed at, and one checks that the relay end-to-end actually delivers. Proven on the test cases; it hasn't carried real work yet.

## The road not taken

For completeness, because I built it and it's sitting there: the rewrite was a Telegram bridge on Pi Durable (`@earendil-works`, pinned at 1.2.0 with `pi-ai` and `chord`, all three experimental and bumped together), with three agents inside one harness — a gateway on `glm-5.3-flash`, a coder on `kimi-k3` doing one background job at a time in `~/Projects`, and a read-only reviewer on `glm-5.3` that inspects the actual diff, runs on a different model family from the writer on purpose, and answers with a verdict line and `file:line` issues.

It had a decent guard, too: `run_shell` runs one program with arguments and never a shell, every call is classified as read, mutate or gated, gated calls need a single-use `confirm <id>` that expires in ten minutes, secrets are redacted out of tool output and stripped from the environment of anything that runs, and `NoNewPrivileges` means nothing gets root, approved or not. Storage was `node:sqlite` — no native builds — and its first day on a development bot left three conversations, 214 entries, 168 tool and generation tasks and 40 submissions in one file.

It ran on a separate bot for about a day, 13 to 14 October, and it's stopped now with its watchdog timers disabled. Hermes never stopped, so there was nothing for the replacement to replace.

## Where this stands

The cloud is one provider on one key, and it isn't moving. Hermes holds the bot and the lanes, with Hal as the first working agent and the board as the queue and the audit trail. The Pi Durable build is parked in the garage, and the good parts of it — the reviewer on a foreign model, the command classification — are the parts worth stealing the next time a lane needs them.

`>_`
