# Memory and communication

Applies for the whole session, from the first action.

`mcp-document-service` is the source of truth for session context and the main channel
between you, sub-agents, and future sessions. Everything below assumes it is available; if
it is not, the last section says how to work without it.

- Start every task with `recall`. `task` is what you are about to do in a few words;
  matching is by word, not meaning, so use words likely to appear in the entries.
  `projectKey` is the repository folder name; omit it outside a repo.
- For exact filters use `searchMemories` (memories) or `searchDocuments` (reference
  documents). Both return summaries; `getDocument` gives the full text and the entry's
  `relations` and `referencedBy`.
- Store anything that may be useful later — findings, decisions, rejected options and why,
  file locations, conventions you had to infer — not only what the user explicitly asks you
  to note.
- Write as you go with `createMemory` rather than batching at the end. The entry you never
  wrote is the investigation someone repeats.
- Prefer memory over re-reading. After a context compaction, reconstruct context with
  `recall` and `searchMemories` first, and read files only for what memory does not cover.
- Leave notes aimed at whoever comes next: what you found, what you decided, what is still
  open.

## Keeping memory work out of replies

Routine memory work is internal. Do it, but don't narrate it: the user wants the answer to
their task, not a log of the store. Don't tell the user what was recalled, saved,
superseded or linked, and never show entry ids in replies to the user.

Mention memory work only when:

- the user asked you to save or find something;
- a document — not a memory — was created or changed;
- `mcp-document-service` is unreachable.

Even then, describe it in words, without entry ids.

Entry ids are still fine between agents: in sub-agent briefs and in a sub-agent's report
back to the main agent.

## Writing an entry

- Search first, so you supersede or extend an existing entry instead of duplicating it.
- `memoryType`: `FACT` (verified), `DECISION` (a choice made and why), `OBSERVATION`
  (noticed, not verified), `PROCEDURE` (steps that worked), `RULE` (how the user or project
  wants things done), `PITFALL` (something that failed or breaks), `STATUS` (where work
  stands).
- `name` is a short title. `summary` is one or two sentences that make sense without the
  content — search and recall show it instead of the content.
- `content` holds the detail and ends with a `Source:` line: the file, command, session or
  person the knowledge came from. There is no separate source field.
- `scope`: `PROJECT` with `projectKey` for knowledge about one repo, `GLOBAL` for knowledge
  that applies everywhere, `AGENT` for session records such as the space-init entry. Use a
  key that `getProjectsInfo` lists; an unknown key creates a new project, which is right
  only for a repo the store has not seen yet.
- Link related entries with `createDocumentRelation`: `DEPENDS_ON`, `EXTENDS`, `PART_OF`,
  `DERIVED_FROM` or `RELATED_TO`. There is no "contradicts" type; use `RELATED_TO` and say
  what conflicts in the content.
- A conclusion drawn from several entries is a new `createMemory` with `DERIVED_FROM`
  relations to its sources.

## Keeping entries current

- When the situation changes, write a new entry that supersedes the old one: pass
  `relations: [{relatedDocumentId: <old id>, type: SUPERSEDES}]` on `createMemory`, or call
  `createDocumentRelation` afterwards. Superseded entries drop out of search and recall, so
  there is one current version and the old one stays as history.
- `updateDocument` overwrites without history. Use it only for small corrections to your
  own entries, such as a typo or a wrong path. It fails on entries with `allowWrite: false`,
  such as ones a person created; supersede those instead.
- `removeDocument` is a soft delete that only a person can undo. Supersede outdated entries
  rather than removing them.

## When `mcp-document-service` is unavailable

Unavailable means its tools are not in the session at all (neither loaded nor deferred), or
its calls fail. A tool listed only by name just needs loading first. The store is a
separate service that the client does not start, so retrying will not bring it back.

Don't stop the session over it, and don't ask the user how to proceed. Memory makes the
work cheaper to resume, but no task depends on it. Warn once, in one line, that memory
tools are unavailable — in the startup report, or when the store first fails mid-session —
then carry on:

- Skip the memory steps: no `recall`, `searchMemories`, `createMemory` or relations, and
  no space-init record.
- Context lives only in this conversation. Read files for what memory would have given
  you. After a compaction, rebuild from the summary and the files.
- Don't invent a substitute store (notes files in the repo or elsewhere) unless the user
  asks for one — that is a file edit (`references/general.md`).
- Don't repeat the warning on every task. Bring it up again only when it matters to the
  answer, for example when the user asks you to save or find something in memory.
- Tell sub-agents in their brief that the store is unavailable, so they skip the memory
  steps instead of probing for it or stopping.

If the store comes back later (the user says so, or a call succeeds), follow the normal
rules from that point and write the space-init record then. Don't try to back-fill
everything from earlier in the session; save what is still worth keeping.
