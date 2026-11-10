# Developing cc-stretch at ESTO

How to install, develop and test cc-stretch on an ESTO Mac. Source: the Claude Docs page
"cc-stretch: setup and local development guide" (as of 2026-11-05, tested with Claude Code 2.1.317 and
cc-stretch 0.5.4).

## Why it needs this setup

A plain `/plugin install` of cc-stretch loads but does nothing on ESTO's Team plan. Claude Code's
built-in guard (`cc-plugin-sec-default`) skips the `prompt.*` hooks of any plugin a user installs, and
those hooks are how cc-stretch tells Claude to write stretches.

The workaround is to install it as a company plugin: a root-owned folder, enabled through a local
managed settings file. Company plugins run before the guard (`tier prepend` in the debug log), so the
guide reaches Claude.

## Before you start

- A Mac with admin rights (`sudo`).
- Claude Code 2.1.300 or later: `claude --version`, or `/status` in the desktop app.
- `git` and `npm`.
- Nothing to do in the Claude console: `managedSourcesBehavior: "merge"` is already set for ESTO.
- Remove any cc-stretch you installed yourself: `/plugin` → Installed → uninstall.

## Install (about 10 minutes)

1. **Download the plugin into the company folder.**

   ```bash
   sudo mkdir -p /opt/esto/claude-plugins
   sudo git clone https://github.com/pro-vi/cc-stretch /opt/esto/claude-plugins/cc-stretch
   ```

   Check: `.claude-plugin/marketplace.json` exists there and lists the plugin with `"source": "./"`.

2. **Lock the folder** so only an admin can change the code.

   ```bash
   sudo chown -R root:wheel /opt/esto
   sudo chmod -R 755 /opt/esto
   ```

   Check: `ls -l /opt/esto/claude-plugins` shows `root` as owner.

3. **Create the local managed settings file.**

   ```bash
   sudo mkdir -p "/Library/Application Support/ClaudeCode"
   sudo tee "/Library/Application Support/ClaudeCode/managed-settings.json" > /dev/null <<'EOF'
   {
     "extraKnownMarketplaces": {
       "cc-stretch": {
         "source": { "source": "directory", "path": "/opt/esto/claude-plugins/cc-stretch" }
       }
     },
     "enabledPlugins": { "cc-stretch@cc-stretch": true }
   }
   EOF
   ```

4. **Confirm Claude Code sees it.** Run `claude`, then `/status`: setting sources must include
   `Enterprise managed settings (remote + file, merged)`. Then `/exit` and run `claude plugin list`:
   cc-stretch shows `Scope: managed` and `✔ enabled`.

5. **Test in the desktop app.** Quit with Cmd+Q, reopen, start a new local session with a folder
   selected, accept the trust prompt and ask something long ("explain how git rebase works"). The reply
   should be short, with phrases you click to open.

## Development loop

Work in your own clone (`~/dev/cc-stretch`) and copy it into the company folder to try it. Don't edit
`/opt/esto` directly, and don't let a Claude session do it: the folder stays root-owned so nothing
running as you can rewrite code that runs ahead of the guard.

1. **Edit and run the checks** (see `CLAUDE.md`):

   ```bash
   npm test
   ```

   `npm test` runs `claude plugin validate .` and `claude plugin test .`. Tests run in their own
   engine, so the guard does not affect them.

2. **Deploy to the company folder:**

   ```bash
   sudo rsync -a --delete --exclude .git --exclude node_modules --exclude tests ~/dev/cc-stretch/ /opt/esto/claude-plugins/cc-stretch/
   ```

   ```bash
   sudo chown -R root:wheel /opt/esto && sudo chmod -R 755 /opt/esto
   ```

   `tests/` is left out per `CLAUDE.md`: a copy that holds `*.test.tsx` files runs them under
   `claude plugin test`.

3. **Refresh Claude Code's cached copy.** Claude Code runs the plugin from
   `~/.claude/plugins/cache/cc-stretch/cc-stretch/<version>/`, not from `/opt` directly:

   ```bash
   claude plugin marketplace update cc-stretch
   ```

   ```bash
   claude plugin update cc-stretch@cc-stretch --scope managed
   ```

   Then `/reload-plugins` in an open session, or quit (Cmd+Q) and reopen the desktop app.

