# Service map

Applies when the first task in a repo arrives, and before any task that touches another
service.

The `service-map` skill keeps one short doc per service in `mcp-document-service`: purpose,
inbound and outbound interfaces, cross-cutting rules, code and test conventions, and open
questions (GAP lines). Reading it is cheaper than rediscovering the same facts from code,
and it is the only place that records who calls a service or consumes its topics.

If `mcp-document-service` is unavailable, there is no map to read: skip this file and work
from the code. Don't mention missing map docs; the startup warning already covers it.

## The current repo's service doc

Look it up when the first task in a repo arrives, not at startup. At startup there is no
task yet, and the next task may be in another repo or need nothing from the map.

- `recall` matches only the task's words, so it usually won't return the service doc. Look
  it up directly: `searchDocuments` with `projectKey` = repo folder name, scope `PROJECT`,
  query `"service map"`, `includeSummary: true`, `includeRelations: true`. The doc is named
  `<service-name> (service map)`.
- Read the summary. Read the full content (`getDocument` with `includeContent: true`) when
  the task touches inbound or outbound interfaces, messaging, cross-cutting rules (tenant
  isolation, auth, transactions), or code and test conventions.
- Also read the set's index doc, the service doc's `PART_OF` target, for the shared
  conventions. The service doc lists only where this service differs from them, so "Same as
  set." means nothing without the index.
- No doc found: carry on normally. Mention once that the repo isn't mapped yet. Don't run
  `/service-map` or suggest running it (`references/general.md`).

## Cross-service tasks

When a task involves another service (a call, an event, a contract change), read that
service's map doc before its code.

- Follow the current service doc's relations: its `DEPENDS_ON` targets are the services it
  calls and the producers of topics it consumes; `referencedBy` entries of type
  `DEPENDS_ON` are its callers and consumers. The callers and consumers named in its
  Inbound and Outbound sections lead to the same docs.
- Search with the service-map naming rules: query `"<name>" "service map"`, scope
  `PROJECT`, and take the result named exactly `<name> (service map)`. Quote every name and
  path; unquoted, a name that matches nothing falls back to any-word search. Write path
  variables as `{}` (`"/api/v1/orders/{}/cancel"`), because variable names are search
  tokens. A short name doesn't match a longer hyphenated one, so search each alias
  separately.
- Open the other repo only when the doc doesn't answer the question.

## Keeping the map right

You often see that the map is wrong before anyone else does.

- If you verify in code something that answers a GAP line or contradicts a line in a
  service doc, say so in one line. Make the fix only when the user directly tells you to
  (`references/general.md`). Closing a GAP means deleting the GAP line and writing the
  answer into the entry it belongs to, with how you know it:
  `Callers: order-service (Feign client OrderCancelClient).`
- Follow the service-map writing rules. `updateDocument` replaces the whole content and has
  no version check, so a write from a stale copy silently erases what someone else added:
  `getDocument` with content right before writing, edit that fresh text, `updateDocument`
  with the full content. Use the same naming rules: exact names, path variables as `{}`,
  the word "gap" only on GAP lines.
- Never supersede or remove a map doc; other docs link to its id. This is an exception to
  `references/memory.md`, which prefers superseding: map docs are always updated in place,
  including ones another run wrote.
- For anything bigger (a new endpoint, topic or dependency, or a changed purpose), say in
  one line that the doc is out of date. If the user tells you to fix it, run `/service-map`
  on that service rather than editing by hand: a run also updates the neighbours' docs, the
  relations and the set index; a hand edit leaves them out of step.
- Map docs are documents, so tell the user in words when you changed one, without ids, as
  `references/memory.md` says.
- Don't copy service-doc content into memories. The doc is the record; a copy goes stale.

## Sub-agents

Briefs for work that touches services name the relevant service docs by id (ids are fine
between agents) and tell the sub-agent to read them before reading code. Sub-agents don't
write map docs: they report a needed fix back and the main agent makes it, so two agents
never write the same doc at once.

## After a context compaction

Re-read the current repo's service doc along with the other rule files.
