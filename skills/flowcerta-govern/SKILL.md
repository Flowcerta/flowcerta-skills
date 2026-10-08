---
name: flowcerta-govern
description: "Governance gate for RPA automations — UiPath (.xaml), Power Automate, Automation Anywhere, Blue Prism. Run `flowcerta context` when starting work in a repository that has workflows; use BEFORE writing or editing any workflow to load the project's policy, and AFTER every edit to gate it. Analysis runs locally; workflow files never leave the machine."
allowed-tools: Bash, Read, Edit, Glob, Grep
---

# Governing the workflows you write

You can produce a working automation faster than a human can review one. This
skill is how you check your own work against the rules the project is actually
held to, before anyone else sees it.

Install is one of:

    npx @flowcerta/cli --version
    dotnet tool install -g flowcerta

## Start here

    flowcerta context

One call for where you are: whether this is a UiPath project, which rules are in
force, whether the machine is signed in and to whom, how stale the organisation's
policy is, whether guardrails bind this repository, and whether UiPath's own `uip`
CLI is on PATH. It ends by naming the command to run next.

Run it once at the start of a session rather than working the rest out with six
separate commands. `--json` if you would rather parse it.

## Before you write

Read `.flowcerta/policy.md`. If it is not there, create it:

    flowcerta policy --output .flowcerta/policy.md

That file lists every rule this project is held to, each with the fix. It is
written for you to hold in context for the session: one line per rule, worst
severity first, so a truncated read still sees what matters. Write workflows that
satisfy it rather than writing first and repairing after.

If the project has a `.flowcerta/rules.json`, the digest already accounts for it:
rules the project switched off are not listed, and rules the project added are.
You do not need to pass anything — `policy` and `gate` both find that file by
walking up from where they are pointed, and both say on stderr when they used it.
The digest's own "Source:" line tells you which rules you are reading.

Two other things about the file. It is generated from the CLI version you have, so
regenerate it if you upgrade. And it is safe to commit — a policy visible in
review is one a human can argue with.

## After every workflow edit

    flowcerta gate <file>

One file per call. If you edited three workflows, gate three times.

| exit | meaning | what you do |
|------|---------|-------------|
| 0 | nothing at or above the threshold | you are done |
| 1 | policy violation | fix it, see below |
| 2 | the tool could not answer | report it and stop |

**Exit 2 is not a violation.** It means a bad path, a bad argument, or a file the
analyzer cannot parse. Do not edit the workflow in response to a 2 — you would be
changing something that may well be fine.

`gate` decides for itself how to judge the file. If the workflow is tracked in git
and modified, it compares against the committed version and reports only what you
introduced. Findings that were already there do not block your change. If there is
no committed version, the whole file is judged, because all of it is new.

Default threshold is `high`. Pass `--fail-on medium` if the project wants more.

The gate applies the same `.flowcerta/rules.json` the digest described, so what
you read and what you are measured against are the same rules. If that file is
malformed, `gate` exits 2 rather than falling back to the defaults — a clean
verdict against the wrong rules is worse than no verdict.

## Say who you are

Pass your own name on every gate:

    flowcerta gate <file> --agent <your-name>

Use the name of the agent you are — `claude-code`, `cursor`, `codex`, `copilot`,
`gemini-cli`, `aider`, `windsurf` — or set `FLOWCERTA_AGENT` once for the session
and leave the flag off. Any other name is fine too; an agent we have never heard
of naming itself is better than one we guess at.

This is the one fact in the whole loop that only you can supply. Flowcerta sees the
workflow and the verdict, but whether an agent or a person wrote the change is
knowable at exactly one moment — this one — and is unrecoverable afterwards. The
platform cannot reconstruct it; UiPath states as a design goal that agent-written
and human-written automations take the same path to production, which is what makes
them indistinguishable later.

Some agents are recognised automatically from the environment, but that is a guess
and it is recorded as one. Naming yourself is a fact, and it is recorded as a fact.

**Do not name yourself as something you are not**, and do not pass `--agent` on
behalf of a human who ran the command. An attribution nobody can trust is worse than
none, because somebody will make a decision with it.

### Where the attribution goes

Nowhere, unless the organisation asked for it. Off by default.