4. **Prove it in the desktop app, in a fresh session.** The first reply of a new desktop session is
   the one that breaks (see the log below). Open a new session, ask a long question as the very first
   message, and check it holds stretches. A follow-up reply proves nothing.

5. **Re-check capabilities** after any change that adds them:
   `claude plugin validate /opt/esto/claude-plugins/cc-stretch`. New `$.http.fetch`, `$.env.get`,
   `$.settings.read`, `$.fs.write` or `tool.check` entries need an AI Ops review before deploying.
   Note: `8f5b49d` adds a `$.env.get('CLAUDE_CODE_ENTRYPOINT')` call, so it needs that review.

`claude --plugin-dir ~/dev/cc-stretch` (named in `CLAUDE.md` for writing the API types) loads the
plugin as a personal one, so the guard skips its prompt hooks and replies look broken. Use it for
types, never to judge whether stretches work.

## Updating from upstream

Review the changes first, then deploy as above and run `claude plugin validate`. The plugin never
updates itself from `/opt`.

## Troubleshooting

| Symptom | Check | Fix |
| --- | --- | --- |
| First reply in a new desktop session has no stretches, later ones do | Debug log for that session; plugin version in `claude plugin list` | A desktop spare session composed its prompt before the app attached (see log below) |
| Replies are normal-length everywhere | `/status` shows `remote + file, merged` | `remote` only → the local file isn't read: check its path and JSON |
| cc-stretch missing from `claude plugin list` | `claude plugin marketplace list` shows it as `Folder` | Fix the `path` in the settings file |
| Debug log says the `prompt.context` hook was skipped | It loaded as a personal plugin | Uninstall your own copy; don't use `--plugin-dir` |
| Works in Terminal, not in the desktop app | Session is local, has a folder, started after a full restart | Cmd+Q, reopen, new local session with a folder |

To see how it loaded, search the session's debug log in `~/.claude/debug/` for `cc-stretch`. You want
`tier prepend` and `admitted`.

## Rollback

```bash
sudo rm "/Library/Application Support/ClaudeCode/managed-settings.json"
```

```bash
sudo rm -rf /opt/esto
```

```bash
claude plugin marketplace remove cc-stretch
```

Then quit and reopen the desktop app; `claude plugin list` no longer shows cc-stretch.

## Risks and rules

- **Company-level trust.** As a company plugin it runs before the guard and can change Claude's
  instructions and approve tool calls your deny rules would block. Keep `/opt/esto` root-owned.
- **Reviewed capabilities, 0.5.4:** no network, no environment variables, no file writes, no
  `tool.check`. `$.process.run` only opens web links you ctrl- or alt-click (`open` on macOS),
  filtered by `isWebLink`.
- **Early-access API.** The function-hook API changes between Claude Code releases; an update can
  break the plugin.
- **Same cost.** Claude still writes the full reply.
- **Your machine only.** Don't share the settings file or push it through MDM without AI Ops.
- **Long-term fix.** If cc-stretch moves its guide to `prompt.submit`, it works as a normal plugin on
  Team plans and this setup can go.

## Breakage log

### 2026-11-10: first reply of new desktop sessions lost its stretches

Found by scanning every transcript in `~/.claude/projects/` for the first reply over 600 characters
in each session and checking it for `{{…|…}}`. Times UTC (EET = UTC+2).

| When (UTC) | What |
| --- | --- |
| 2026-11-05 15:21 → 2026-11-10 14:53 | Every new desktop session's first long reply holds stretches (0.5.4 from `/opt`, Claude Code 2.1.317–2.1.321) |
| 2026-11-10 14:10 | Upstream 0.5.6 copied into `/opt` and the managed cache refreshed (`cache/…/0.5.6`) |
| 2026-11-10 14:53 | Last desktop session whose first reply worked |
| 2026-11-10 15:05 | First desktop session whose first reply had no stretches; 15:18, 15:40 and 15:49 the same. Follow-ups in those sessions did hold stretches |
| 2026-11-10 15:57 | A session patched `register.tsx` in `/opt` directly; `8f5b49d` commits the same idea in this repo |
| 2026-11-10 20:07 | A session rolled `/opt` back to the cached 0.5.4 with `rsync`; this left `/opt` owned by the user, not root |

