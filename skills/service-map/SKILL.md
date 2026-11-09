---
name: "service-map"
description: 'Map one microservice into mcp-document-service so agents can understand a multi-service system without reading every repo. Each run writes or updates a short service doc (purpose, inbound, outbound, cross-cutting rules, conventions, tests, open GAP questions), keeps one index doc per set of related services with the conventions they share, links docs with PART_OF and DEPENDS_ON, and answers GAP questions that earlier runs left in other services'' docs. Use it whenever the user runs /service-map or asks to map a service, add a repo to the service map, document a microservice for agents, refresh a service''s map entry, or fill service-map gaps, even without the words "service map" when the goal is a cross-service picture stored in the document service. Not for full service documentation, READMEs, AGENTS.md files, or answering one question about how some code works.'
---

# service-map

One run maps one service. It produces or updates:

1. **The service doc**: one per service, in the service's own project. Later runs update it in
   place, so its id stays stable; other docs link to it.
2. **The set index doc**: one per set of related services. One line per service, plus the
   conventions two or more services share. Shared conventions live only here.
3. **Relations**: service doc `PART_OF` index doc; service doc `DEPENDS_ON` the doc of each
   service it calls. `recall` follows relations across projects, so these links are how an
   agent working in one repo finds the neighbouring services.

Every run also answers questions earlier runs left open in other services' docs, so the map
gets more complete with each service analyzed.

This is a map for agents, not a manual. A service doc should read in about 2k tokens.

## Input

- The service: a repo path. Default: the current working directory.
- Optional: the set key (for example `company-platform`). If missing, work it out (step 3).

## Before you start

- Session rules come from hl-init or space-init: their `references/communication-style.md`
  and `references/memory.md`. If neither skill ran in this session, read those two files from
  whichever is installed before step 1. Routine memory work stays silent.
- Probe the store with `getProjectsInfo`. If `mcp-document-service` is unreachable, stop and
  tell the user; this skill has nothing to write to.
- Read `references/templates.md` before drafting any doc, and `references/code-discovery.md`
  before reading code.

## Names and search

Search is word-based Postgres full text (`websearch_to_tsquery`, English config), not
semantic. Gap filling only works if every run writes names the same way and searches for them
the same way. These rules were verified against the live store on 2026-10-15:

- **Service names** exactly as callers use them: `spring.application.name`, the Feign
  `name`, the discovery id. Record other names (artifactId, k8s service name, repo folder) as
  aliases in Identification, because another service may only know one of them.
- **Paths** in full, as the service itself serves them: context path + class mapping +
  method mapping, no host, no gateway prefix. **Write every path variable as `{}`**:
  `POST /api/v1/orders/{}/cancel`. Variable names are search tokens, so `{id}` and
  `{orderId}` never match each other.
- **Topics, queues, gRPC services, class names** exactly as in code or config.
- **Quote every name and path in a query**: `"billing-service"`. Unquoted, a name that
  matches nothing falls back to any-word search and returns every doc containing "service".
- **The word "gap" appears only on GAP lines.** The section is called "Open questions";
  summaries and the index doc never say "gap". "gaps" stems to "gap", so if the word shows
  up anywhere else, `GAP "billing-service"` stops meaning "has an open question and mentions
  billing-service".
- Several terms in one query: `GAP "a" OR GAP "b"`. Parentheses don't group.
- A resource base path matches its sub-paths (`"/api/v1/orders"` finds
  `/api/v1/orders/{}/cancel`). A short name does not match a longer hyphenated one
  (`"auth-service"` misses `auth-service-v2`), so search each alias separately.
- To find another service's doc: query `"<name>" "service map"`, scope `PROJECT`, and take
  the result named exactly `<name> (service map)`.
- Search pages hold at most 50 results. If a page comes back full, fetch the next one.

## Gaps

A gap is a question this run can't answer from its own code, almost always about another
service. It is one line in the Open questions section:

```
- GAP: who calls POST /api/v1/orders/{}/cancel?
- GAP: who produces topic payment.completed.v1?
- GAP: what does notification-service do with topic order-events?
- GAP: which service is behind property pricing.base-url?
```

- Use the exact names and paths from the rules above, so a later run can find the line.
- The entry the gap belongs to says `unknown` (`Callers: unknown.`), so a reader of Inbound or
  Outbound sees the hole without jumping to the end.
- **Closing a gap**: delete the GAP line and write the fact into its entry, with how you know
  it: `Callers: order-service (Feign client OrderCancelClient).`
- A caller or consumer list means "known among mapped services". When a later run finds
  another caller, it appends to the list; the gap closes with the first one found.
