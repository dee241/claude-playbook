<p align="center">
  <img src="https://raw.githubusercontent.com/dee241/claude-playbook/main/.github/banner.gif" alt="Kadince Import Playbook" width="1000px" height="240px">
</p>

<h1 align="center">
  📖 Kadince Import Playbook
  <img src="https://em-content.zobj.net/source/animated-noto-color-emoji/356/sparkles_2728.gif" alt="sparkles" width="28px">
</h1>

<p align="center">
  <em>The CLAUDE.md Claude Code loads when prepping customer data for a Kadince import.</em>
</p>

<!-- ============================= BADGES ============================= -->
<p align="center">
  <img src="https://img.shields.io/static/v1?label=|&message=CLAUDE%20CODE&color=8B52FF&style=plastic&logo=anthropic&logoColor=white"/>
</p>

<br/>

## 🟣 About

This repo holds one file: `CLAUDE.md`. It's the playbook Claude Code reads before working on a Kadince data import!

<br/>

## 📂 What's in the repo

<table align="center" bordercolor="9851F7">
  <tr>
    <th>File</th>
    <th>Purpose</th>
  </tr>
  <tr>
    <td><code>CLAUDE.md</code></td>
    <td>The full import playbook. The only working file in the repo.</td>
  </tr>
  <tr>
    <td><code>README.md</code></td>
    <td>This page.</td>
  </tr>
</table>

<br/>

## 🚀 Using it in Claude Code

**1. Clone once to a fixed spot:**

```bash
git clone https://github.com/dee241/claude-playbook.git ~/dev/claude-playbook
```

**2. Point your import project at it.** Add this line to that project's `CLAUDE.md`:

```md
@~/dev/claude-playbook/CLAUDE.md
```

Claude Code loads it at the start of each session. The first time, it asks you to approve the import — approve it. ✅

> 💡 You can also copy `CLAUDE.md` into your working folder and Claude Code picks it up automatically, but the import line keeps you on the latest version without re-copying

<br/>

## 🔄 Updating

When the playbook changes, pull the latest:

```bash
cd ~/dev/claude-playbook && git pull
```

The next session uses the updated version.

<br/>

## 👥 Sharing with the team

<p align="center">
  <img src="https://img.shields.io/static/v1?label=|&message=ORG%20REPO&color=A479FC&style=plastic&logo=github&logoColor=white"/>
  <img src="https://img.shields.io/static/v1?label=|&message=TEAM%20ACCESS&color=B18AFF&style=plastic&logo=github&logoColor=white"/>
  <img src="https://img.shields.io/static/v1?label=|&message=CODEOWNERS&color=BFA0FE&style=plastic&logo=git&logoColor=white"/>
</p>

- Host the repo under the organization account, not a personal one.
- Give the team read access through a GitHub team; keep write access to a smaller group.
- Add a `CODEOWNERS` file so edits go through review.

<br/>

## 🔒 Heads up

Keep credentials and any customer data out of this file. It's version-controlled and readable by anyone with repo access.

<br/>

<!-- ============================= DETAILS ============================= -->
<details>
<summary><h3 align="center">🤔 Why reference instead of copy?</h3></summary>

<br/>

A copied file goes stale the moment someone updates the original. A referenced file is always current after a `git pull`, and there's exactly one place to fix a mistake.

- Put the `@` import in a **project's** `CLAUDE.md` → loads only for that repo.
- Put it in your **user** `~/.claude/CLAUDE.md` → loads for every project you work in.

</details>

<br/>

<p align="center">
  <sub>Made with 💜 for cleaner Kadince imports.</sub>
</p>
