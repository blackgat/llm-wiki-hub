# JOIN.md — attach a machine to a hub that already exists

**You are an agent. This file is your task.** It is the counterpart to
[`SETUP.md`](SETUP.md): that one builds a hub, this one adds a writer to a hub
someone else already built.

Pick by what is at the remote. If it already contains `AGENTS.md`, you are
here. Do not run `SETUP.md` against it — every step of it assumes an empty
tree, and several of them are destructive once there is history.

The sequence below was executed once end to end, against a populated hub, from
a second machine.

---

## 0. Before you touch anything

One question, not five:

| | Question | Default |
| --- | --- | --- |
| **Ask** | Where should the local clone live on this machine? | — |

**Everything `SETUP.md` asks, this hub has already answered.** Folder taxonomy,
page types, prose language, timezone, filename rules — they are in `AGENTS.md`,
they are settled, and other machines have already written under them. Read
them. Do not re-ask them, and do not re-decide them because a different shape
would have been better. A joining writer that reorganises what it joins is the
single most expensive mistake available here.

---

## 1. Clone and read

```sh
git clone <remote> <hub>
```

Then read, in this order: `AGENTS.md` (the schema — all of it), `INDEX.md`,
and the tail of `LOG.md`.

**Do not scaffold.** No `git init`, no copying from `templates/`, no filling in
placeholders. `SETUP.md` §1 and §2 do not apply to you — with one exception,
noted in §3 below.

> If `AGENTS.md` still contains `<PLACEHOLDER>` text, that hub was never
> finished. Stop and say so. Writing into a half-filled schema is how two
> machines end up with two different conventions.

---

## 2. Verify

