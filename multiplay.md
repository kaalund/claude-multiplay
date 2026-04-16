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

**When Claude detects a dev server starting**, it will:
1. Check if `cloudflared` is installed (if not, offer to install it)
2. Automatically run `cloudflared tunnel --url http://localhost:PORT`
3. Display the shareable URL prominently
4. Keep the tunnel running in the background

**Manual tunnel creation:**
If you need to create a tunnel manually, say "share my dev server" or "create tunnel"

**Implementation when dev server detected:**
```bash
# Detect common dev server patterns:
# - "Local: http://localhost:5173"
# - "Server running at http://localhost:3000"
# Parse the port number, then:

# Check if cloudflared is installed
if ! command -v cloudflared &>/dev/null; then
  echo "📦 Installing cloudflared..."
  brew install cloudflare/cloudflare/cloudflared
fi

# Start tunnel in background
echo "🌐 Creating shareable link..."
cloudflared tunnel --url http://localhost:PORT > /tmp/cloudflare-tunnel.log 2>&1 &
TUNNEL_PID=$!

# Wait for tunnel URL to appear
sleep 3
TUNNEL_URL=$(grep -oE 'https://[a-z0-9-]+\.trycloudflare\.com' /tmp/cloudflare-tunnel.log | head -1)

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
  echo "Tunnel running in background (PID: $TUNNEL_PID)"
  echo "To stop: kill $TUNNEL_PID"
else
  echo "❌ Failed to create tunnel. Check /tmp/cloudflare-tunnel.log"
fi
```

**Stopping the tunnel:**
The tunnel will automatically stop when you exit Claude or restart your dev server. To manually stop: `pkill cloudflared`

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
