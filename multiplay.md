---
name: multiplay
description: Use when pair-coding with shared repo, enabling seamless turn-taking with git handoffs via handoff and takeover triggers. Automatically creates cloudflare tunnels for live dev server sharing.
---

# Multiplay - Pair Programming Handoffs

## Overview

Automates git handoff workflows for pair-coding sessions where two developers take turns using Claude Code on the same codebase.

**Core principle:** One trigger per handoff/takeover instead of manual git add, commit, push, pull sequences.

**Trigger words:** `handoff` (finish your turn) and `takeover` (start your turn). These are NOT slash commands - they're patterns that Claude recognizes to execute the appropriate git workflow.

## When to Use

- Two developers taking turns with Claude Code
- Shared git repository both can push/pull
- User says "handoff", "takeover", "let my partner take over", or "I'm taking over"
- User says "share my dev server", "create tunnel", or when you detect a dev server starting

**Auto-detect dev server start:**
When you see output containing patterns like:
- `Local: http://localhost:PORT` (Vite)
- `Server running at http://localhost:PORT` (generic)
- `ready - started server on` (Next.js)
- `Compiled successfully!` followed by a localhost URL (Create React App)
- `App running at: http://localhost:PORT` (Vue CLI)

**Immediately offer to create a shareable link using cloudflared**

**Don't use for:**
- Simultaneous work on different branches (use normal git workflow)
- Solo development (unnecessary overhead)
- Repositories without shared remote access

## Live Testing Your Partner's Changes

**Important:** To test your partner's changes before taking over, you DON'T need to pull their code. Instead, connect to their running dev server!

**How it works:**
1. Anders starts coding and runs their dev server (e.g., `npm run dev`)
2. Claude automatically detects the dev server and offers to create a shareable link
3. Anders shares the cloudflare link with Bob
4. Bob opens the URL and sees Anders' changes live with hot reload
5. When Anders runs `handoff`, Bob can then run `takeover` to get the code

### Auto-Sharing with Cloudflare Tunnel

**CRITICAL:** Tunnel creation requires dev server configuration changes. Follow the complete procedure below.

**When Claude detects a dev server starting**, follow this sequence:
1. Check if `cloudflared` is installed (if not, install it)
2. **Configure dev server to allow cloudflare hosts** (see dev server configs below)
3. Restart dev server for config to take effect
4. Create tunnel with HTTP/2 protocol (more reliable than default QUIC)
5. Verify tunnel connection established
6. Display the shareable URL

**Manual tunnel creation:**
If you need to create a tunnel manually, say "share my dev server" or "create tunnel"

**Dev Server Configuration (REQUIRED FIRST STEP)**

Most dev servers block unknown hosts by default. Configure BEFORE creating tunnel:

**Vite (vite.config.js):**
```javascript
export default defineConfig({
  // ... existing config
  server: {
    allowedHosts: ['.trycloudflare.com'],
  },
})
```

**Next.js (next.config.js):**
```javascript
module.exports = {
  // ... existing config
  async headers() {
    return [
      {
        source: '/:path*',
        headers: [
          { key: 'Access-Control-Allow-Origin', value: '*' },
        ],
      },
    ]
  },
}
```

**Webpack Dev Server:**
```javascript
devServer: {
  allowedHosts: ['.trycloudflare.com'],
}
```

