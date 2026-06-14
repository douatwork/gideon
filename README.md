# Gideon — single cross-agent skill

This folder is one skill that does everything operational in Gideon from a
single invocation:

- **Scaffolds** the `core/ stack/ taste/ workflows/` folders (plus `AGENTS.md`,
  `.sync/`, and a git repo) when Gideon doesn't exist yet.
- **Bootstraps** a new agent runtime — writes the user-level root instruction
  file that points at Gideon and registers a per-agent state file.
- **Syncs** bidirectionally — contributes what the agent knows back into Gideon,
  then derives native skills from Gideon's workflow files and records the synced
  commit.

It collapses Gideon's three otherwise-separate concerns (`workflows/agents-init.md`,
`workflows/agent-sync.md`, and the previously-manual folder setup) into one
idempotent, mode-detecting skill. Run it any time; it does only what's needed.

## Why a single skill

Gideon's repo keeps `agents-init.md` and `agent-sync.md` as *system instructions*
that each runtime reads ad hoc and never turns into a skill. That works, but it
leaves the operator-facing entry points scattered and requires the user to know
which document to point an agent at. Folding them into one installable skill
gives every runtime the same first-class verb — invoke `gideon` and it figures
out whether to scaffold, bootstrap, or sync. The cost is one derived artifact per
runtime; the payoff is a uniform, discoverable entry point.

## Installing it per runtime

`SKILL.md` is written runtime-agnostically (`name` + `description` frontmatter,
instruction-only body). Drop it into each runtime's skills directory:

| Runtime | Where it goes | Notes |
|---|---|---|
| **Claude Code** | `~/.claude/skills/gideon/SKILL.md` | Frontmatter `name`/`description` is enough. |
| **pi** | runtime skills dir, `gideon/SKILL.md` | Same `SKILL.md` frontmatter contract. |
| **Codex** | `gideon/SKILL.md` **+** `gideon/agents/openai.yaml` | Codex needs the extra YAML; mirror `name`/`description` into it. |
| **other** | the runtime's skills/commands dir | Any runtime that loads a markdown skill with a name + description trigger works as-is. |

Bootstrapping is a chicken-and-egg only once: copy `SKILL.md` into the runtime,
then invoke it (e.g. *"run the gideon skill"*). From there it writes the root
instruction file, so later sessions discover Gideon on their own and you just say
*"sync with Gideon"* when things change.

## Relationship to the canonical workflow files

This skill is a faithful superset of `public/workflows/agents-init.md` and
`public/workflows/agent-sync.md` — same git-safety discipline, same bidirectional
phase order, same state-file mechanics — plus the scaffold step those files
assume has already happened. If the upstream sync/init contract changes, update
`SKILL.md` to match; it is the derived artifact and Gideon's workflow files remain
the source of truth.
