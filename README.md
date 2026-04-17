# Claude Multiplay

A Claude Code skill for seamless pair programming with automatic git handoffs and live dev server sharing.

## What You Get

- **One-Command Handoffs**: Say "handoff" to commit & push. Say "takeover" to pull & start coding.
- **Live Preview**: Your partner sees your changes in real-time with hot reload
- **Zero Config Tunnels**: No accounts or auth tokens needed to share your localhost
- **Smart Sync**: Auto-detects when you're behind and syncs before pushing

---

## Complete Setup Guide (For Beginners)

Follow these steps if you're new to pair programming with git.

### Step 1: Create a GitHub Account (First Time Only)

**Person A (You):**

1. Go to [github.com](https://github.com)
2. Click **Sign up**
3. Create your account (remember your username!)

**Person B (Your Partner):**

1. Do the same - create their own GitHub account
2. Share their username with you

---

### Step 2: Install SourceTree (First Time Only)

**Both developers need to do this:**

1. Download SourceTree: [sourcetreeapp.com](https://www.sourcetreeapp.com/)
2. Install it (follow the installer)
3. Open SourceTree
4. Skip Bitbucket/Atlassian account setup (click "Skip" or "Use GitHub")
5. When asked to add GitHub account:
   - Select **GitHub**
   - Click **Connect Account**
   - Sign in to GitHub in your browser
   - Authorize SourceTree

---

### Step 3: Person A - Create the Project Repository

**Person A only:**

1. **Create a folder for your project:**
   ```bash
   mkdir my-project
   cd my-project
   ```

2. **Initialize git:**
   ```bash
   git init
   git branch -M main
   ```

3. **Create a GitHub repository:**
   - Go to [github.com/new](https://github.com/new)
   - Repository name: `my-project` (or whatever you want)
   - Make it **Private** (unless you want it public)
   - **Don't** check any of the "initialize" boxes
   - Click **Create repository**

4. **Connect your local folder to GitHub:**
   
   GitHub will show you commands. Copy and run them:
   ```bash
   git remote add origin https://github.com/YOUR_USERNAME/my-project.git
   ```
   
   (Replace `YOUR_USERNAME` with your actual GitHub username)

5. **Make your first commit:**
   ```bash
   echo "# My Project" > README.md
   git add README.md
   git commit -m "Initial commit"
   git push -u origin main
   ```

6. **Invite Person B as a collaborator:**
   - Go to your repo: `https://github.com/YOUR_USERNAME/my-project`
   - Click **Settings** tab
   - Click **Collaborators** in the left sidebar
   - Click **Add people**
   - Enter Person B's GitHub username
   - Click **Add [username] to this repository**

7. **Person B will receive an email invite** - they need to accept it!

---

### Step 4: Person B - Clone the Repository

**Person B only (after accepting the invite):**

1. **Get the repository URL from Person A:**
   
   Person A should share: `https://github.com/PERSON_A_USERNAME/my-project.git`

2. **Clone using SourceTree:**
   - Open SourceTree
   - Click **+ New** tab
   - Click **Clone from URL**
   - **Source URL:** Paste the repository URL
   - **Destination Path:** Choose where to save it (e.g., `/Users/yourname/my-project`)
   - Click **Clone**

   Or clone using terminal:
   ```bash
   git clone https://github.com/PERSON_A_USERNAME/my-project.git
   cd my-project
   ```

3. **You now have the same code!**

---

### Step 5: Install Multiplay Skill

**Both developers:**

```bash
# Create skills directory
mkdir -p ~/.claude/skills/multiplay

# Download the skill
curl -o ~/.claude/skills/multiplay/SKILL.md \
  https://raw.githubusercontent.com/kaalund/claude-multiplay/main/SKILL.md

# Restart Claude Code
```

---

### Step 6: Install Cloudflare Tunnel (First Time Only)

**Both developers:**

```bash
# On Mac:
brew install cloudflare/cloudflare/cloudflared

# On Windows (using Chocolatey):
choco install cloudflared
```

Don't worry - Claude will auto-install this if you skip it!

---

## How to Use Multiplay

### Person A Starts Coding

```bash
# Person A:
cd my-project
code .  # or open in your editor

# Start Claude Code
claude

# Start your dev server (Vite, Next.js, etc.)
npm run dev
```

**Claude automatically detects your dev server**, configures it to allow tunnel connections, creates a Cloudflare tunnel, and shows:
```
✅ Shareable link ready!
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔗 Share this with your pair partner:

   https://abc-123-def.trycloudflare.com

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

> **Note:** Claude may need to add `allowedHosts` config to your dev server (e.g. Vite, Webpack) and restart it. This is normal - it only happens once per project.

**Share that link with Person B** - they can watch your changes live!

### When Person A is Done

```bash
# Person A types in Claude:
handoff
```

Claude will:
1. Stage all your changes
2. Create a commit
3. Push to GitHub

### Person B Takes Over

```bash
# Person B types in Claude:
takeover
```

Claude will:
1. Pull the latest changes
2. Show what changed
3. You're ready to code!

```bash
# Person B starts their dev server:
npm run dev
```

Claude creates a **new** shareable link for Person A to watch.

### Keep Taking Turns

- Person B codes, then types: `handoff`
- Person A types: `takeover`
- Repeat!

---

## Using SourceTree to Push/Pull

If you prefer SourceTree instead of the `handoff`/`takeover` commands:

### To Push Your Changes:
1. Open SourceTree
2. Review changed files at bottom
3. Check the files you want to commit
4. Enter commit message at bottom
5. Click **Commit** button
6. Click **Push** button in toolbar

### To Pull Partner's Changes:
1. Open SourceTree
2. Click **Pull** button in toolbar
3. Click **OK**

---

## Troubleshooting

### "No remote configured"

```bash
# Check if remote exists:
git remote -v

# If empty, add it:
git remote add origin https://github.com/USERNAME/REPO.git
```

### "Permission denied" when pushing

You need to authenticate SourceTree with GitHub:

1. **Generate a Personal Access Token:**
   - Go to: [github.com/settings/tokens](https://github.com/settings/tokens)
   - Click **Generate new token (classic)**
   - Name it: "SourceTree"
   - Check: **`repo`** (full control of private repositories)
   - Click **Generate token**
   - **COPY THE TOKEN** (you won't see it again!)

2. **Add to SourceTree:**
   - SourceTree → **Preferences** (⌘,)
   - Click **Accounts** tab
   - Click **Add...**
   - Host: **GitHub**
   - Auth Type: **Basic**
   - Username: Your GitHub username
   - Password: **PASTE THE TOKEN HERE**
   - Click **OK**

### Can't see partner's changes live

Make sure:
- Your partner ran `handoff` to push their changes
- You ran `takeover` to pull them
- Or just pull in SourceTree

The live link is for **watching** as they code. Use `takeover` to actually get the code.

### Tunnel link shows "This host is not allowed"

Your dev server is blocking the Cloudflare tunnel domain. Claude should handle this automatically, but if it didn't:

**For Vite** - add to `vite.config.js`:
```javascript
server: { allowedHosts: ['.trycloudflare.com'] }
```

**For Webpack Dev Server** - add to config:
```javascript
devServer: { allowedHosts: ['.trycloudflare.com'] }
```

Then restart your dev server and tell Claude to create the tunnel again.

### Tunnel fails with timeout errors

This usually means the default QUIC protocol is blocked by your network. Tell Claude "create tunnel with http2" or it should use `--protocol http2` automatically.

### Tunnel link doesn't work / multiple tunnels

Kill any old tunnels and try again:
```bash
pkill cloudflared
```
Then ask Claude to create a new tunnel.

---

## What's Happening Behind the Scenes

When you say **"handoff"**, Claude runs:
```bash
git add -A
git commit -m "Your changes"
git push origin main
```

When you say **"takeover"**, Claude runs:
```bash
git pull --rebase origin main
```

The Cloudflare tunnel lets your partner's browser connect to your `localhost` through the internet. It's temporary and auto-closes when you stop your dev server. Claude uses `--protocol http2` for reliable connections.

---

## Requirements

- Git installed
- GitHub account
- SourceTree (or command line git skills)
- Claude Code
- Homebrew (for cloudflared)
- A project with a dev server (Vite, Next.js, Create React App, etc.)

---

## License

MIT - See [LICENSE](LICENSE) file