**Implementation when dev server detected:**
```bash
# STEP 1: Install cloudflared if needed
if ! command -v cloudflared &>/dev/null; then
  echo "📦 Installing cloudflared..."
  brew install cloudflare/cloudflare/cloudflared
fi

# STEP 2: Detect port from dev server output
# Parse from patterns like "Local: http://localhost:5173"
PORT=5173  # or parse from actual output

# STEP 3: Configure dev server (example for Vite)
# Check if vite.config.js exists and needs updating
if [ -f "vite.config.js" ]; then
  if ! grep -q "allowedHosts" vite.config.js; then
    echo "⚠️  Vite needs configuration to allow cloudflare tunnels"
    echo "Adding allowedHosts to vite.config.js..."
    # Use Edit tool to add server.allowedHosts config
    # Then restart dev server
  fi
fi

# STEP 4: Clean up any existing tunnels
pkill cloudflared 2>/dev/null
sleep 1
rm -f /tmp/cloudflare-tunnel.log

# STEP 5: Create tunnel with HTTP/2 protocol (more reliable than QUIC)
echo "🌐 Creating shareable link..."
cloudflared tunnel --url http://localhost:$PORT --protocol http2 > /tmp/cloudflare-tunnel.log 2>&1 &
TUNNEL_PID=$!

# STEP 6: Wait for connection to establish (8 seconds, not 3)
sleep 8

# STEP 7: Verify tunnel is running and connected
if ps -p $TUNNEL_PID > /dev/null; then
  # Check if connection actually registered
  if grep -q "Registered tunnel connection" /tmp/cloudflare-tunnel.log; then
    echo "✓ Tunnel connected to Cloudflare edge"
  else
    echo "⚠️  Tunnel started but connection not confirmed"
  fi
  
  # Extract URL using grep -a to handle binary characters
  TUNNEL_URL=$(cat /tmp/cloudflare-tunnel.log | grep -a "trycloudflare.com" | grep -oE 'https://[a-z0-9-]+\.trycloudflare\.com' | head -1)
  
  if [ -n "$TUNNEL_URL" ]; then
    echo ""
    echo "✅ Shareable link ready!"
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo "🔗 Share this with your pair partner:"
    echo ""
    echo "   $TUNNEL_URL"
    echo ""
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo ""
    echo "📍 Local:  http://localhost:$PORT"
    echo "🌐 Remote: $TUNNEL_URL"
    echo ""
    echo "Tunnel running in background (PID: $TUNNEL_PID)"
    echo "To stop: pkill cloudflared"
  else
    echo "❌ Failed to get tunnel URL. Check /tmp/cloudflare-tunnel.log"
    tail -10 /tmp/cloudflare-tunnel.log
  fi
else
  echo "❌ Tunnel process died. Logs:"
  cat /tmp/cloudflare-tunnel.log
fi
```

**Common Issues & Fixes:**

| Issue | Symptom | Fix |
|-------|---------|-----|
| **Host not allowed** | "This host is not allowed" error | Add `.trycloudflare.com` to dev server's allowedHosts config, restart server |
| **Connection timeout** | "Failed to dial a quic connection" | Use `--protocol http2` instead of default QUIC |
| **Multiple tunnels** | Conflicting processes | Run `pkill cloudflared` before creating new tunnel |
| **Tunnel not ready** | Empty URL after 3 seconds | Increase wait time to 8 seconds, check for "Registered tunnel connection" in logs |
| **Port mismatch** | Tunnel points to wrong port | Dev server port may change on restart - parse actual port from output |

**Stopping the tunnel:**
- Manual: `pkill cloudflared`
- Automatic: Tunnel stops when Claude exits or you restart dev server
- Check running tunnels: `ps aux | grep cloudflared`

## Setup & Prerequisites

Before using `handoff` or `takeover`, verify your git repository has a remote configured:

**Check if remote exists:**
```bash
git remote -v
```

If the output is empty or doesn't show "origin", you need to set up a remote repository.

### Setting Up a Remote (For Non-Technical Users)

A "remote" is like a shared Google Drive folder for your code. Both you and your pair partner need access to it.

**Step 1: Create a shared repository**

Choose one of these services (all free for basic use):

**GitHub:**
1. Go to https://github.com/new
2. Name your repository (e.g., "my-project")
3. Choose "Private" if you don't want it public
4. Click "Create repository"
5. Copy the URL shown (looks like `https://github.com/yourname/my-project.git`)

**GitLab:**
1. Go to https://gitlab.com/projects/new
2. Name your project
3. Set visibility (Private/Public)
4. Click "Create project"
5. Copy the URL from the project page

**Step 2: Connect your local repository to the remote**

```bash
# Replace YOUR_REPO_URL with the URL you copied above
git remote add origin YOUR_REPO_URL

# Verify it worked
git remote -v
```

**Step 3: Grant your pair partner access**

- **GitHub:** Settings → Collaborators → Add people (enter their GitHub username)
- **GitLab:** Project Settings → Members → Invite member

**Step 4: Push your initial code** (only needed once):

```bash
git branch -M main
git push -u origin main
```

Now you're ready to use `handoff` and `takeover`!

## Triggers

**CRITICAL:** These are ATOMIC operations. Execute the entire script - don't break into manual steps.

**IMPORTANT:** Before executing any trigger, verify remote exists. If `git remote -v` is empty, guide user through setup above.

**Note:** `handoff` and `takeover` are trigger patterns, not slash commands. When Claude sees these words, it executes the corresponding bash script.

### handoff

Completes your turn and hands off to pair.

