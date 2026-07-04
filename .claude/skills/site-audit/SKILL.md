---
name: site-audit
description: Run a full health-check/audit of the 楓星升等計算器 (maplestar) MapleStory leveling calculator website — covers Firebase rules and config, GitHub repo settings and Pages deployment health, desktop and mobile UI/UX (actually driven in a browser, not just read from source), accessibility, SEO/Search Console readiness, performance, and anything else the project turns out to depend on. Use this whenever the user asks to review, audit, optimize, or generally "look at" the whole site — phrases like "整體網站有沒有優化空間", "review the site", "check the website for issues", "audit firebase/github/UI", "有什麼可以優化的" — even if they don't say the word "audit". Delegates the legwork to a subagent so the main conversation stays uncluttered, then reports back a categorized findings list.
---

# Site Audit

This skill runs a comprehensive, read-only audit of the maplestar project and reports findings back to the user in a categorized list. It exists because this project touches several systems at once (a static site, GitHub Pages deploys, Firestore), and it's easy for issues in any one of them to go unnoticed if nobody looks at the whole picture periodically.

## Project facts (bake these into the subagent brief — it starts with no memory)

- Repo path: `C:\Users\xornv\OneDrive\桌面\maplestar`
- GitHub repo: `xyzzxc00/maplestar` (public), deployed via GitHub Pages at `https://xyzzxc00.github.io/maplestar/`
- Pages deployment: GitHub Actions workflow at `.github/workflows/pages.yml` (uses `concurrency: group: pages, cancel-in-progress: false` specifically to stop overlapping deploys from racing each other and failing — this was a real recurring bug, so re-check it's still intact)
- Firebase project ID: `maplestar-calc`, Firestore is used for a community-submitted EXP-rate database (`exp_records` collection). Rules live in `firestore.rules`, deployed via `firebase deploy --only firestore:rules --project maplestar-calc`
- `gh` CLI and `firebase` CLI are both already installed and authenticated on this machine — use them directly, don't try to re-auth
- Local dev server: a `.claude/launch.json` config named `"static"` runs `npx serve .` on port 5173 — use `preview_start` with name `"static"` to view the site live
- It's a single-file static site (`index.html`, ~1800 lines, inline CSS/JS, no build step) — there is no bundler, no node_modules, no framework
- This is a solo hobbyist side project with low traffic. Findings should be prioritized pragmatically: real bugs and anything that could break the site or leak/corrupt data first, cosmetic nice-to-haves last. Don't apply enterprise-grade paranoia to a low-traffic personal tool.

## How to run the audit

Spawn one subagent (general-purpose type) to do the actual work, so the main conversation doesn't get cluttered with dozens of tool calls. Give it a single, self-contained prompt — it will not have access to this conversation's history, so restate the project facts above plus the checklist below. Run it in the foreground (you need its findings before you can report to the user).

Tell the subagent explicitly:
- **Actually verify things, don't just read code and guess.** For the UI/UX sections this means literally starting the dev server and clicking through the site with the preview tools (this project's established convention is: verify UI by running it, not by reading the HTML and assuming). For Firebase/GitHub sections this means actually running `gh`/`firebase` CLI commands, not assuming based on file contents alone.
- **This audit is read-only/observational.** Do not deploy Firestore rules, do not change GitHub repo settings, do not push commits, do not modify production state. If it spots something that should be fixed, it reports the finding — the user decides whether to act on it. (Local, non-destructive `git diff`/`gh api` GET calls and read-only `firebase` commands are fine and expected.)
- **Go looking for the unknown, not just the checklist.** Part of the point of this audit is catching things nobody remembered to mention. If it notices the project depends on something not listed below (another Firebase product, a GitHub App, an npm dependency, anything), say so.

### Checklist to cover

1. **Firebase**
   - Read `firestore.rules` in the repo and assess it for gaps: missing field validation, unbounded values, anything that would let a client write malformed or abusive data to `exp_records`.
   - There's no way to diff deployed rules against the repo file via CLI directly, so instead run `firebase deploy --only firestore:rules --project maplestar-calc --dry-run` if supported by the installed CLI version, and if not, just note whether the repo's `firestore.rules` last-modified time is newer than the last known deploy (check git log) as a signal they might have drifted — flag it as "verify these match what's live" rather than asserting drift with certainty.
   - Sanity-check `firebase.json` / `.firebaserc` are minimal and correct (should just point `firestore.rules` at the right file and alias the project).
   - Run `firebase projects:list` / `firebase apps:list --project maplestar-calc` to see if other Firebase products (Hosting, Auth, Storage, Functions, Analytics) are enabled but unused, or used but undocumented.
   - If there's any way to check Firestore usage/quota from the CLI, do so and flag anything that looks like spam or abnormal volume in `exp_records`.

