---
name: "space-init"
description: 'Set up the standing rules for a coding session in any project and point to the rule files that hold them: how to communicate (terse, few words), general rules that apply to every task (doing only what the user asked, answering questions without changing anything, never reading dependency sources without permission), mcp-document-service as the shared long-term memory and communication layer, and comment rules that follow the project''s own style first. Takes no arguments and performs no coding work. Use it whenever the user runs /space-init or says "space-init", "space init", "init the session", "set up the session", "start a coding session", "load the session rules", or asks to restore the working conventions after a context compaction. Run it before any other work when asked, and come back to it later in the session to find which rule file covers the task at hand.'
---

# space-init

Session bootstrap. No arguments, no coding work.

Running this fixes how you work for the rest of the session. These rules outrank your
default habits and stay in force until the user says otherwise. They apply to sub-agents
too — carry them into every brief.

The rules themselves live in `references/`, one file per area. Each file is the authority
for its area and may grow over time, so read the file when its area comes up rather than
working from the one-line summary below. When the user changes a convention, edit the
relevant reference file, not this one.

## Startup sequence

1. Read `references/communication-style.md` first. It governs every response you write
   from here on, including the report at the end of this sequence, so it cannot wait.
2. Check that the `mcp-document-service` MCP is reachable (`getProjectsInfo` is the
   cheapest probe, and it lists the project keys). If it is not, say so immediately and
   ask how to proceed — the memory rules lean on it.
3. Read `references/general.md` and `references/memory.md`. They govern every action from
   here on, so they are not worth deferring.
4. Record that space-init is active with `createMemory`: scope `AGENT`, memoryType
   `PROCEDURE`, `name` "space-init is active". The `name`, `summary` and `content` list
   only the active rule files below — no task, project or repo state. Find the previous
   record with `searchMemories` (scope `AGENT`, query "space-init is active") and supersede
   it: pass `relations: [{relatedDocumentId: <old id>, type: SUPERSEDES}]`, one per current
   record returned, so only the new one stays current. After a context compaction this is
   how the rules come back — re-read the files rather than trusting a summary in the entry.
5. Report in one short line that the session is ready, plus any problems (for example
   `mcp-document-service` unreachable). Don't list recalled or saved entries.

## New session or restore

What happens after the report depends on why the skill ran. Tell the cases apart by what
this conversation already holds; if you can't tell, ask.

### New session

The skill ran at the start of a conversation, with no earlier work in it.

This is a new conversation. The next task may be unrelated to earlier sessions. Prior
memory is background, not the agenda. Don't assume the user wants to continue earlier
work.

Stop and wait for the actual task. Do not start work the user has not asked for. Don't
call `recall` at startup; the first `recall` uses the task's own words once the task
arrives, as `references/memory.md` says. If `getProjectsInfo` does not list the repo's
key, the store has nothing on it yet — expected for a project it has not seen.

### Restore after a context compaction

The conversation already holds work in progress, usually as a compaction summary at the
top. The rules are being reloaded, not the session restarted, so the new-session rule
above does not apply.

Rebuild the working context: `recall` with the words of the task in progress, then
`searchMemories` for what it misses, as `references/memory.md` says. Then continue that
task from where it stopped. Don't wait for a new task or ask the user to repeat it.

## Rule files

| File | Read it | Covers |
| --- | --- | --- |
| `references/communication-style.md` | First at startup, and again after a compaction | How every response is written: few words, terse bullet-like style that keeps every warning and exact detail, directness, plain language, structure, corrections, filler to avoid. Applies to all output, in every brief |
| `references/general.md` | At startup, and again after a compaction | Rules that apply to every task: do only what the user asked — questions get answers, no file edits or other non-read operations without a direct instruction, no suggestions to implement; no decompiling or reading dependency/library sources without the user's permission — ask for the sources instead |
| `references/memory.md` | At startup, and again after a compaction | `mcp-document-service` as source of truth: what to store, when to write, how to write an entry, keeping entries current, recovering context, keeping memory work out of replies |
| `references/service-map.md` | When the first task in a repo arrives, and before any task that touches another service | Using the docs the `service-map` skill builds: finding and reading the repo's service doc and its set index, reading a neighbour's doc before its code, small fixes to map docs, naming map docs in sub-agent briefs, re-reading after a compaction |
| `references/comments.md` | Before writing or changing any comment | Project style first, no new comments in a project that barely has any, then class and method level only and what a comment may say |
