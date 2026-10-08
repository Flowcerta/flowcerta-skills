# flowcerta-skills

Agent skills that teach a coding agent to check the RPA automations it writes.

```
npx skills add Flowcerta/flowcerta-skills
```

Works with any agent the [skills](https://github.com/vercel-labs/skills) CLI supports:
Claude Code, Cursor, Codex, Gemini CLI, Copilot, Windsurf and the rest.

## flowcerta-govern

An agent can write a UiPath workflow in seconds. Nothing in that loop knows your
organisation forbids hardcoded credentials until somebody opens the pull request
two days later.

The skill is three moves. Read the project's policy before writing anything, gate
each workflow after editing it, and run `explain` on whatever failed. It needs the
CLI on the machine:

```
npx @flowcerta/cli --version        # no install, per-platform binary
dotnet tool install -g flowcerta    # needs .NET 8, much smaller
```

Analysis runs locally. Workflow files never leave the machine, and nothing is
uploaded unless your organisation turned on provenance reporting, which the skill
tells the agent before it runs a single gate.

UiPath (`.xaml`), Power Automate, Automation Anywhere and Blue Prism.

## Why a gate and not a reviewer

UiPath's own agent skills include `uipath-review`, which runs Workflow Analyzer
from inside the agent and reports what it found. We measured it on a real project:
29 findings, exit code 0. An agent branching on that cannot tell a clean workflow
from one that breaks every rule the organisation has.

`flowcerta gate` exits 1. An agent and a CI job both already know what to do with
that.

Two decisions carry most of the weight.

Exit 1 and exit 2 mean different things and are never collapsed. 1 is "your
automation broke a rule". 2 is "I could not answer": a bad path, a bad argument, a
file that will not parse. An agent that read the second as the first would start
editing a workflow whose only problem was a typo.

The gate judges the change, not the file. A tracked, modified workflow is compared
against its committed version, so an agent is not blocked by debt it inherited from
whoever wrote the file first. Untracked files are judged whole, because all of it
is new.

## What the agent cannot do from here

There is no fixer and no way to suppress a finding. The agent edits its own draft
and the verdict stays with the tool.

The skill carries one hard rail: never change business logic to satisfy a
governance rule. If the only way to pass is to alter what the automation does, the
agent is told to stop and report it. A passing gate on a broken automation is worse
than a failing gate on a working one, because the failing one gets looked at.

It is also told to stop after three attempts at the same violation. A rule it has
misunderstood will not start making sense on the fourth try.

## This repo is a mirror

The skill is written and tested in the Flowcerta platform repo and published here
on each CLI release, by a workflow rather than by hand. Tags here match the CLI
version the skill was published with, so `cli-v0.1.2` on npm and `v0.1.2` here
describe the same thing.

A pull request against this repo would be overwritten by the next release. Open an
issue instead and it gets fixed at the source.

Apache-2.0. https://flowcerta.com/agents/