2. **GitHub**
   - `gh api repos/xyzzxc00/maplestar/pages` — confirm Pages is healthy and points at the Actions build type (not "legacy" branch deploys — that mode is known to cause overlapping-deploy failures on this repo).
   - `gh run list --repo xyzzxc00/maplestar --limit 15` — check recent workflow runs for failures, especially the Pages deploy workflow. If the most recent run for the current HEAD commit failed, flag that the live site may be stale.
   - `gh api repos/xyzzxc00/maplestar/dependabot/alerts` and secret-scanning alerts (`gh api repos/xyzzxc00/maplestar/secret-scanning/alerts`) — review anything open. Note: a Firebase *web* API key showing up in secret scanning is expected/benign (it's meant to be public); don't flag it as a real risk, but do flag anything else.
   - Repo metadata: description, homepage URL, branch protection — report only if something looks actually wrong or newly missing, not as a recurring nag once it's already set.
   - Confirm `.github/workflows/pages.yml` still has the `concurrency` block intact (this was added specifically to fix a recurring deploy-race bug — if it's been removed or edited, flag it prominently, that's a regression).

3. **UI/UX — desktop**
   - `preview_start` with launch config `"static"`, then actually click through both tabs (⚔ 練等計算 calculator, 👥 社群經驗資料庫 community DB), toggle dark/light theme, and interact with a few inputs.
   - Use `preview_console_logs` (check for JS errors), `preview_network` (check for failed requests), and `preview_screenshot`/`preview_snapshot` to look for visual bugs — contrast issues, elements that don't restyle correctly between themes (there's history of exactly this bug — a `<select>` once stayed dark-themed in light mode — so specifically check select/dropdown elements in both themes), overlapping elements, broken layouts.

4. **UI/UX — mobile**
   - Same as above but with `preview_resize` set to 375x812 (or the "mobile" preset). Re-test both tabs and both themes at this width. Look for horizontal overflow, cut-off text, touch targets that are too small/close together, and whether the timer/EXP-tracker section's reordering on mobile still looks intentional.

5. **Accessibility**
   - Check that `<label>` elements use `for`/`id` binding to their inputs (including any dynamically-created rows in the multi-level calculator, which use template-generated IDs).
   - Check filter/search inputs that only have placeholder text have an `aria-label`.
   - Check `aria-live` regions exist on the result panels so screen readers announce calculation updates.
   - Spot-check color contrast in both themes for body text and muted/secondary text.

6. **SEO / Search Console**
   - State plainly whether you can determine if Google Search Console is connected for this property — you very likely cannot verify this without access to the user's Google account, so say so explicitly instead of guessing either way.
   - Check for `sitemap.xml` and `robots.txt` in the repo root — note if missing (they currently don't exist).
   - Check `<meta name="description">`, Open Graph tags, `<link rel="canonical">`, and whether the canonical URL matches the actual GitHub Pages URL.
   - Note whether structured data (e.g. JSON-LD for a WebApplication/SoftwareApplication) could help since it's a public tool, but treat this as a nice-to-have, not a defect.

7. **Performance**
   - Check script tag placement (should be at the bottom of `<body>`, not blocking `<head>` — this was fixed once already, confirm it's still that way).
   - Check for anything else obviously blocking first render, and note the overall page weight (it's a single ~1800-line HTML file with no build step, so there isn't much to optimize here — don't invent enterprise-scale performance advice for a project this size).

8. **Anything else discovered**
   - Explicitly look around for services/config the checklist above doesn't mention — other CI workflows, other cloud services referenced in the HTML (e.g. other CDN scripts), unexpected files in the repo root, etc.

### Report format

Have the subagent return its findings, and then relay them to the user (translate to the language the user is using in the conversation) as a categorized list matching the 8 sections above. For each finding include: what's wrong or could be better, why it matters (or explicitly note "low priority/cosmetic" if it's minor), and — only if it's a small, safe, local code change — whether you already fixed it or are proposing to. Skip sections entirely in the output if the subagent found nothing worth reporting there, rather than padding the report with "no issues found" filler for every category.

Do not silently apply fixes for anything that touches GitHub settings, Firebase rules deployment, or other production/shared state — those require the user's go-ahead first, consistent with how this project has been managed so far. Local index.html edits are fine to make directly if the user asks for them after seeing the report, but this skill's own job is to report, not to fix.
