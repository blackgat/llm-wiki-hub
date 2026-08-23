# SETUP.md — build a co-authored wiki hub

**You are an agent. This file is your task.** Follow it top to bottom and you
will have a working hub: a git repository that several agents on several
machines can maintain together, with provenance on every fact and an audit
trail of who wrote what.

You do not need to read `SPEC.md` to execute this. Read it when you want the
reasoning, or when something here does not fit the situation.

Everything below is verified: the merge behaviour, the generator output, and the
conflict resolution were run before this file was written.

---

## 0. Before you touch anything — ask

Five questions. Three have safe defaults you may apply silently if the human is
not available; two you must ask.

| | Question | Default |
| --- | --- | --- |
| **Ask** | Where is the git remote? (URL, or "local only for now") | — |
| **Ask** | What is this hub *about*? One sentence. | — |
| Default | Page types (`type:` values)? | derive from the topic; start with 2–4 |
| Default | Timezone for all timestamps? | `UTC` |
| Default | ASCII-only filenames? | yes |

Two things you must **not** decide alone:

- **Public or private remote.** If the hub will hold anything about a private
  environment, say so and let the human choose.
- **Folder taxonomy.** Ask what the top-level groupings under `wiki/` should be.
  If the human has no opinion, use a flat `wiki/` — it is fine up to a few dozen
  pages and costs nothing to split later.

---

## 1. Scaffold

```sh
mkdir -p <hub> && cd <hub>
git init
mkdir -p wiki raw/captures raw/statements tools
```

Copy from this repository's `templates/`:

| From | To | Then |
| --- | --- | --- |
| `templates/gitattributes` | `<hub>/.gitattributes` | — |
| `templates/gitignore` | `<hub>/.gitignore` | — |
| `templates/AGENTS.md` | `<hub>/AGENTS.md` | **replace every `<PLACEHOLDER>`** |
| `tools/build-index.py` | `<hub>/tools/build-index.py` | — |

If your agent tool auto-loads a differently-named instruction file, create it as
a one-line pointer rather than a second copy:

```sh
echo '@AGENTS.md' > CLAUDE.md      # example: Claude Code
```

Create the two special files:

```sh
printf '# Log\n\nAppend-only. One line per operation. Sort before reading.\n' > LOG.md
python tools/build-index.py
```

**Git does not track empty directories**, so a fresh clone would arrive without
`wiki/` and the generator would fail with "wiki not found". Put a `.gitkeep` in
all three:

```sh
touch wiki/.gitkeep raw/captures/.gitkeep raw/statements/.gitkeep
```

**Do not skip filling in `AGENTS.md`.** A hub whose schema still says
`<LANGUAGE>` and `<TZ>` will produce inconsistent writing from every agent that
reads it.

---

## 2. First commit

```sh
git add -A
git commit -m "Scaffold co-authored wiki hub. [<agent>@<host>]"
```

This is the one time `git add -A` is acceptable — there is nothing else in the
tree. From here on the protocol forbids it.

If a remote was given:

```sh
git remote add origin <URL>
git push -u origin main
```

---

## 3. Verify — do not skip this

Three checks. Each one has caught a real defect.

### 3.1 The generator is deterministic

```sh
python tools/build-index.py
python tools/build-index.py
```

The second run must print `unchanged`. If it rewrites the file, two agents will
generate different indexes from the same tree and conflict endlessly.

### 3.2 `raw/` bytes survive a round trip

```sh
printf 'a\r\nb\r\n' > raw/captures/roundtrip-probe.txt
git add raw/captures/roundtrip-probe.txt && git commit -qm probe
rm raw/captures/roundtrip-probe.txt && git checkout -- raw/captures/roundtrip-probe.txt
od -c raw/captures/roundtrip-probe.txt | head -1
```

You must still see `\r \n`. If it became `\n`, `.gitattributes` is missing
`raw/** -text` and the hub is silently rewriting its own archive. Fix it before
going further, then delete the probe:

```sh
git rm -q raw/captures/roundtrip-probe.txt && git commit -qm "Remove probe."
```

### 3.3 A real concurrent write

This is the check people skip and regret. Simulate two agents:

