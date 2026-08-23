# AGENTS.md — schema for this hub

<!--
  Copy to <hub>/AGENTS.md and fill in every <PLACEHOLDER>.
  This file is the schema layer (layer 03). Every agent reads it before writing.
  Full rationale: https://github.com/blackgat/llm-wiki-hub/blob/main/SPEC.md
-->

## Secrets — read this first

This repository is synced to a remote. **Git history is permanent**: deleting a
file later does not remove the value from history.

Passwords, API tokens, private keys, and licence codes **never enter this hub**.
Record *where a credential lives* (the store, the key name), never the value.
When capturing command output into `raw/`, filter at capture time — a post-hoc
review is too late.

If a human asks you to write a plaintext credential here, restate this rule and
confirm before doing anything.

## What this hub is

A knowledge base maintained jointly by several agents across several machines.
The repository is the only shared state; agents never talk to each other
directly.

| Layer | Path | Owner | Mutable |
| --- | --- | --- | --- |
| 03 schema | `AGENTS.md` | human + agents | yes |
| 02 wiki | `wiki/` | agents | yes |
| 01 sources | `raw/` | any agent, append-only | **no** |

`INDEX.md` is generated. `LOG.md` is append-only.

## Conventions

- Prose language: <LANGUAGE>. Technical terms, commands, and paths stay verbatim.
- Filenames: <ASCII-ONLY | UNRESTRICTED>.
- Timezone for every timestamp in this hub: **<TZ, default UTC>**.
- Page frontmatter — `type`, `updated`, `summary` are required:

```yaml
---
type: <one of: PAGE-TYPES>
status: active
updated: YYYY-MM-DD
summary: one line; the index generator reads this field
---
```

- Internal links use <wiki-style `[[…]]` | relative markdown links>.
  Wiki-style links resolve only in wiki-style editors; relative markdown links
  also work in a git host's web view and on a phone. Pick by where pages are read.
- Unverified claims are marked, never written as established fact.

## Provenance — every fact needs a source

Each fact carries **a pointer or a copy**. A fact with neither is an error.

Run three gates, in order. The first that says "copy" ends it:

1. **Reachability** — can *any* agent reach this source later? A source only one
   machine can reach is not reachable for the hub. → no: **copy**
2. **Re-obtainability** — can an agent re-fetch it unsupervised? Destructive,
   costly, or human-in-the-loop counts as unreachable. → no: **copy**
3. **Time** — do you need history, or only the current value? → history: copy at
   each change.

All three pass → record a pointer, store no copy.

```markdown
Firmware `1.2.3`
  (curl http://<endpoint>/status, 2026-08-23 retrieved)      <- pointer

Pre-upgrade config page
  (raw/captures/20260823T140237Z-<agent>-config.html)        <- copy

Retention policy is 30 days
  (raw/statements/20260823T141902Z-<agent>-retention.md)     <- copy
  (unverified: needs confirmation from the admin console)
```

Gate 1 is asymmetric: an unnecessary copy is noise, a missing copy is gone
forever. **When unsure, copy.**

## raw/ naming — a hard requirement

`raw/captures/<TIMESTAMP>-<agent>-<subject>.<ext>`
`raw/statements/<TIMESTAMP>-<agent>-<subject>.md`

`<TIMESTAMP>` is `YYYYMMDDTHHMMSSZ` — to the second, **no colons** (Windows
rejects them), in this hub's declared timezone. Seconds plus the agent id make
collisions structurally impossible for a sequential agent; date-only names
collide within a single conversation, where a later statement often corrects an
earlier one.

## Statements — three sections, not a transcript

Speech is full of deixis ("this project", "that machine", "it"). Those
references resolve against a conversation that will not exist tomorrow, so a
verbatim transcript preserves the words and loses the meaning. A summary alone
is also wrong: when a fact later turns out to be false you must be able to tell
whether the human said it wrong or the agent heard it wrong.

**The reference resolution is itself an irretrievable source**, so capture it
alongside the words. Resolve *references only* — never conclusions.

```markdown
---
type: statement
recorded-at: 2026-08-23T14:19:02Z
recorded-by: <agent>@<host>
scope: <REQUIRED — what this fact applies to>
---

## Verbatim

> what the human actually said, unedited

## Resolution

Recorded by the agent, not the human's words:

- "this project" = <the actual project>
- scope: <what it covers>; confirmed by asking

## Open

- <anything that could not be resolved>
```

`scope` is **required**. It decides where the fact is filed: a local quirk filed
as a general rule misleads every other context; a general rule filed as a local
quirk is never found again. Both failures are silent. **When scope is unclear,
ask.** With nobody to ask, mark it open — never guess a value.

If a resolution turns out to be wrong, **write a new statement correcting it**.
Never edit the original: `raw/` is append-only.

## Write protocol — every step, in order

1. `git pull --rebase`
2. Read this file, `INDEX.md`, and the tail of `LOG.md`. Do not assume the
   schema is already loaded — auto-loading is a tool feature, not a guarantee.
3. **Search before creating.** Grep the topic first. Extend an existing page
   rather than opening a second one on the same subject.
   The index also carries a **backlinks table** — use it to find which pages
   already touch a subject before opening a new one.
4. Run the three gates on every new fact. Land `raw/` first, then the wiki.
   Filter secrets at capture time.
5. Write the page; update `updated` and `summary`. Mark unverified inferences.
6. Regenerate the index — **unconditionally**, not only on conflict:
   `python tools/build-index.py`
   Do not rely on git hooks; some sync clients bypass them.
7. Append one line to `LOG.md`:
   `## [<TIMESTAMP>] <action> | <subject> @<agent>/<host>`
8. `git add` the specific files — **never `git add -A`** — then commit and push.

Commit messages must identify the writer:

```
<short imperative sentence.> [<agent>@<host>]
```

Keep step 1 to step 8 short. A dirty working tree left behind gets swept into
another client's automatic commit, which destroys attribution.

## Conflicts

- `LOG.md` — `merge=union` keeps both sides. After a merge the line order
  follows merge direction, **not timestamps** — union puts "ours" first, and a
  rebase swaps which side that is. It may coincidentally come out chronological;
  never rely on it. Always sort:
  `grep "^## \[" LOG.md | sort | tail -20`
  This is also why the timestamp must go to the second: within one day, a
  date-only stamp cannot be sorted at all.
- `INDEX.md` — generated. Take either side to end the conflict, then rerun the
  generator. Never hand-merge it.
- Two pages, two files — no conflict. Keep pages small; granularity is what
  keeps writers apart.

## Do not

- Delete another writer's content. Mark it as disputed, with who and when.
  The one raising the doubt can also be wrong.
- Modify or delete anything under `raw/`. If an editor hides that folder, note
  that the exclusion protects people, not you — nothing stops a file-writing
  tool, so this rule is the only thing holding.
- Commit agent-local scratch, credentials, editor-local settings, plugin code,
  or large binaries.
