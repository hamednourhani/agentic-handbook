---
title: "Case Study: What /compact Actually Does"
tags: [compaction, claude-code, case-study, context-window]
status: draft
---

# Case Study: What /compact Actually Does

Per [Context Window Anatomy](./context-window-anatomy.md), conversation history (Messages) is the
only block that grows unbounded across a session. A compaction command is the lever that controls it
— and it does much less, and much more precisely, than "summarize the chat."

## What gets touched, and what doesn't

- **System prompt, system tools, custom-agent catalog, memory files, skills catalog** — untouched.
  These are fixed blocks reloaded fresh from disk/config every single turn regardless of compaction
  state; compaction was never designed to target them.
- **Conversation history (Messages)** — this is the **only** thing compaction touches. The current,
  long transcript is condensed by the model itself into a structured summary capturing key facts,
  decisions, and current task/file state, discarding the verbatim back-and-forth (large tool payloads,
  full file diffs, repeated exchanges) that no longer needs to ride along word for word. Typically the
  most recent exchanges are kept verbatim, and everything older gets folded into the summary. From that
  point on, every future request's history = `[compact summary] + [messages sent after compacting]` —
  not the original full transcript.

**Concrete illustration**: twenty file edits totaling hundreds of thousands of tokens of transcript
can collapse to a summary of a few thousand tokens — something like "user requested a series of
additions to a working document; all edits applied successfully; current file covers topics A and B"
— plus the last couple of raw exchanges kept verbatim. Every turn after that resends the small
summary, not the full diff history.

**Compaction has its own one-time cost**: the model has to read the whole transcript once to
summarize it (a large input, plus an output) — but it pays for itself by shrinking every subsequent
turn's resend from then on.

**Automatic triggering**: a reserved buffer near the model's context ceiling exists specifically so
the runtime can trigger compaction on its own as a session approaches running out of room, rather than
hard-failing.

## The summary's actual structure

The model-generated summary follows a fixed template, roughly:

- **Primary request and intent** — what the user actually asked for, overall.
- **Key technical concepts** — the important facts/decisions established.
- **Files and code sections** — what was touched, and why each mattered.
- **Errors and fixes** — problems hit and how they were resolved.
- **Problem solving** — how open questions got resolved.
- **All user messages** — kept close to verbatim, in order (preserving actual phrasing/intent, not a
  paraphrase).
- **Pending tasks** — anything still open.
- **Current work** — the exact state at the moment compaction triggered.
- **Optional next step** — what would logically follow.

Separately, compaction also **re-establishes file context** within a size budget: recently-read
*small* files get reinjected in full, since they easily fit back into the fresh summary. A recently-
read *large* file (a long working log, for instance) instead gets demoted to a pointer — "too large to
include, re-read it if needed" — rather than being reinjected verbatim. This is why, immediately after
compacting, small config-like files the assistant had open may come back fully visible again, while a
large document requires an explicit re-read to recover its content.

**Key mechanical takeaway**: compaction isn't only message-summarization — it also tries to restore
previously-read file contents into the fresh context, but only within a size budget. Small files get
reinjected whole; large ones get demoted to a path-only pointer.

## The transcript on disk is never touched destructively

1. **The full raw transcript stays on disk, unchanged.** Nothing about earlier turns is deleted or
   rewritten out of the underlying session log.
2. **Compaction appends one new entry** — the structured summary — to that same append-only log.
   Additive, not subtractive.
3. **Only the reconstruction rule for the live conversation changes going forward.** Instead of
   rebuilding the resent conversation from every raw turn since the session started, the runtime now
   rebuilds it from `[system prompt + the newest summary entry + turns after the compaction point]`.
   Old raw turns still physically exist in the underlying log; they're simply no longer resent to the
   model on future turns.

**One-line version**: the underlying transcript is permanent and append-only, never pruned by
compaction. The live conversation sent to the model is a *reconstruction* from that log; compaction
changes the reconstruction rule (summary-forward instead of everything-forward), not the log itself.

## The compaction marker

The summary entry a compaction appends isn't just text — it's tagged as a special marker, distinct
from ordinary user/assistant/tool-result entries.

1. On every future turn, the runtime doesn't replay the underlying log from the very first line — it
   finds the *most recent* compaction marker and rebuilds the live conversation as `[system prompt +
   that marker's summary + everything logged after it]`.
2. Everything before the marker still physically sits in the log (recoverable for debugging or an
   explicit full-history view) but is dead weight for building the next live request.
3. A second, later compaction appends a new marker, which becomes the new start-from-here point; the
   prior marker becomes just more skipped history.

The marker is what turns "append-only log" into "resumable log" — it tells the runtime exactly where
the live conversation actually begins, without deleting anything that came before it.
