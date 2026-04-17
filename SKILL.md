---
name: multiplay
description: Use when pair-coding with shared repo, enabling seamless turn-taking with git handoffs via handoff and takeover triggers. Automatically creates cloudflare tunnels for live dev server sharing.
---

# Multiplay - Pair Programming Handoffs

Automates git handoff workflows for pair-coding sessions. One trigger per handoff/takeover instead of manual git sequences.

**Trigger words:** `handoff` (finish your turn) and `takeover` (start your turn). These are patterns Claude recognizes, not slash commands.

## When to Use

- Two developers taking turns with Claude Code on a shared repo
- User says "handoff", "takeover", "let my partner take over", "I'm taking over"
- User says "share my dev server", "create tunnel", or you detect a dev server starting

**Don't use for:** simultaneous branch work, solo dev, or repos without a shared remote.

## Triggers

**CRITICAL:** These are ATOMIC operations. Execute the entire workflow - never break into manual steps.

Before executing, verify remote exists with `git remote get-url origin`. If no remote, guide user: create repo on GitHub/GitLab, `git remote add origin URL`, grant partner access, `git push -u origin main`.

### handoff

Stages all changes, commits with descriptive message, pushes. Auto-recovers from push rejection.

When user says "handoff":
1. Run `git remote get-url origin` - if fails, tell user to set up remote
2. Run `git add -A`
3. Generate commit message from `git diff --stat` - describe what changed, not "handoff"
4. Run `git commit -m "descriptive message"`
5. Run `git push origin main`
6. If push rejected: run `git pull --rebase origin main` then `git push origin main`
7. If rebase has conflicts: pause for user resolution

**Do NOT:** stage/commit before running handoff (it includes that), pull before handoff (it handles rejection), break into manual steps.

### takeover

Syncs latest changes from partner.

When user says "takeover":
1. Run `git remote get-url origin` - if fails, tell user to set up remote
2. Run `git pull --rebase origin main`
3. Run `git status` to verify clean state
4. If rebase has conflicts: pause for user resolution

## Cloudflare Tunnel (Dev Server Sharing)

Lets your partner view your running dev server live via a public URL.

### When to Create a Tunnel

Auto-detect dev server output patterns:
- `Local: http://localhost:PORT` (Vite)
- `Server running at http://localhost:PORT` (generic)
- `ready - started server on` (Next.js)
- `App running at: http://localhost:PORT` (Vue CLI)
- `Serving HTTP on :: port PORT` (Python http.server)

Or when user says "share my dev server" / "create tunnel".

**For static HTML projects** (no Node/framework): start a simple server first:
```bash
python3 -m http.server 8000
```
Then create the tunnel pointing to that port. No `allowedHosts` config needed.

### Tunnel Creation Procedure

Follow this exact order. Skipping steps causes failures.

**Step 1: Install cloudflared if needed**
```bash
command -v cloudflared || brew install cloudflare/cloudflare/cloudflared
```

**Step 2: Configure dev server to allow cloudflare hosts (framework servers only)**

Framework dev servers (Vite, Webpack) block unknown hosts. Skip this step for simple/static servers (`python3 -m http.server`, `npx serve`, etc.).

- **Vite** (`vite.config.js`): add `server: { allowedHosts: ['.trycloudflare.com'] }`
- **Next.js** (`next.config.js`): no host blocking by default, usually fine
- **Webpack Dev Server**: add `devServer: { allowedHosts: ['.trycloudflare.com'] }`

If you modify config, restart the dev server and parse the new port from output.

**Step 3: Kill existing tunnels and create new one**
```bash
pkill cloudflared 2>/dev/null; sleep 1; rm -f /tmp/cloudflare-tunnel.log
cloudflared tunnel --url http://localhost:$PORT --protocol http2 > /tmp/cloudflare-tunnel.log 2>&1 &
```

**Step 4: Wait and verify (8 seconds, not less)**
```bash
sleep 8
grep -a "Registered tunnel connection" /tmp/cloudflare-tunnel.log
```

**Step 5: Extract and display URL**
```bash
TUNNEL_URL=$(grep -aoE 'https://[a-z0-9-]+\.trycloudflare\.com' /tmp/cloudflare-tunnel.log | head -1)
```

Display the URL prominently to the user. If empty, show `tail -10 /tmp/cloudflare-tunnel.log` for debugging.

**To stop:** `pkill cloudflared`

### Common Tunnel Failures

| Symptom | Cause | Fix |
|---------|-------|-----|
| "This host is not allowed" in browser | Dev server blocks unknown hosts | Add `.trycloudflare.com` to allowedHosts, restart server |
| "Failed to dial a quic connection" / timeout | QUIC blocked by network | Use `--protocol http2` (already in procedure above) |
| Error 1033 / conflicting tunnels | Multiple cloudflared processes | `pkill cloudflared` before creating new tunnel |
| URL empty after wait | Connection still establishing | Increase wait, check for "Registered tunnel connection" in log |
| Tunnel connects but wrong content | Port changed after server restart | Parse actual port from server output, recreate tunnel |