What the desktop app log (`~/Library/Logs/Claude/main.log`, local time) and the transcripts add:

- Every desktop session is a pre-started spare ("spare … starting", then "claimed (ready, lead N ms)"),
  the working ones too. Leads were 15–37 s before the break; after it, failures had leads of 2 s, 3 s
  and 158 s. Spare timing alone does not separate working from failing sessions.
- The app quit and relaunched at 16:05 and 16:11 local, on the same app version (4.23118.1) and
  Claude Code 2.1.321. 0.5.6 went into `/opt` and the cache at 16:10 local, in between.
- Failing sessions lacked the guide for the whole session, not just the first reply. Their later
  stretches appear only after the user asked about stretch text and Claude had read the README or
  `register.tsx`, so Claude was copying the syntax.
- The 14:53 UTC "working" session read `README.md` before its first reply, so it may be the same
  copying. The last clean pass is a first reply at 14:03 UTC (16:03 local), before the restart.
- This session, a spare started at 22:08 local one minute after `/opt` was rolled back to 0.5.4,
  has the guide.
- The `prompt.compose` handler is identical in 0.5.4 and 0.5.6 (same `e.surfaces.length === 0`
  gate). If 0.5.6 is the cause, it is not through that handler: suspects are module load in the
  desktop worker (new imports, `$.config.list`, the `open-all`/`close-all` commands). No desktop
  session wrote a debug log, so this is unproven.

Probe, 2026-11-10 20:19 UTC, 0.5.4 deployed: two sessions started in the background by another
session (not opened on screen, first prompt sent at spawn) each answered a long first question with
no stretches. Both versions have the same `e.surfaces.length === 0` gate, and a session nobody is
viewing has no surface when its first prompt is composed, so this probe can't tell 0.5.4 from
0.5.6. It does show the gate drops the guide whenever the first prompt beats the window attaching,
which fits the failing sessions claimed 2–3 s after their spare started.

Hand test, 2026-11-10 20:28 and 20:32 UTC, 0.5.4 deployed: two sessions opened by hand in the app
(spares claimed after 23 s and 11 s) answered long first questions (9 939 and 20 909 characters) with
no stretches. **So 0.5.4 fails too: the version is not the cause, and neither is a short lead.**

Conclusion: the cause is a race in the desktop app's spare sessions, present in every version. As
`docs/BACKLOG.md` (desktop-first-request) records, a spare composes its system prompt about 2 s after
boot, before the app attaches, with `surfaces` `[]`, so the `e.surfaces.length === 0` gate drops
the guide for the session. When the app happens to attach first, the guide lands, which is why some
sessions (this one included) work and most others don't. `8f5b49d` is the fix: with
`CLAUDE_CODE_ENTRYPOINT=claude-desktop` it adds the guide even with no surface.

Fix test, 2026-11-10 20:36 UTC: `8f5b49d` deployed to `/opt` (root-owned again), app restarted. A
session opened by hand in `data-orchestration` (spare claimed after 18 s) answered a long first
question (5 960 characters) with 11 stretches. One session, after four straight failures on the
old code: the fix works so far.

Open:

- Repeat the hand test a few more times over the next days to rule out a lucky race.
- `$.env.get` review by AI Ops for `8f5b49d`.
- Old `tests/` left in `/opt` (rsync `--delete` skips excluded paths):
  `sudo rm -rf /opt/esto/claude-plugins/cc-stretch/tests`.
- Earlier plan, now superseded: open new sessions in the app yourself and ask a
  long first question. A/B test: with 0.5.4 deployed, open three fresh desktop sessions (folder outside this repo) and ask
  a long first question; then deploy 0.5.6 and repeat. Start the app with debug logging so
  `~/.claude/debug/` records the plugin load.
- `8f5b49d` is not deployed and needs the `$.env.get` capability review. It only helps if the cause
  is the surfaces gate.
- Re-lock `/opt/esto` (`sudo chown -R root:wheel /opt/esto`).
