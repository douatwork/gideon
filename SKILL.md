---
name: gideon
description: >-
  Set up, bootstrap, or sync Gideon — the user's personal manual. Use when the
  task is to scaffold Gideon's folders, register an agent runtime with it, or run
  a bidirectional sync. Not for merely explaining what Gideon is.
---

# Gideon

Gideon is a structured manual of the user — their operating context (`core/`),
what they have at their disposal (`stack/`), how their output should look
(`taste/`), and the procedures they want run (`workflows/`). It is plain
Markdown, no framework. Your job in this skill is to be the *runtime* for it:
find it (or build it), plug yourself in, and keep yourself in sync — all from a
single invocation.

This one skill folds together what Gideon's repo splits across
`workflows/agents-init.md` (one-time bootstrap) and `workflows/agent-sync.md`
(ongoing sync), and adds the missing first step: creating the folders if they
aren't there. Run it any time. It is idempotent — it detects which situation
you're in and does only what's needed.

---

## Step 0 — Locate Gideon

You need the absolute path to the Gideon directory before anything else.

1. If the user named a path, use it.
2. Otherwise check, in order: an explicit env var (`GIDEON_DIR`), a path
   recorded in your own root instruction file, then common spots
   (`~/gideon`, `~/lib-me`, `~/kb/2-areas/atlas`, the current repo).
3. A directory **is** Gideon if it contains `AGENTS.md` plus any of the four
   folders `core/ stack/ taste/ workflows/`.
4. If you find nothing and the user gave no path, ask where Gideon should live.
   Default to `~/gideon` only with the user's go-ahead.