```sh
cd .. && git clone <hub> hub-b && cd hub-b
# agent B writes
printf -- '---\ntype: note\nstatus: active\nupdated: 2026-01-01\nsummary: from B\n---\nB\n' > wiki/b.md
printf '## [20260101T000200Z] write | b @agent-b/host-b\n' >> LOG.md
python tools/build-index.py
git add wiki/b.md LOG.md INDEX.md && git commit -qm "Add b. [agent-b@host-b]"

cd ../<hub>
# agent A writes a different page, and pushes first
printf -- '---\ntype: note\nstatus: active\nupdated: 2026-01-01\nsummary: from A\n---\nA\n' > wiki/a.md
printf '## [20260101T000100Z] write | a @agent-a/host-a\n' >> LOG.md
python tools/build-index.py
git add wiki/a.md LOG.md INDEX.md && git commit -qm "Add a. [agent-a@host-a]"

cd ../hub-b && git pull --rebase
```

Expected, and what each outcome means:

| Observation | Meaning |
| --- | --- |
| `LOG.md` merges with **no conflict** | `merge=union` is working |
| `LOG.md` line order follows **merge direction, not timestamps** | expected — it may coincidentally look chronological; never rely on it, always `sort` |
| `INDEX.md` **conflicts** | correct — it must not be silently resolved |
| `wiki/a.md` and `wiki/b.md` both present, no conflict | page granularity is working |

Resolve the index the documented way — take a side to clear the conflict state,
then regenerate:

```sh
git checkout --ours INDEX.md
python tools/build-index.py
git add INDEX.md && git rebase --continue
```

Then clean up:

```sh
cd .. && rm -rf hub-b
cd <hub> && git rm -q wiki/a.md && git commit -qm "Remove verification fixture."
python tools/build-index.py && git add INDEX.md && git commit -qm "Rebuild index."
```

> If `INDEX.md` did **not** conflict, check whether someone added
> `INDEX.md merge=ours` to `.gitattributes`. Remove it. Git has no built-in
> `ours` driver, so the line is usually inert — but if `merge.ours.driver` is
> configured it swallows the conflict, and the agent never learns to regenerate.

---

## 4. Wire up the agents

### An agent whose working directory is the hub

Nothing to do. The schema auto-loads, git runs in the right place.

### An agent working elsewhere, writing to the hub as a secondary target

Three things break, and all three have the same shape — the tool assumes the
working directory is the subject:

1. **File access is scoped to the working directory.** Grant access to the hub
   path explicitly (in Claude Code: `--add-dir <hub>`, or `/add-dir` mid-session).
2. **The schema does not auto-load.** Package the protocol as a *user-level*
   skill or command so it travels with the agent instead of with the project.
   Its first instruction must be: read `<hub>/AGENTS.md`.
3. **Git resolves to the wrong repository.** Every command must name the hub:
   `git -C <hub> ...`. Never rely on the current directory.

Allow-list only the `-C <hub>` form in your tool's permission settings. That way
a forgotten `-C` prompts instead of silently committing to the wrong repo.

**The hub's location cannot live inside the hub.** It goes in each agent's own
configuration — an environment variable, the agent's config file, or a pointer
line in the surrounding project's instructions. This is the one piece of shared
state that legitimately sits outside.

### An agent that cannot run git

It can still read and write files. Either give it a version-control tool, or
accept that another writer commits on its behalf — and know that you lose
attribution when that happens.

---

## 5. Report back

Tell the human:

- Where the hub is, and the remote (or that there is none yet)
- Which decisions you defaulted rather than asked
- The result of each verification in step 3
- What remains: filling in `AGENTS.md` placeholders you could not answer, and
  wiring up any agent beyond the one you are running as

Then stop. Do not start writing content into a hub whose schema is still
half-filled.

---

## Ordering rules — why the sequence matters

- Schema before the second writer. Two agents writing under different
  assumptions diverge faster than you can reconcile them.
- Verification before real content. Every check in step 3 is cheap on an empty
  hub and expensive once there is history.
- Unsupervised scheduled agents **last**, after the supervised path is stable.
  An unattended writer with push access multiplies whatever is already wrong.
