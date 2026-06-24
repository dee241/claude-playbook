<!-- Banner (optional). Add a file at .github/banner.gif in the repo, or paste any image/gif URL.
     Delete these two lines if you don't want a banner. -->
<p align="center">
  <img src="https://raw.githubusercontent.com/dee241/claude-playbook/main/.github/banner.gif" alt="Kadince Import Playbook" width="1000px" height="240px">
</p>

<h1 align="center">Kadince Import Playbook</h1>

<p align="center">
  The CLAUDE.md that Claude Code loads when preparing customer data for a Kadince import.
</p>

<p align="center">
  <img src="https://img.shields.io/static/v1?label=|&message=CLAUDE%20CODE&color=8B52FF&style=plastic&logo=anthropic&logoColor=white"/>
  <img src="https://img.shields.io/static/v1?label=|&message=MARKDOWN&color=905AFF&style=plastic&logo=markdown&logoColor=white"/>
  <img src="https://img.shields.io/static/v1?label=|&message=VERSION%20CONTROLLED&color=9461FC&style=plastic&logo=git&logoColor=white"/>
</p>

## About

This repo holds one file: `CLAUDE.md`. It's the playbook Claude Code reads before working on a Kadince data import — the workflow, the known gotchas, and the decisions carried over from past imports. Keeping it in one place means everyone runs the same process, and an update lands for the whole team at once instead of being copied around.

## What's in the repo

| File | Purpose |
| --- | --- |
| `CLAUDE.md` | The full import playbook. This is the only working file in the repo. |
| `README.md` | This page. |

## Using it in Claude Code

1. Clone the repo once to a fixed location:

```bash
   git clone https://github.com/dee241/claude-playbook.git ~/dev/claude-playbook
```

2. Reference it from the project you're importing in. Add this line to that project's `CLAUDE.md`:

```md
   @~/dev/claude-playbook/CLAUDE.md
```

   Claude Code loads it at the start of each session. The first time, it asks you to approve the import — approve it.

You can instead copy `CLAUDE.md` to the root of your working folder and Claude Code picks it up automatically, but the import line is what keeps you on the latest version without re-copying.

## Updating

When the playbook changes, pull the latest:

```bash
cd ~/dev/claude-playbook && git pull
```

The next session uses the updated version.

## Sharing with the team

- Host the repo under the organization account rather than a personal one.
- Give the team read access through a GitHub team; keep write access to a smaller group.
- Add a `CODEOWNERS` file so edits go through review.

## Note

Keep credentials and any customer data out of this file. It is version-controlled and readable by anyone with access to the repo.