Three checks, in this order. They overlap with
[`SETUP.md` §3](SETUP.md#3-verify--do-not-skip-this) but they are not the same
checks, because you are not testing the same thing. The hub was verified when
it was built. **You are verifying this machine against it.**

> **`SETUP.md` §3 is destructive on a populated hub.** As written, it commits a
> probe file and fixture pages into the hub you are standing in. That is fine
> on an empty tree and unacceptable once there is history — and following it
> literally here is exactly the accident this file exists to prevent.
> Everything below that needs a commit runs in **throwaway clones**, and
> nothing in §2 is ever pushed.

### 2.1 Your generator agrees with the last writer's

Run the generator the hub ships in `tools/` — once, immediately after cloning
and **before writing anything**. It must report `unchanged`.

If it rewrites `INDEX.md` instead, your machine and the previous writer's
disagree about what the same tree should produce, and from here every commit
you make reverts theirs and theirs reverts yours.

Diagnose it before you write. It is your environment, and solving it is your
job. **Do not commit the rewritten index as "just a rebuild"** — that buries
the disagreement instead of fixing it, and the next writer inherits it.

This check cannot exist in `SETUP.md`: it needs a second machine. §3.1 there
proves the generator is deterministic *on one machine*. What a multi-writer hub
depends on is that it is deterministic *across* them, and the first moment that
becomes testable is now.

### 2.2 You can write, not only read

Prove a push works before you believe you have joined:

```sh
git push --dry-run origin <branch>
```

It authenticates against the remote and changes nothing. A clone that pulls but
cannot push looks fully wired up and is half dead: the agent follows every step
of the write protocol, appends to `LOG.md`, commits — and fails at the last
one, having already recorded that it did the work.

How credentials work on this machine is this machine's problem; solve it there.
Three properties are not negotiable, whatever you use:

- It works **non-interactively.** An agent has nobody to answer a prompt. A
  credential store that unlocks only for an interactive terminal is not
  configured, however well it works when you test it by hand.
- **The secret does not enter the hub.** Not `.git/config`, not the remote URL,
  not the working tree. Git history is permanent; a token committed once is
  disclosed even after the file is deleted.
- It is **scoped to this repository** ([`SPEC.md` §12](SPEC.md#12-adoption-order)).
  Writers are added one machine at a time precisely so that one machine's
  credential can be revoked without touching the others.

### 2.3 Your git honours the hub's `.gitattributes`

`.gitattributes` is applied by the *client*, so the two behaviours the hub
depends on are properties of your git as much as of the repository — and a new
client can fail checks that passed when the hub was built.

Both need commits, so both run in throwaway clones:

```sh
git clone <hub> /tmp/hub-a          # a copy to wreck — NOT the clone you write from
git clone /tmp/hub-a /tmp/hub-b     # the second writer
```

In `/tmp/hub-a`, run the byte round-trip of `SETUP.md` §3.2: a file with CRLF
committed into `raw/`, removed, and checked out must come back with its `\r`
intact. If it comes back `\n`, your client is normalising line endings and the
hub is silently rewriting its own archive.

Then run [`SETUP.md` §3.3](SETUP.md#33-a-real-concurrent-write) with `/tmp/hub-a`
substituted everywhere it says `<hub>`. `LOG.md` must union-merge without
conflict, `INDEX.md` must conflict, and the two pages must both survive.

Delete both clones. **Never run `git push` during this check** — `/tmp/hub-a`'s
origin is your working clone, and a fixture that reaches it will reach the hub
on your first real write.

---

## 3. Wire the agent up

[`SETUP.md` §4](SETUP.md#4-wire-up-the-agents) covers this and is unchanged for
you: working directory, file-access scope, `git -C <hub>`, and — if a human
will open the hub in an editor — the editor rules, which are the dangerous
part.

Four things are specific to joining:

- **The hub's location goes in this machine's configuration, never in the
  hub.** It is the one piece of shared state that legitimately sits outside.
- **If the hub already has a page about this machine, write as that name.** A
  hub that documents infrastructure often documents the machine you are joining
  from. Reuse its identifier in log lines and commit trailers; a second name
  for a machine the hub already knows splits its own audit trail.
- **If your tool auto-loads an instruction file of its own, check the hub has
  one.** The schema lives in `AGENTS.md`; a hub built by a different agent may
  carry nothing your tool reads at launch. Adding the one-line pointer
  ([`SETUP.md` §1](SETUP.md#1-scaffold) gives the form) is not scaffolding — it
  admits a client, it does not change a convention.
- **Editor and plugin configuration does not travel.** Whatever the schema says
  was configured — auto-commit disabled, an exclusion list, a sync mode — was
  applied on a different machine, and the file holding it is almost certainly
  gitignored on purpose. Apply it here *before* an editor opens the hub, not
  after.

---

## 4. Make the first write a small one

Your first write is the real test of §2. Follow the hub's write protocol
exactly as `AGENTS.md` states it, on something small and genuinely useful.

Do not open with a bulk import. A joining machine that gets attribution,
timestamps, or index regeneration subtly wrong should find that out in a change
that takes one commit to correct.

---

## 5. Report back

- Where the clone is, and the identity you will write as
- The result of each check in §2
- Anything you defaulted rather than asked
- **Anything in the schema you believe is wrong or stale.** Say it; do not fix
  it silently on your way in. A joining writer has fresh eyes and no context —
  that combination finds real errors and invents imaginary ones at about the
  same rate, and the people who have lived with the hub can tell which is
  which. Correcting it is a normal write, made *after* you have joined, through
  the write protocol, with a source.

---

## The short version

| Do not | Because |
| --- | --- |
| Re-ask `SETUP.md`'s design questions | They are answered, and already written under |
| Scaffold, or fill in templates | The hub exists; you are the client |
| Run `SETUP.md` §3 against the hub | It commits probes and fixtures into shared history |
| Commit an index rebuild you cannot explain | It buries a cross-machine disagreement |
| Let a credential reach the repository | Git history is permanent |
| Bulk-import on day one | Errors compound before anyone has reviewed one |
