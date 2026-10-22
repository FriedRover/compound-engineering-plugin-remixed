# House style (fork customization)

This file is specific to the `compound-engineering-plugin-remixed` fork. It points `ce-noslop` at the writing-standards repo when that repo is installed.

<!-- INSTALL: The installation agent must set this path to the actual location of the writing-standards clone on this machine. The default below is the location `INSTALL.md` in that repo requires. -->
Writing-standards path: `~/Projects/writing-standards`

## Check first

Read `core.md` in the path above. If the directory or `core.md` does not exist, stop here and use the Register in `SKILL.md` as written.

## What changes

For text an agent writes for the user or for another agent, write under `core.md` and `voice/technical.md` from the path above. This covers plan and brainstorm documents, PR descriptions, review and babysit reports, handoffs, and replies. The technical voice takes the place of the neutral idiom in "Repo or team artifact" and of the "Agent talking to the user" register where they differ.

- The tests in `SKILL.md` stay on as the check after drafting.
- Boundaries in `SKILL.md` and the caller's contract still win. A section template, a required token, or a pinned heading keeps its form.
- The "Actions for you" block in `voice/technical.md` applies to chat replies only. Leave it out of files.
- Text under the user's name keeps the "The user's own writing" register. Do not load `voice/matt-house.md` here.

## Load once

Read the two files once per run, in the top-level agent. A subagent does not load them.

## Optional check

After writing a file longer than a few lines, run `uv run python -m tools.lint <file>` from the writing-standards directory. Report the hits. Fix them only in text you wrote.