- Never invent an answer. If the code only hints (a URL built at runtime, a topic name from an
  env var you can't resolve), leave the gap open, and sharpen the question if you learned
  something.

## Workflow

### 1. Identify the service

- Repo root: `git rev-parse --show-toplevel` (or the given path). `projectKey` = the root
  folder's name.
- Commit: `git rev-parse --short HEAD`; if `git status --porcelain` prints anything, note
  "with uncommitted changes". Date: today.
- Service name, aliases, groupId, artifactId, base package (see `code-discovery.md`).
- If the repo holds several deployable services, ask the user which one to map.
- `recall` with the service name and "service map", `projectKey` set. Prior memories about the
  repo can save code reading.

### 2. Find existing docs

- The service doc: `searchDocuments` with `projectKey`, scope `PROJECT`, query
  `"service map"`, `includeRelations: true`. It is the result named
  `<service-name> (service map)`. If found, its `PART_OF` target is the current set.
- Index docs: `searchDocuments` scope `PROJECT`, query `"service map index"`,
  `includeSummary: true`, size 50.

### 3. Decide the set

Skip this when the user named the set or the service doc is already `PART_OF` an index (unless
the code now points elsewhere; then ask).

Evidence, strongest first: shared parent POM or BOM, shared internal libraries, groupId and
base package prefix (`com.company.*`), deployment files that list several services (compose,
helm, k8s), config that names other services. Read each index doc's "Set evidence" section
and compare.

- Clear match: use that set.
- Strong evidence, no match: create a new set. Key: the shared identifier in lowercase with
  hyphens, for example parent `company-platform-parent` gives `company-platform`.
- Weak or conflicting evidence: ask the user, showing the evidence and a proposed key.

Two services in one set may never talk to each other. That's normal.

### 4. Read the code

Follow `references/code-discovery.md`. For a large repo you may split the reading across
read-only sub-agents (inbound, outbound, cross-cutting + conventions + tests). Give each the
naming rules above and the template sections it fills, and have it return findings as text.
Sub-agents never write to the service or index docs.

### 5. Gather what the map already knows

- Set members: `getDocument` on the index doc; its `referencedBy` entries of type `PART_OF`.
- Docs that mention this service: `searchDocuments` scope `PROJECT`,
  query `"<service-name>" OR "<alias>" ...`, size 50; keep set members only. From their
  Outbound sections take who calls which of your endpoints, who consumes your topics, and who
  produces the topics you consume. That fills your own `Callers:`, `Consumers:` and
  `Producer:` entries.
- The index doc's shared conventions, so your doc lists only differences.

### 6. Update the index doc

Do this before the service doc, because a new service doc needs the index id for `PART_OF`.

- New set: `createDocument` per the index template.
- Existing set: `getDocument` right before writing, then add or refresh this service's single
  line and update Set evidence if you found more.
- Conventions: for each convention of this service not yet in the index, search set members
  for a distinctive word of it (`"MapStruct"`, `"ProblemDetail"`). If another member's doc has
  it, move it into the index with both service names and remove it from that member's doc
  (another write, same read-edit-write rule). If this service breaks a convention the index
  lists, note the difference in this service's doc.

### 7. Write the service doc

- Draft it in a scratch file. Check before writing: `wc -c` at most ~8000 characters (about
  2k tokens; group harder if over), summary at most 1024 characters, no "gap" outside GAP lines,
  every path variable written as `{}`.
- **New**: `createDocument` with scope `PROJECT`, `projectKey`, type `MARKDOWN`, name
  `<service-name> (service map)`, summary, and relations: `PART_OF` the index, `DEPENDS_ON` the
  docs of services it calls and of producers of topics it consumes (only those that exist).
- **Existing**: `getDocument` with `includeContent: true` right before writing. Rebuild
  everything that comes from this repo's code. Keep what other runs added (callers,
  consumers, answers) for endpoints and topics that still exist; drop it for ones that are
  gone. Then `updateDocument` with content and summary. Never supersede or recreate the doc.
- Sync relations: add missing `PART_OF` and `DEPENDS_ON`; `removeDocumentRelation` for
  `DEPENDS_ON` targets this service no longer uses.

### 8. Update neighbours

Collect every set doc that needs an edit, then write each one once. Every neighbour entry
that should name this service gets it (append to the list; close the matching GAP):

- a service you call: its Inbound entry for that endpoint, `Callers:`;
- a producer of a topic you consume: its Outbound `Produces <topic>` line, `Consumers:`
  with what you do with it;
- a consumer of a topic you produce: its Inbound messaging entry, `Producer:`;
- a service that calls you (step 5): its Outbound entry for you, `Target does:`.

If the neighbour's doc has no matching entry, don't add one to it; note the mismatch in your
own entry instead.

Relations: services that call you and consumers of your topics get `DEPENDS_ON` from their
doc to yours if missing (`createDocumentRelation`, source = their doc).
- **GAP search**: one or a few queries of the form
  `GAP "<service-name>" OR GAP "<alias>" OR GAP "<base path>" OR GAP "<topic>" ...` covering
  your name, aliases, resource base paths, topics and gRPC services. Scope `PROJECT`, size 50,
  keep set members, skip your own doc. A hit means the doc has at least one open question
  and mentions one of your terms somewhere; read its GAP lines and answer only those your
  code answers.

For each doc: `getDocument` with content, apply all edits, `updateDocument`. Count the gaps you
closed.

### 9. Report

Follow the session's communication style. In words, no entry ids:

- which docs were created or updated (this service's doc, the set index, and the
  neighbours' docs by service name);
- how many gaps were closed in other services' docs;
- how many stay open in this service's doc;
- a new set key, if you created one, and anything you had to ask or decide.

## Writing rules

- `updateDocument` replaces the whole content and has no version check. A write based on a
  stale copy silently erases what another run added. So only the main agent writes docs, one
  at a time, and each write is `getDocument` (with content) → edit that fresh text →
  `updateDocument`. Don't reuse a copy read earlier in the run.
- Always send the complete content; there is no partial update of content.
- Never `removeDocument` a service or index doc and never supersede one. Their ids are what
  the links point to.
- Write only what you saw in code or in another service's doc. Other docs are context, not
  proof about your own code.
- Don't copy doc content into memories; the docs are the record. A memory is for what the
  template has no place for, such as a pitfall you hit while analyzing.