Call the resolved path `$GIDEON` for the rest of this skill. Determine the
**repo root** that contains it (`git -C "$GIDEON" rev-parse --show-toplevel`,
or `$GIDEON` itself if it isn't yet a repo).

Pick a stable **agent name** for yourself — a slug identifying this runtime
(`claude-code`, `pi`, `codex`, …). It keys your state file; reuse the same
slug every run.

---

## Step 1 — Git safety (always, before any write)

All changes to Gideon must be recoverable via git.

```bash
git -C "$GIDEON" rev-parse --is-inside-work-tree 2>/dev/null \
  && git -C "$GIDEON" status --porcelain
```

- If `$GIDEON` is **not** in a git repo and you are about to scaffold, you'll
  `git init` during Step A.
- If it **is** a repo and the working tree is dirty, stage everything
  (including untracked files) and commit a snapshot before proceeding:
  `git -C "$GIDEON" add -A && git -C "$GIDEON" commit -m "snapshot: pre-gideon-skill"`.

This snapshot is your rollback point. Never leave Gideon modified-but-uncommitted
when you finish.

---

## Step 2 — Choose the mode

Check three things and route. The modes are cumulative: a fresh setup falls
through all three; a routine run does only Sync.

| Condition | Run |
|---|---|
| The four folders (`core stack taste workflows`) don't all exist under `$GIDEON` | **A. Scaffold**, then B, then C |
| You have no root instruction file, or no `.sync/<agent-name>` state file exists | **B. Bootstrap**, then C |
| Both above are satisfied | **C. Sync** only |

---

## A. Scaffold — create Gideon from nothing

Build the skeleton. Create only what's missing; never overwrite an existing
file here.

1. Create the directory and, if it isn't already version-controlled,
   `git -C "$GIDEON" init`.
2. Create the four folders and a `.sync/` directory:
   `core/ stack/ taste/ workflows/ .sync/`.
3. Write the manifests below (starter stubs — terse, meant to be filled in).
   If a file already exists, leave it.

**`$GIDEON/AGENTS.md`** — the entry point agents read first:

```markdown
# Gideon — Agent Entry Point

Gideon is a structured manual of a person: operating context, tooling, taste,
and workflows. Read it to bootstrap context about the user without being told.

## Reading order
1. This file — orientation
2. `stack/README.md` — environment and tooling; consult before suggesting commands
3. `workflows/README.md` — workflow sources for your native skills
4. `core/README.md` and `taste/README.md` — identity and output conventions, when the task calls for them

Load supporting files only when a manifest points to them.

## Directory map
| Directory | Holds | Answers |
|---|---|---|
| `core/` | Identity, beliefs, heuristics | Who is this person, and why do they decide the way they do? |
| `stack/` | Tools, services, hardware, environments | What do they have, and what can I reach from here? |
| `taste/` | Voice, code style, naming, sample corpus | How should output look and sound? |
| `workflows/` | Procedures agents run as skills | What's the process for the task in front of me? |

## Core behavior
- Treat Gideon as canonical; one durable fact lives in exactly one place.
- Keep runtime-native files (CLAUDE.md, skills) thin and derived — point at Gideon, don't copy it.
- Update Gideon first when a rule changes, then regenerate native files.
- Commit every change to Gideon so it can be rolled back.
```

**`$GIDEON/core/README.md`**
```markdown
# Core
Identity, beliefs, decision heuristics — the *who* and *why*. Suggested files:
`profile.md`, `beliefs.md`, `heuristics.md`. Optional and private by nature;
leave blank and the rest of Gideon still works.
```

**`$GIDEON/stack/README.md`**
```markdown
# Stack
Tool discovery for agents: software, hardware, services, environments available
on this system. Scan before suggesting commands or configs. Suggested files:
`hardware.md`, `environments.md`, `agent-runtime.md`. One file per domain;
list it here.
```

**`$GIDEON/taste/README.md`**
```markdown
# Taste
How output should look and sound: writing voice, code style, naming. Suggested
files: `writing-style.md`, `file-naming.md`, and a `corpus/` of real samples.
Populate `corpus/` first, then distill the style guides from it.
```

**`$GIDEON/workflows/README.md`**
```markdown
# Workflows
Source of truth for recurring procedures. Each non-system file here is a **skill
source**: agents derive a native skill from it. This `gideon` skill is what does
that derivation (see the sync workflow). Add a file per procedure you want agents
to run as a skill.
```

**`$GIDEON/.sync/.gitkeep`** — empty, so the state directory is tracked.

4. Commit the skeleton:
   `git -C "$GIDEON" add -A && git -C "$GIDEON" commit -m "gideon: scaffold skeleton"`.

Then continue into **B. Bootstrap**.

> Note: this skill intentionally does **not** copy `agents-init.md` /
> `agent-sync.md` into `workflows/` — it *is* their replacement. A scaffolded
> Gideon is operated entirely through this skill. The four content folders are
> the user's to fill.

---

## B. Bootstrap — register this runtime

Wire yourself into Gideon so future sessions read it automatically.

1. **Root instruction file.** Create or update your runtime's user-level root
   instruction file so it points at Gideon — its location, the reading order,
   and which files are authoritative vs. derived. Point at Gideon; do not embed
   its content. Use the user-level config, not a project file, so it applies
   everywhere. Typical locations:

   | Runtime | Root file |
   |---|---|
   | Claude Code | `~/.claude/CLAUDE.md` |
   | Codex | `~/.codex/AGENTS.md` |
   | pi | runtime's user instruction file |
   | other | nearest user-level "always-loaded" instruction file |

   Minimal content to add (merge into an existing file; don't clobber):

   ```markdown
   ## Gideon
   The user's personal manual lives at `<$GIDEON>`. At session start, read
   `<$GIDEON>/AGENTS.md` and follow its reading order. Treat Gideon as
   canonical; keep these instructions thin and derived. To bootstrap, sync, or
   set up Gideon, run the `gideon` skill.
   ```

2. **State file path.** Ensure `$GIDEON/.sync/<agent-name>` will exist for your
   slug. It holds the last-synced Gideon commit hash; the Sync step writes it.

3. Continue into **C. Sync** to run the first reconciliation and seed the state
   file.

---

## C. Sync — reconcile both directions

The heart of the skill. Two phases, always in this order.

### Phase 1 — Agent → Gideon (contribute)

Enumerate everything you know that isn't already sourced from a Gideon file:

- Native skills you hold that have no matching file in `$GIDEON/workflows/`.
- Facts in your memory or context — stack details, the user's preferences,
  things the user has told you — not yet reflected in Gideon.

For each item:

1. Pick the right home: `core/`, `stack/`, `taste/`, or `workflows/`.
2. **Read the existing file before writing.** Merge your additions; never
   clobber existing content.
3. Update that file, or create a new one if none fits.

When done, if anything changed, commit:
`git -C "$GIDEON" add -A && git -C "$GIDEON" commit -m "sync: agent contributions from <agent-name>"`.
If Phase 1 produced no changes, skip its commit.

### Phase 2 — Gideon → Agent (derive)

For every file in `$GIDEON/workflows/` **except** any legacy system files
(`agents-init.md`, `agent-sync.md` — they are instructions, not skill sources):

1. Read the workflow file.
2. Produce or overwrite the corresponding native skill in your runtime's format,
   placed in your skills directory:
   - Claude Code / pi: a `SKILL.md` (or single markdown) with `name` +
     `description` frontmatter.
   - Codex: `SKILL.md` plus `agents/openai.yaml`.
3. The skill is **derived** — its body points back at the Gideon workflow as the
   source of truth, it doesn't fork the content.

Phase 2 writes only to your own skills directory, never to Gideon.

### State file

After Phase 2 succeeds, record the current Gideon HEAD and commit it:

```bash
HASH=$(git -C "<repo-root>" log -1 --format=%H -- "$GIDEON")
printf '%s\n' "$HASH" > "$GIDEON/.sync/<agent-name>"
git -C "$GIDEON" add "$GIDEON/.sync/<agent-name>" \
  && git -C "$GIDEON" commit -m "sync: <agent-name> at $HASH"
```

This persists sync state across machines and reinstalls. It is a record, not a
diff cache — every sync is a full reconciliation, so it stays simple and
idempotent.

---

## Safety & invariants

- **Pre-write snapshot first** (Step 1) — your rollback point for any Gideon
  content change.
- **Read before overwrite** — never replace Gideon content you haven't read.
- **Phase 2 is one-way out** — it writes only your skills, never Gideon.
- **Don't advance the state file on failure** — if deriving a skill fails, leave
  the old skill and the old hash in place.
- **Leave Gideon clean** — finish with a committed working tree, never
  modified-but-uncommitted.
- **Scaffold never overwrites** — Step A only creates what's missing.