**What it does (all atomically):**
1. Stage all changes (`git add -A`)
2. Commit with descriptive message
3. Push to remote
4. **Auto-detect** push rejection and **auto-recover** (sync and retry)

**User invocation:**
- User says "handoff", "let my partner take over", or "I'm done"
- You execute the full script below (don't break into steps)

**Implementation (execute as single atomic operation):**
```bash
# Pre-flight check: verify remote exists
if ! git remote get-url origin &>/dev/null; then
  echo "❌ No remote repository configured!"
  echo ""
  echo "To use handoff, you need a shared remote repository (like GitHub or GitLab)."
  echo "Both you and your pair partner need access to push/pull code."
  echo ""
  echo "Quick setup:"
  echo "1. Create a repository on GitHub.com or GitLab.com"
  echo "2. Run: git remote add origin YOUR_REPO_URL"
  echo "3. Run: git push -u origin main"
  echo ""
  echo "See the 'Setup & Prerequisites' section for detailed instructions."
  exit 1
fi

# Stage everything
git add -A

# Commit with auto-generated message
git commit -m "Handoff: [brief description of work done]"

# Push, handling rejection
if ! git push origin main; then
  echo "Behind remote, syncing..."
  git pull --rebase origin main
  git push origin main
fi

echo "✓ Handed off to pair. They can use takeover to sync"
```

**Auto-generate commit message** from git diff summary - describe what was changed, not "handoff" generically.

### takeover

### takeover

Starts your turn by syncing latest changes.

**What it does:**
1. Pull latest changes with rebase
2. Verify clean working tree
3. Confirm ready to work

**User invocation:**
- User says "takeover" or "I'm taking over"

**Implementation:**
```bash
# Pre-flight check: verify remote exists
if ! git remote get-url origin &>/dev/null; then
  echo "❌ No remote repository configured!"
  echo ""
  echo "To use takeover, you need a shared remote repository (like GitHub or GitLab)."
  echo "Your pair partner should have set this up and granted you access."
  echo ""
  echo "If they already set it up, run:"
  echo "  git remote add origin THEIR_REPO_URL"
  echo ""
  echo "If you're setting up for the first time, see 'Setup & Prerequisites' section."
  exit 1
fi

# Pull with rebase for linear history
git pull --rebase origin main

# Verify clean state
git status

echo "✓ Synced with pair's changes. Ready to work."
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| **No remote configured** | Run `git remote -v` to check. If empty, follow "Setup & Prerequisites" to create and connect a remote repository |
| Forgot to `takeover` before starting | Push will fail. Run `takeover`, resolve conflicts, then continue |
| Partner pushed while you're working | Run `takeover` to sync, resolve conflicts, then `handoff` |
| Conflicts during `takeover` | Git pauses for manual resolution. Fix conflicts, `git rebase --continue`, then retry |
| Working on uncommitted changes during `handoff` | All changes get committed. Verify `git status` first if you want to exclude files |

## Red Flags - STOP and Use handoff

These thoughts mean you're about to break the atomic operation:

| Thought | Reality |
|---------|---------|
| "Stage and commit first, then run handoff" | `handoff` INCLUDES staging and committing. Don't do it manually. |
| "Pull before handing off" | `handoff` auto-detects if behind and pulls automatically. Don't pull first. |
| "User is behind, need to sync manually" | `handoff` handles this. Just run the full script. |
| "Let me break this into steps" | Execute the entire `handoff` script atomically. |

**If user is behind remote:** `handoff` detects and recovers automatically. Don't add manual steps.

## Push Rejection Handling

When `handoff` push fails (you're behind remote):

```bash
! [rejected]        main -> main (fetch first)
```

**Auto-recovery:**
1. Run `git pull --rebase origin main`
2. Resolve any conflicts
3. Retry push

**Built into `handoff`** - no manual intervention unless conflicts occur.

## Optional: Auto-Sync on Session Start

Add to settings.json for automatic pull when Claude Code starts:

```json
{
  "hooks": {
    "SessionStart": "git pull --rebase origin main 2>&1 || echo 'Could not auto-sync'"
  }
}
```

**Benefit:** Never forget to sync before starting work.

**Risk:** May pull unexpected changes if you branched off locally.

## Quick Reference

| Action | Trigger | What it does |
|--------|---------|--------------|
| Finish your turn | `handoff` | Stage, commit, push (auto-handle rejection) |
| Start your turn | `takeover` | Pull latest with rebase |
| Share dev server | Auto-detected or "share dev server" | Creates cloudflare tunnel and displays link |
| Auto-pull on start | Add SessionStart hook | Syncs automatically when Claude Code launches |

## Real-World Example

**Session 1 - Developer A (Anders):**
```
Anders: "Create a new React app with Vite"
Claude: [creates project]
Anders: "npm run dev"
Claude: [detects dev server starting on localhost:5173]
       🌐 Creating shareable link...
       
       ✅ Shareable link ready!
       ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
       🔗 Share this with your pair partner:
       
          https://abc-123-def.trycloudflare.com
       
       ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Anders: [shares link with Bob, Bob can now watch in real-time]
Anders: "Add login form component"
Claude: [implements LoginForm.tsx - Bob sees it hot reload instantly]
Anders: "handoff"
Claude: [stages, commits "Add login form with email/password fields", pushes]
       ✅ Handed off to pair
```

**Session 2 - Developer B (Bob):**
```
Bob: "takeover"
Claude: [pulls latest, shows LoginForm.tsx was added]
       ✓ Synced with pair's changes
Bob: "npm run dev"
Claude: [detects dev server, creates new tunnel]
       🔗 Share: https://xyz-456-ghi.trycloudflare.com
       
Bob: [shares link with Anders]
Bob: "Add authentication API"
Claude: [implements auth.ts - Anders watches live]
Bob: "handoff"
Claude: [stages, commits "Add auth API with JWT tokens", pushes]
```

**Session 3 - Developer A (forgot to takeover):**
```
Anders: "Add password reset feature"
Claude: [implements PasswordReset.tsx]
Anders: "handoff"
Claude: [tries to push, REJECTED - behind remote]
       Behind remote, syncing...
       [auto-runs git pull --rebase]
       [retries push, succeeds]
       ✓ Handed off to pair
```

## Implementation Notes

- `handoff` and `takeover` are **trigger patterns**, not slash commands or bash aliases
- When user says "handoff" or "takeover", you execute the corresponding bash script
- Use `git add -A` to stage all changes (including untracked files)
- Prefer `--rebase` over merge for cleaner history
- Always check for conflicts after rebase - pause for user resolution if needed
- Generate descriptive commit messages from `git diff --stat` summary

## Cloudflare Tunnel Troubleshooting

**Real-world debugging session (2026-04-16):**

Issues encountered and resolved:

1. **Error 1033 - Cloudflare unable to resolve tunnel**
   - **Cause**: Multiple cloudflared processes running simultaneously
   - **Fix**: `pkill cloudflared` before creating new tunnel
   - **Lesson**: Always clean up existing tunnels first

2. **"Failed to dial a quic connection" - timeout errors**
   - **Cause**: Default QUIC protocol blocked by firewall/network
   - **Symptoms**: Logs show "timeout: handshake did not complete in time"
   - **Fix**: Use `--protocol http2` explicitly
   - **Lesson**: HTTP/2 is more reliable for quick tunnels

3. **"This host is not allowed" - Vite blocking cloudflare**
   - **Cause**: Vite's host checking blocks unknown domains by default
   - **Symptoms**: Tunnel connects but browser shows "host not allowed"
   - **Fix**: Add `server.allowedHosts: ['.trycloudflare.com']` to vite.config.js
   - **Lesson**: Dev server must be configured BEFORE tunnel creation, requires restart

4. **Port mismatch after server restart**
   - **Cause**: Vite chose different port (5173 vs 5175) after restart
   - **Symptoms**: Tunnel points to old port, connection fails
   - **Fix**: Parse actual port from dev server output, recreate tunnel
   - **Lesson**: Always verify port after config changes that require restart

5. **Tunnel URL not appearing**
   - **Cause**: Not waiting long enough for connection
   - **Symptoms**: Logs show "Requesting new quick Tunnel" but no URL yet
   - **Fix**: Wait 8 seconds instead of 3, verify "Registered tunnel connection" in logs
   - **Lesson**: Connection establishment takes time, verify success before displaying URL

**Correct order of operations:**
1. ✅ Configure dev server to allow cloudflare hosts
2. ✅ Restart dev server (if config changed)
3. ✅ Parse actual port from server output
4. ✅ Kill existing cloudflared processes
5. ✅ Create tunnel with `--protocol http2`
6. ✅ Wait 8 seconds for connection
7. ✅ Verify "Registered tunnel connection" in logs
8. ✅ Extract and display URL
