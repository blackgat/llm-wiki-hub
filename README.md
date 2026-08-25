# LLM Wiki Hub

A knowledge base that several LLM agents, on several machines, maintain
together — with a source behind every fact and an audit trail of who wrote what.

One git repository is the **hub**. Every agent is an interchangeable client.
Agents never talk to each other; all coordination goes through the hub — which
is where the name comes from, and what separates this from the single-agent
[LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
it derives from.

## Point an agent at this

Give any capable coding agent this line:

> Read https://raw.githubusercontent.com/blackgat/llm-wiki-hub/main/SETUP.md and set up a
> co-authored wiki hub for me.

[`SETUP.md`](SETUP.md) is written as an executable task: it asks you the two
questions that have no safe default, scaffolds the repository, and then runs
three verifications before declaring success. Nothing in it is aspirational —
the merge behaviour, the generator output, and the conflict resolution were all
run before it was written.

Adding a machine to a hub that already exists is a different task, with
different verifications and a real chance of damaging what is already there.
That one is [`JOIN.md`](JOIN.md).

## What you get

```
<hub>/
├─ AGENTS.md         the schema every agent reads before writing
├─ INDEX.md          generated catalogue — the retrieval entry point
├─ LOG.md            append-only record of every operation
├─ wiki/             the knowledge, owned entirely by the agents
├─ raw/              immutable sources: captures/ and statements/
└─ tools/build-index.py
```

## The ideas worth knowing before you start

**Three layers, different owners.** Sources are immutable, the wiki belongs to
the agents, the schema evolves with you. The wiki can be handed over safely
because every *fact* in it is reconstructible from the sources.

**Every fact carries a pointer or a copy.** A pointer names a place any agent
can reach; a copy lives in the hub. Three questions decide which — none of them
asks *which machine*. When unsure, copy: a redundant copy is noise, a missing
one is gone forever.

**Statements are not transcripts.** What a human said is full of references
("this project", "it") that resolve against a conversation nobody will have
tomorrow. The resolution of those references is itself an unrecoverable source,
so it is recorded alongside the words — never instead of them, so that a later
error can be traced to the speaker or to the listener.

**Concurrency is handled by file layout, not locking.** Small pages rarely
collide. The two files everyone touches get explicit treatment: the log
union-merges, and the index is regenerated rather than merged.

Full reasoning, including what this design deliberately does *not* solve:
[`SPEC.md`](SPEC.md).

## Repository contents

| Path | |
| --- | --- |
| [`SETUP.md`](SETUP.md) | agent-executable bootstrap |
| [`JOIN.md`](JOIN.md) | agent-executable path for a machine joining a hub that already exists |
| [`SPEC.md`](SPEC.md) | the architecture and its eight invariants |
| [`templates/`](templates/) | schema, `.gitattributes`, `.gitignore` to copy into a new hub |
| [`tools/build-index.py`](tools/build-index.py) | index generator; no dependencies |

## Credit

The three-layer pattern comes from Andrej Karpathy's
[LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f).
That design assumes one human and one agent working side by side; this one adds
what several writers across several machines need — concurrency rules, an
admission criterion for the source layer, and attribution.

## Licence

[MIT](LICENSE).