When an organisation does collect it, each verdict sends the agent name and how sure
we are, the project name, the rule ids and severities that breached, the mode, the
threshold, and pass or fail. Optionally the workflow's file name, if the org allows
it.

**Never the workflow, and never a finding's description** — descriptions quote the
file. Never a file path, never a line number. `flowcerta context` states on its
`Reporting:` line whether this is on, before you run a single gate. `--no-report`,
or `FLOWCERTA_NO_TELEMETRY`, declines on this machine whatever the org set.

## On a violation

1. Run `flowcerta explain <ruleId>` for each rule id in the output. The
   explanation says what the rule catches and what to do instead.
2. Fix the workflow.
3. Run `flowcerta gate <file>` again.

**Stop after three attempts.** If it still fails, report the remaining violations
with their rule ids and what `explain` said, and let a human decide. Nothing in
the tool enforces this cap, so it is on you: a rule you have misunderstood will
not start making sense on the fourth try, and by then you have burned an
afternoon of someone's tokens on a loop.

## The rule you must not break

**Never change business logic to satisfy a governance rule.**

If the only way to pass the gate is to alter what the automation does — skip a
step, drop a field, change which system it writes to — stop and report it. A
passing gate on a broken automation is worse than a failing gate on a working one,
because the failing one gets looked at.

There is no `--fix` and no way to suppress a finding from here. You edit your own
draft; the verdict is not yours to change.

## Who wrote this before you

    flowcerta provenance <file>

Before editing a workflow somebody else has been working on, ask what is known
about it: which coding agents have gated it, how often, and how often they failed.
Only the file name is sent, and the file need not exist locally — you can ask
about something you have only seen in a diff.

Several files at once, which is the shape a pull request has:

    flowcerta provenance Main.xaml Billing.xaml --markdown

`--markdown` prints a table meant for a code review. In CI, pipe it wherever
reviews happen:

    flowcerta provenance $(git diff --name-only origin/main...HEAD -- '*.xaml') \
      --markdown | gh pr comment --body-file -

It prints markdown rather than posting anything itself, because Flowcerta has no
business knowing which forge you use.

Read the answer carefully in one respect. **"No verdicts recorded" is not a clean
history.** It means nothing was reported — the file may never have been gated, or
the organisation may not collect provenance at all. It is the absence of evidence,
not evidence of absence, and the command says so rather than showing you zeroes.

The same applies to the unattributed count: it covers CI runs, wrapper scripts and
agents nobody recognises as well as people. It does not mean a person.

## What your organisation will not let you run

    flowcerta guardrails

Some organisations deny or gate specific `uip` commands — a deploy, a shared-asset
delete. `guardrails` shows the rules and the reason for each. `--apply` writes them
into the permission file of every agent that reads one, whichever agent you are:

| agent | file |
|---|---|
| Claude Code | `.claude/settings.json` |
| Cursor CLI | `.cursor/cli.json` |
| Codex CLI | `.codex/rules/flowcerta.rules` |
| Gemini CLI | `~/.gemini/policies/flowcerta.toml` (`--target gemini-cli`) |

Read the output rather than assuming it applied cleanly. The formats do not agree
about what a permission is — Cursor has no "ask first" list, Codex has no glob
syntax — so anything that did not survive translation is printed under **Not
expressible**, per agent. If a rule of yours is listed there, that rule is not
binding your agent and you should treat the command as off-limits yourself.

**It is not a sandbox.** It restricts what an agent will run. Commands a person
types into a terminal are outside it, and the files are editable like any other
file in the repository.

## What this does not do

It reads workflow definition files. It does not run the automation, and it cannot
tell you whether the automation is correct — only whether it breaks a rule that
was written down. A clean gate means you did not violate policy. It does not mean
the automation works.

Signed out, you get the built-in rules. `flowcerta login` adds the organisation's
own rules and policy packs on top, and `flowcerta policy` will then say so.

## If you need to authenticate

Use the environment variable. Nothing is written to disk and it works in a container:

    FLOWCERTA_API_KEY=<key>

**Do not run a bare `flowcerta login`.** There is no browser where you are, so it
exits 2 and tells a human what to do — which is the right outcome, but it is not
something you can resolve yourself. `flowcerta whoami` tells you whether a key is
already in force; it exits 0 either way, so "not signed in" is an answer, not a
failure. Being signed out is fine: the built-in rules and the project's own config
still apply.
