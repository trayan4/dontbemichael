# Proposed enhancement: give non-Claude engines a real system prompt, so their role survives compaction and resume

Tracking issue: https://github.com/agentvivekkumar/dontbemichael/issues/298

## The problem

A Claude Code team member gets its identity, house rules, folder boundaries, work style and
standing goal through `--append-system-prompt` (`src/main/hive.ts:1293`). The code itself says
why that matters: the house rules are "part of the system prompt, so they survive compaction"
(`src/main/hive.ts:214`).

Every other engine gets the same text as the session's **first user message** instead
(`src/main/hive.ts:1121-1137`, returned at `:1271-1272`: a positional arg for Codex and Grok,
`-i` for Antigravity, typed into the TUI for Crush). The comment there admits "after the first
turn the session continues normally."

So on a Codex or Gemini member, the instructions that define the office live in an ordinary
conversation turn. They are subject to:

- **Auto-compaction.** A long-running office agent will compact. The first user message gets
  summarized with everything else, so the agent can lose the exact wording of rules like
  "only Michael assigns work", "stay in your folder", the "no made up information" house rules,
  or its mailbox mode. TODOS.md already notes agents drift "late in a long session or after a
  compaction" on Claude; on the other engines the base instructions themselves are exposed.
- **Resume.** `codex resume <id>` continues the old conversation. Any change made in Edit Agent
  since (work style, role description, standing goal) never reaches the resumed session,
  because the new prompt is not the system prompt and the old one is already in history.

This is a blocker for the repo's stated goal of adding Codex/Gemini to `BUILD_ENGINES`, and it
is not in TODOS.md's 11-item "Make a second engine ready for an office" list or the open issues.

## The proposal

Add an `instructionsDelivery` field to `AgentProviderPreset` (`src/shared/agentProvider.ts`)
and deliver `injectedPrompt(...)` at the system/developer level wherever the CLI supports it:

| Engine | System-level channel (verify against current CLI version) |
|---|---|
| Claude Code | `--append-system-prompt` (unchanged) |
| Codex | `developer_instructions` in the agent's private `CODEX_HOME/config.toml`, which `hive.ts` already writes for hooks |
| Gemini CLI | `GEMINI_SYSTEM_MD` pointing at a per-agent file (or a per-agent `GEMINI.md` context file), alongside the per-agent settings file it already writes |
| No system channel (Crush, proxy tier) | Keep the initial prompt, **plus** re-inject it on `SessionStart` with source `compact`/`resume` via the hook bridge (Codex already has `SessionStart`/`PostCompact` wired at `hive.ts:3711`) |

Then:

1. Keep a short first user message ("You're on shift; check your inbox") so the session still
   starts working, but move the rules out of it.
2. Rewrite the per-agent instructions file on every spawn and resume, so Edit Agent changes take
   effect on a resumed Codex session.
3. Record per engine whether its instructions are "system-level" or "re-injected on compact",
   and have `test/engine-readiness.test.cjs` refuse an engine that has neither, the same way
   issue #139 proposes for the `isolation` capability.

## Tests

- Codex spawn: argv has no protocol positional; `CODEX_HOME/config.toml` has
  `developer_instructions` with the agent's role and house rules.
- Codex resume after editing the work style: the config file has the new text.
- An engine with no system channel: a fake `SessionStart{source:"compact"}` frame through the
  bridge returns the protocol as additional context.
- Live check (cheap): run a Codex worker, force `/compact`, then ask it "whose folder can you
  write to and who assigns your work?" and compare with a run that used the old positional prompt.

## Why this one

It's small (a preset field plus two config writers), it follows patterns the repo already has
(per-agent `CODEX_HOME`, per-agent Gemini settings, hook bridges), and it removes a quiet
correctness gap that would show up as "the Codex agent forgot its job" only after hours of real
use, exactly the kind of bug that's hard to trace back once an owner reports it.
