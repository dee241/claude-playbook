<!-- ============================= BANNER ============================= -->
<!-- 👉 Swap this for your own banner gif. ~1000×240 looks best.
     Drop a file at .github/banner.gif in the repo, or paste a Giphy/Tenor URL. -->
<p align="center">
  <img src="https://raw.githubusercontent.com/dee241/claude-playbook/main/.github/banner.gif" alt="Claude Playbook banner" width="1000px" height="240px">
</p>

<h1 align="center">
  📖 Claude Playbook
  <img src="https://em-content.zobj.net/source/animated-noto-color-emoji/356/sparkles_2728.gif" alt="sparkles" width="28px">
</h1>

<p align="center">
  <em>One shared brain for every Claude Code session — referenced everywhere, owned by the team, never copy-pasted.</em>
</p>

<!-- ============================= BADGES ============================= -->
<p align="center">
  <img src="https://img.shields.io/static/v1?label=|&message=CLAUDE%20CODE&color=8B52FF&style=plastic&logo=anthropic&logoColor=white"/>
  <img src="https://img.shields.io/static/v1?label=|&message=MARKDOWN&color=905AFF&style=plastic&logo=markdown&logoColor=white"/>
  <img src="https://img.shields.io/static/v1?label=|&message=MAINTAINED-YES&color=9461FC&style=plastic&logo=git&logoColor=white"/>
  <img src="https://img.shields.io/static/v1?label=|&message=PRs%20WELCOME&color=9F70FE&style=plastic&logo=github&logoColor=white"/>
</p>

<br/>

<!-- ============================= WHAT ============================= -->
<h2 align="center">🟣 What is this?</h2>

This repo is the **single source of truth** for how Claude Code should behave on our projects — conventions, standards, "always do this / never do that" rules, and the context every teammate would otherwise have to re-explain.

Instead of emailing a file around or pasting it into each session, everyone **references this one repo**. You clone it once, point Claude Code at it, and pull when it changes. No more downloading a fresh copy every time. 🎉

<br/>

<!-- ============================= INSIDE ============================= -->
<h2 align="center">🧩 What's inside</h2>

<table bordercolor="9851F7" align="center">
  <tr>
    <th>File</th>
    <th>What it covers</th>
    <th>When it loads</th>
  </tr>
  <tr>
    <td><code>playbook.md</code></td>
    <td>The core playbook — standards, conventions, workflow</td>
    <td>Every session</td>
  </tr>
  <tr>
    <td><code>.claude/rules/*.md</code></td>
    <td>Optional modular rules, split by topic</td>
    <td>Per <code>paths</code> scope</td>
  </tr>
  <tr>
    <td><code>README.md</code></td>
    <td>This page — setup + onboarding (humans only)</td>
    <td>Never (not read by Claude)</td>
  </tr>
</table>

> 💡 Rename / add files to match your real structure. Keep the playbook lean — long files eat context and lower adherence.

<br/>

<!-- ============================= QUICK START ============================= -->
<h2 align="center">🚀 Quick start (one-time setup)</h2>

**1. Clone once to a predictable spot:**

```bash
git clone https://github.com/dee241/claude-playbook.git ~/dev/claude-playbook
```

**2. Point Claude Code at it.** Add one line to your project's `CLAUDE.md` *(or your personal `~/.claude/CLAUDE.md` to load it everywhere)*:

```md
# Team playbook
@~/dev/claude-playbook/playbook.md
```

That's it. Claude Code reads it automatically at the start of every session. The first time, it'll pop an approval dialog for the external import — approve it. ✅

<br/>

<!-- ============================= UPDATING ============================= -->
<h2 align="center">🔄 Keeping it current</h2>

When the playbook changes, grab the latest — no re-downloading, no copy-paste:

```bash
cd ~/dev/claude-playbook && git pull
```

Next session picks up the new version automatically.

<br/>

<!-- ============================= ACCESS ============================= -->
<h2 align="center">👥 Giving the team access</h2>

1. **Host it under the org**, not a personal account → e.g. `kadince/claude-playbook`. That makes it the team's, not one person's.
2. **Grant access via a GitHub team** (e.g. `@kadince/engineers` = read, a smaller group = write).
3. **Add a `CODEOWNERS` file** so edits to the playbook require review. Now "who can change it" is a permission, managed centrally.
4. Each teammate runs the **Quick start** once. Done.

<br/>

<!-- ============================= CONTRIBUTING ============================= -->
<h2 align="center">🛠️ Changing a rule</h2>

<p align="center">
  <img src="https://img.shields.io/static/v1?label=|&message=OPEN%20A%20PR&color=A479FC&style=plastic&logo=github&logoColor=white"/>
  <img src="https://img.shields.io/static/v1?label=|&message=GET%20REVIEW&color=B18AFF&style=plastic&logo=githubactions&logoColor=white"/>
  <img src="https://img.shields.io/static/v1?label=|&message=MERGE&color=BFA0FE&style=plastic&logo=git&logoColor=white"/>
</p>

- Edit through **pull requests** so the playbook evolves on purpose, not by drift.
- 🔒 **Keep secrets out** — API keys, credentials, tokens. This file is shared and version-controlled.

<br/>

<!-- ============================= DETAILS ============================= -->
<details>
<summary><h3 align="center">🤔 More about this playbook</h3></summary>

<br/>

**Why reference instead of copy?** A copied file goes stale the moment someone updates the original. A referenced file is always current after a `git pull`, and there's exactly one place to fix a mistake.

**Where should the import line live?**
- In a **project's** `CLAUDE.md` → loads only for that repo.
- In your **user** `~/.claude/CLAUDE.md` → loads for every project you touch.

**Prefer not to edit each project's `CLAUDE.md`?** You can launch with `claude --add-dir ~/dev/claude-playbook` (with the additional-directories memory env var set) and it'll load the playbook's memory files from that directory instead.

</details>

<br/>

<p align="center">
  <sub>Made with 💜 for cleaner Claude Code sessions.</sub>
</p>
