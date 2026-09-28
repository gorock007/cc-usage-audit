# cc-usage-audit

Find out where your Claude Code usage actually goes.

Claude Code keeps a transcript of every session on your machine, and every
reply in it records how many tokens it used. This script reads those
transcripts, prices each request at API rates, and tells you what drove the
spend. It answers questions `/usage` doesn't:

- Which **models** and **session types** (interactive, subagents, headless/hooks) cost the most
- How much went to **output**, **cache writes** and **cache reads**, and how much happened past 150k context
- **Cache reloads**: turns that wrote the whole context to the cache again, split into "after an idle gap longer than the cache TTL" and "other" (compaction, `/model` switches)
- **Subagents** by type and the model they actually ran on
- **Headless sessions** (Agent SDK, `claude -p`, plugin hooks) grouped by how their prompt starts. This is how you find a hook that calls a model every time a session stops
- **What each turn was doing** (reading, git, tests, sleep loops, your own categories) and what those turns cost
- Your **worst 5-hour window** and the most sessions you ran at once

It's local only: no network calls, no dependencies, one file, Python 3.8+.

## Run it

```bash
curl -O https://raw.githubusercontent.com/<you>/cc-usage-audit/main/cc_usage_audit.py
python3 cc_usage_audit.py --days 7
```

Useful flags:

```bash
python3 cc_usage_audit.py --since 2026-09-16 --until 2026-09-28   # a fixed period
python3 cc_usage_audit.py --project my-repo                       # only folders containing this
python3 cc_usage_audit.py --redact                                # hide project names and prompts, for screenshots
python3 cc_usage_audit.py --json > usage.json                     # raw aggregates
```

### Label your own session types

If you run an orchestrator and workers, tell the script how to recognise them
and every table splits by role:

```bash
python3 cc_usage_audit.py --days 14 \
  --role worker="You are a worker"               \  # first prompt matches this regex
  --role-tool orchestrator="agents create"       \  # any tool call input matches this regex
  --bash-category coordination="terminals read|handoffs/|STATUS\.md"
```

`--role` matches the session's first prompt, and `--role-tool` matches the
input of any tool call in the session. `--bash-category` adds your own
categories for Bash turns, checked before the built-in ones.

## Reading the report

- **Dollars are API-equivalent.** They're what the tokens would cost on the API, used as a proxy for how fast you use up a Claude subscription. Anthropic doesn't publish plan limits in dollars, so compare the numbers to each other, not to your bill.
- **Cache writes vs reads.** Every turn re-sends the whole conversation. Cached context is cheap to re-read, but anything new that enters the context is written to the cache at 1.25× (5-minute) or 2× (1-hour) the input price. If writes dominate, the fix is to put less into context or to rewrite it less often, not to shorten the conversation.
- **Cache reloads "after idle"** mean a session sat longer than the cache TTL (one hour on a subscription, five minutes on API keys and usage credits), so the next turn paid to write the whole context again. Long-lived sessions that wait on something are the usual cause.
- **Turn categories** give each request the first tool it called. The cost of a turn is mostly the context it carries, so a cheap-looking action ("check status") costs about as much as real work.
- Only sessions on **this machine** are included, not claude.ai or your other devices.

## Prices

Prices per million tokens are at the top of the script (`PRICES`). They were
current when this was written; check them against the
[pricing page](https://docs.claude.com/en/docs/about-claude/pricing). You can
override or add models without editing the script:

```json
{ "claude-some-new-model": [3.0, 15.0, 0.30, 3.75, 6.0] }
```

```bash
python3 cc_usage_audit.py --prices my-prices.json
```

The order is input, output, cache read, 5-minute cache write, 1-hour cache
write. Models with no price are listed at the end of the report instead of
being silently dropped.

## How it works

Transcripts live in `~/.claude/projects/<project>/<session>.jsonl`, with
subagents in `<session>/subagents/agent-*.jsonl` (and their type in
`.meta.json`). Streamed replies write the same usage block once per content
chunk, so requests are de-duplicated by `requestId`. Each request is priced
from its `usage` fields, including the split between 5-minute and 1-hour cache
writes.

## License

MIT
