# Memory — what the employee accumulates

An Ansur employee starts new and uncertain, and gets better at *this* job over
time by writing down what it learns. **Memory is that accumulated experience** —
corrections, confirmed terminology, working query patterns, company-specific
facts — captured from doing the work, not from pretraining.

## How it works

- **Plain files, not a database.** Memory is markdown under `memory/` in the
  bundle repo. **No vector store, no RAG** — the employee reads and writes
  `memory/*.md` with ordinary file tools (`read`, `write`, `edit`, `grep`), the
  same way it touches any file. There is no special "memory tool."
- **It persists across conversations.** At the end of each turn the platform
  diffs the employee's `memory/` (and `skills/`) edits and commits them back to
  the bundle repo over the wire. The next session re-seeds from that repo, so what
  the employee learned on Monday is there on Tuesday — across threads and channels.
- The employee never runs git and never sees the machinery; it just sees a
  `memory/` directory on "a real computer."

> **Memory ≠ the thread event log (TEL).** TEL is the per-conversation history that
> drives a single thread's state. Memory is durable, git-versioned, and shared
> across *all* of the employee's threads. They're different layers.

## The `memory/` convention

```
memory/
├── INDEX.md            # lists what's in memory, so the employee knows what to read
└── domain-notes.md     # the accumulated knowledge
```

`INDEX.md` is the entry point — the employee's prompt tells it "INDEX.md lists
available files." Keep it a short table of `file — what's in it`.

A real `domain-notes.md` (from a production SAP/sales employee) holds entries like:
- *"YBP = foreign currency. When users say 'YBP' they mean the document's own
  foreign currency, not converted to TRY."* (a terminology correction)
- *"Live production orders: join `OWOR."DocEntry" = AWO1."DocEntry"`,
  `OWOR."Status" = 'R'` = released/onaylı."* (a working query pattern)
- *"User requested no follow-up before 18:00."* (a standing preference)

That's the genre: things you only learn by doing the job.

## What you (the author) do

Mostly nothing — the persistence is automatic. Two things you *can* do:

- **Seed it.** Commit starter files into the bundle's `memory/` (and an
  `INDEX.md`) so a brand-new employee begins with known company facts instead of a
  blank slate. This is the single highest-leverage way to make a cold employee
  competent on day one.
- **Curate it.** Memory is git-versioned and editable — you can correct or revert
  a bad note the employee wrote, like any file in the repo.

In the employee's `prompt.md`, tell it *when* to write memory ("when a user
corrects you on terminology, record it in `memory/domain-notes.md`") — the habit
comes from the prompt; the persistence comes from the platform.

## Why it's useful

Many "the agent got it wrong" failures aren't the model being dumb — they're
**missing context**: terminology not captured yet, a company rule the model can't
know. Those don't get fixed by a bigger model; they get fixed by memory growing.
Seeding + a prompt that tells the employee to record corrections is how you shorten
the climb from new-hire to competent.

## Status note

End-to-end memory persistence is **live on a single-node deployment**. Making the
inbound seed fully node-agnostic (true multi-node) is in progress — not something
the bundle author configures.
