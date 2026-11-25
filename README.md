# Docker Sandboxes: A Hands-On Guide with Claude

> **Status**: This guide covers the Docker Sandboxes `sbx` CLI and was last checked
> against **v0.54.1** (November 2026). `sbx` changes quickly and some commands are
> marked experimental. If something here doesn't match your version, check
> `sbx <command> --help` and the [Docker Sandboxes docs](https://docs.docker.com/ai/sandboxes/).

---

## What you'll need

- A paid Claude subscription
- A GitHub account with a token that has permissions to push and pull
- macOS on Apple Silicon or Windows 11
- *(Optional, for section 15)* VS Code with the Remote - SSH extension, or Cursor

## What you'll learn

By the end of this guide you'll be able to:

- Install and configure the `sbx` CLI
- Run Claude autonomously inside an isolated microVM sandbox
- Store credentials securely and have them injected automatically
- Use clone mode to let Claude work on its own Git clone and branches without touching your working tree
- Run multiple agents in parallel on the same repo
- Forward live ports from a sandbox to your browser
- Manage network policies so Claude can only reach what you allow
- Mount multiple workspaces and debug inside a running sandbox
- SSH into a sandbox and connect your IDE to it, so your editor's extensions, terminals, and tasks run inside the sandbox instead of on your host

The guide uses **DevBoard** — a full-stack Next.js + FastAPI issue tracker included
alongside this file (`backend/`, `frontend/`, `docker-compose.yml`). DevBoard has
real-world complexity: a REST API, a Postgres database, JWT auth, tests, and a handful
of intentional bugs and unfinished features that make ideal exercises for Claude.

---

## Table of contents

1. [How Docker Sandboxes work](#1-how-docker-sandboxes-work)
2. [Fork and clone this repo](#2-fork-and-clone-this-repo)
3. [Installation](#3-installation)
4. [Secrets and credentials](#4-secrets-and-credentials)
5. [Create your sandbox](#5-create-your-sandbox)
6. [Orient yourself](#6-orient-yourself)
7. [The interactive TUI dashboard](#7-the-interactive-tui-dashboard)
8. [Run tests and fix bugs](#8-run-tests-and-fix-bugs)
9. [Clone mode and parallel agents](#9-clone-mode-and-parallel-agents)
10. [Docker Compose inside the sandbox](#10-docker-compose-inside-the-sandbox)
11. [Port forwarding with `sbx port`](#11-port-forwarding-with-sbx-port)
12. [Network policies](#12-network-policies)
13. [Multiple workspaces](#13-multiple-workspaces)
14. [Debugging with `sbx exec`](#14-debugging-with-sbx-exec)
15. [Connect your IDE over SSH](#15-connect-your-ide-over-ssh)
16. [Custom templates](#16-custom-templates)
17. [Appendix A: Prompt library](#appendix-a-prompt-library)
18. [Appendix B: CLI quick reference](#appendix-b-cli-quick-reference)
19. [Appendix C: Troubleshooting](#appendix-c-troubleshooting)

---

## 1. How Docker Sandboxes work

When you run `sbx run claude`, Docker Sandboxes:

1. Spins up a **lightweight microVM** — its own Linux kernel, not just a container namespace.
2. Gives the VM a **private Docker daemon**, so Claude can run `docker build` or `docker compose up` without touching your host Docker.
3. **Mounts your workspace directory** at its exact host path inside the VM. File changes are instant in both directions — no copy-on-write delay.
4. Routes all HTTP/HTTPS traffic from the VM through a **host-side proxy** that enforces your network policy and injects API credentials. Claude never sees raw credentials.
5. Starts Claude with `--dangerously-skip-permissions` so it can act autonomously without prompting you on every file change.

The result: Claude can build images, install packages, run tests, and edit your code —
and none of that can escape the VM to touch your host system, your other containers,
or any network destination you haven't explicitly allowed.

```
Your machine
├── Host Docker daemon    ← your stuff, untouched
├── Host filesystem       ← workspace dir shared (read/write); nothing else
│
└── Sandbox (microVM)
    ├── Private Docker daemon  ← Claude builds here
    ├── /your/workspace        ← live-mounted from host
    └── Outbound HTTP proxy    ← enforces network policy, injects creds
```

---

## 2. Fork and clone this repo

Before diving into the exercises, fork this repo to your GitHub account so you can push branches and open pull requests as you work through the guide.

**Step 1** — Fork on GitHub by clicking the **Fork** button at the top of the repo page.

**Step 2** — Clone your fork to your local machine:

```bash
git clone https://github.com/<your-username>/sbx-quickstart.git ~/sbx-quickstart
cd ~/sbx-quickstart
```

From this point on, all commands assume you're in `~/sbx-quickstart`. In some cases, such as opening a new shell, you will need to change into that directory again, but the guide should direct you if that's the case. 

---

## 3. Installation

> Docker Desktop is **not** required to run `sbx`

### macOS (Apple Silicon required)

**Install the CLI via Homebrew**

```bash
brew install docker/tap/sbx
```

### Windows (x86_64, Windows 11 required)

**Enable the Windows Hypervisor Platform** (requires an elevated terminal):

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName HypervisorPlatform -All
```

> Restart your machine when prompted.

**Install the `sbx` CLI via `winget`**

```powershell
winget install -h Docker.sbx
```

Now that `sbx`has been installed it's time to do some initial configuration. 

**Sign in**

You need to sign in with your Docker ID in order to use `sbx`. Follow the OAuth workflow.

```bash
sbx login
```

**Set default network policy**

On your **very first run** the daemon prompts you to choose a network policy:

```
Choose a default network policy:

     1. Open         — All network traffic allowed, no restrictions.
     2. Balanced     — Default deny, with common dev sites allowed.
     3. Locked Down  — All network traffic blocked unless you allow it.

  Use ↑/↓ to navigate, Enter to select, or press 1–3.
```

Choose **Balanced** for this guide. It allows AI provider APIs, package managers
(npm, pip, PyPI), GitHub, and container registries out of the box. You can add more
hosts later with `sbx policy allow`.

---

## 4. Secrets and credentials

`sbx` has a built-in secrets manager that stores credentials in your OS keychain —
never in plain text on disk or inside the VM. When Claude makes an outbound request
that needs authentication, the host-side proxy intercepts it and injects the credential
automatically. Claude can make authenticated API calls but can never read, log, or
exfiltrate the raw credential.

Store your GitHub token **now**. Service secrets are **global by default** —
available to every sandbox, including ones you create later:

```bash
gh auth token | sbx secret set github
```

> **Good to know**: adding, updating, or removing a secret takes effect in existing
> sandboxes without a restart. To scope a secret to a single sandbox, add
> `--sandbox <name>` (for example `sbx secret set openai --sandbox quickstart`).
> Sandbox-scoped secrets take precedence over global ones.

You can doublecheck that the credential was added. 

```bash
sbx secret ls
```

You should see something like: 

```
SCOPE      SERVICE   SECRET
(global)   github    gho_TK421****...****R2D2
```

### Supported services

| Service      | Environment variable(s)    | API domain(s)                                                                                       |
|--------------|----------------------------|-----------------------------------------------------------------------------------------------------|
| `anthropic`  | `ANTHROPIC_API_KEY`        | `api.anthropic.com`, `console.anthropic.com`, `claude.ai`, `mcp-proxy.anthropic.com`                |
| `cursor`     | `CURSOR_API_KEY`           | `api2.cursor.sh`, `api3.cursor.sh`, `repo42.cursor.sh`, `cursor.com`                                |
| `droid`      | `FACTORY_API_KEY`          | `api.factory.ai`, `app.factory.ai`, `relay.factory.ai`                                              |
| `github`     | `GH_TOKEN`, `GITHUB_TOKEN` | `api.github.com`, `github.com`, `raw.githubusercontent.com`, `gist.github.com`, `copilot.github.com`, `api.githubcopilot.com` |
| `google`     | `GOOGLE_API_KEY`           | `generativelanguage.googleapis.com`, `oauth2.googleapis.com`, `aiplatform.googleapis.com`, `vertexai.googleapis.com` |
| `groq`       | `GROQ_API_KEY`             | `api.groq.com`                                                                                      |
| `mistral`    | `MISTRAL_API_KEY`          | `api.mistral.ai`                                                                                    |
| `nebius`     | `NEBIUS_API_KEY`           | `api.studio.nebius.com`, `api.tokenfactory.nebius.com`                                              |
| `openai`     | `OPENAI_API_KEY`           | `api.openai.com`, `openai.com`, `chatgpt.com`, `www.chatgpt.com`                                    |
| `openrouter` | `OPENROUTER_API_KEY`       | `openrouter.ai`                                                                                     |
| `xai`        | `XAI_API_KEY`              | `api.x.ai`                                                                                          |

`sbx secret set` also accepts `copilot` and `devin`, and kits can declare their own
services. Run `sbx secret set --help` for the current list.

For services not in this list, you can write values to `/etc/sandbox-persistent.sh`
inside the sandbox via `sbx exec`. Unlike `sbx secret`, this stores the value inside
the VM where the agent can read it directly — use only for tokens where proxy injection
isn't needed. If you still want proxy injection, try the experimental
`sbx secret set-custom`, which keys a secret to a domain and an environment variable.

---

## 5. Create your sandbox

### Bring your Claude config (optional)

The sandbox can only see your workspace directory — per-user config files like `~/.claude/CLAUDE.md` or `~/.claude/settings.json` are not available inside the VM. If you rely on any of those, copy them into your project before creating the sandbox:

```bash
cp ~/.claude/CLAUDE.md ~/sbx-quickstart/CLAUDE.md
```

> **Note**: Symlinks won't work here. The sandbox cannot follow a symlink that points outside its designated workspace, so copy the files directly.

---

`sbx create` provisions a sandbox without attaching to it — useful when you want to
set it up, verify it appears in `sbx ls`, or script multiple sandboxes before starting
any of them. `sbx run` then attaches an agent session to an existing sandbox (or
creates one on the fly if it doesn't exist yet).

> **Important**: run these commands from your cloned repo directory. If you followed section 2, that's `~/sbx-quickstart`. The `.` tells `sbx` to mount the current directory as the workspace.

Start a sandbox named `quickstart` using the claude agent:

```bash
sbx create --name=quickstart claude .
```

Confirm it was created:

```bash
sbx ls
```

Now attach to it:

```bash
sbx run quickstart
```

### Log in to Claude

Once the sandbox starts, the Claude interface loads. Authenticate with:

```
/login
```

Choose your preferred login option.

> **Note**: If you choose Option 1, the `c for copy` shortcut does not work and the
> sandbox will not automatically open a browser. Copy the login URL manually, complete
> the OAuth flow, and paste the returned code back into the sandbox. You only need to
> do this once.


---

## 6. Orient yourself

Once Claude is authenticated, give Claude the following prompt:

```
Explore this codebase and give me:
1. A summary of the architecture and tech stack
2. How to run it locally (without Docker Compose if possible)
3. Any obvious issues or areas of concern you spot at a glance
4. What the test suite covers
```

Claude will read the source files, check the requirements, and report back. Because the
workspace is mounted directly into the VM, Claude sees your actual files — including
any changes you make on the host while it's running.

### Controlling the session

`sbx run` is interactive — it occupies your terminal as a live agent session.

- Press **`Ctrl-C` twice** to exit the session and drop back to your host terminal. 
- Type **`!`** before any command inside Claude to run it as a shell command without leaving the session — e.g. `!ls` or `!git status`.

Go ahead and exit the sandbox by **pressing `ctrl-c` twice** 

---

## 7. The interactive TUI dashboard

Running `sbx` with no arguments opens the TUI. 

```bash
sbx
```

The dashboard shows all sandboxes as cards with live CPU and memory usage.

| Key     | Action                                               |
|---------|------------------------------------------------------|
| `c`     | Create a new sandbox                                 |
| `s`     | Start or stop the selected sandbox                   |
| `Enter` | Attach to the agent session (same as `sbx run`)      |
| `x`     | Open a shell inside the sandbox (`sbx exec`)         |
| `r`     | Remove the selected sandbox                          |
| `Tab`   | Switch between the Sandboxes panel and Network panel |
| `?`     | Show all shortcuts                                   |

The **Network panel** (press `Tab`) shows a live log of every outbound connection the
sandbox makes — which hosts were reached, which were blocked. Use the arrow keys to
navigate the log and allow or block hosts directly. This is the fastest way to debug
"why can't Claude install this package?"

**Press `Ctrl-C` and then `Y`** to exit the dashboard without stopping any sandboxes.

---

## 8. Run tests and fix bugs

This exercise shows Claude working in **direct mode** — the default, where edits land
in your working tree. You'll have Claude run the test suite, identify failures, fix the
bugs, and confirm everything passes.

Reconnect to your sandbox:

```bash
sbx run quickstart
```

**Step 1 — Run the tests:**

Give Claude the following prompt:

```
Set up the Python environment for the FastAPI backend and run the test suite.
Report:
- Which tests pass
- Which tests fail, with the full error output
- Your diagnosis of each failure

Use pytest with verbose output: cd backend && pytest tests/ -v
```

It will take about 3-4 minutes for this to all complete. 

**Step 2 — Fix the bugs:**

Give Claude the following prompt:

```
Two tests are failing due to a pagination bug. Fix it:

1. test_pagination_first_page_returns_results — the pagination offset is wrong
2. test_pagination_second_page — related to the same bug

For each fix:
- Explain what the bug is and why it causes the failure
- Show the diff of your change
- Re-run the specific test to confirm it passes before moving on
```

While Claude works, you can open the affected files in your editor on the host. You'll see Claude's
edits — the workspace mount is bidirectional and instant. No copy
step, no polling delay.

**Step 3 — Confirm everything passes:**

Give Claude the following prompt:

```
Run the full test suite and confirm all tests pass (or are intentionally skipped).
```

Installed packages **persist** across agent restarts for this sandbox. If you stop
and reconnect to `quickstart`, you won't need to `pip install` again.

---

## 9. Clone mode and parallel agents

### Direct mode vs. clone mode

Everything in section 8 ran in **direct mode** — Claude edited your working tree and
you could see the changes immediately. That's great for interactive work.

**Clone mode** gives Claude its own private Git clone of your repo inside the
sandbox. Your repo is mounted **read-only** (at `/run/sandbox/source`), Claude edits
and commits in its clone, and nothing reaches your working tree until you fetch it.
Use it when you want a clean set of commits to review before anything lands, or when
running several tasks at once.

|                  | Direct mode (section 8) | Clone mode                              |
|------------------|-------------------------|-----------------------------------------|
| Claude edits     | Your working tree       | A private Git clone inside the sandbox  |
| You see changes  | Immediately             | After `git fetch sandbox-<name>`        |
| How to enable    | Default                 | `--clone`, at creation time only        |

A few rules to know before you start:

- **Clone mode is fixed at creation.** You can't switch `quickstart` to clone mode,
  so you'll create a second sandbox for this section.
- **The clone follows the branch you have checked out** when you create the sandbox.
  It's a Git clone, so it only contains **committed** work. Uncommitted changes and
  untracked files (such as a `CLAUDE.md` you copied in section 5) aren't in it.
- **Your host repo gets a `sandbox-<name>` remote** for fetching Claude's commits.
  Fetching only works while the sandbox is running.
- **`sbx rm` deletes the clone** and removes that remote. Fetch or push anything you
  want to keep first.
- Run `--clone` from your main checkout, not from inside a Git worktree.

> **Note**: earlier versions of this guide used a `--branch` flag that created
> worktrees under `.sbx/`. Current `sbx run` has no `--branch` flag — use clone mode
> instead.

### Single task: fix a bug on its own branch

If you are currently in the sandbox **press `ctrl-c` twice to exit**

**Step 1 — Commit the section 8 fix to a branch.** The pagination fix from section 8
is still uncommitted in your working tree, so a clone wouldn't see it. Save it on a
new branch:

```bash
cd ~/sbx-quickstart
git checkout -b fix-bugs
git commit -am "Fix pagination offset in list_issues()"
```

> **Note**: `git commit -am` commits changes to files Git already tracks, so stray
> files like a copied `CLAUDE.md` or a virtual environment aren't swept in. If Git
> reports `nothing to commit`, the fix isn't in your working tree — that's fine,
> carry on.

**Step 2 — Create a clone-mode sandbox.** With `fix-bugs` still checked out:

```bash
sbx create --clone --name quickstart-clone claude .
sbx ls
```

You now have two sandboxes: `quickstart` (direct mode) and `quickstart-clone`
(clone mode).

**Step 3 — Give Claude the task:**

```bash
sbx run quickstart-clone
```

If Claude asks you to sign in, run `/login` as in section 5. Your GitHub token is
already available — stored secrets are global by default.

Give Claude the following prompt:

```
You're working on the fix-bugs branch. One test is still failing: the updated_at
field in backend/app/models.py never changes on update. Set up the Python
environment, fix the bug, run the backend tests to confirm, and commit to
fix-bugs with a descriptive message.
```

This is a fresh sandbox, so Claude has to install the Python dependencies again —
expect it to take a few minutes.

**Step 4 — Review from a second terminal.** Leave the session running. Your working
tree hasn't changed — Claude's commit lives only in the clone until you fetch it:

```bash
cd ~/sbx-quickstart

git fetch sandbox-quickstart-clone

git log --oneline fix-bugs..sandbox-quickstart-clone/fix-bugs

git diff fix-bugs..sandbox-quickstart-clone/fix-bugs
```

**Step 5 — Bring it in and open a PR:**

```bash
git merge --ff-only sandbox-quickstart-clone/fix-bugs

git push -u origin fix-bugs

gh pr create \
  --head fix-bugs \
  --title "Fix: pagination offset and missing onupdate" \
  --body "Fixes off-by-one error in list_issues() and adds missing onupdate= to models."
```

### Parallel agents

One clone-mode sandbox can hold many branches, but **branches alone don't isolate
work** — two tasks editing the same working directory would trample each other. The
fix is to give each task its own **Git worktree** inside the sandbox. (If tasks also
need separate tools or running services, use separate sandboxes instead.)

DevBoard has two unimplemented features that make ideal parallel tasks:

- **Search** — `GET /issues/search?q=` returns 501; the query logic needs to be written
- **Notifications** — `send_status_change_notification()` is a no-op stub; it needs to actually send emails on issue status changes

The `prompts/` directory in this repo includes a ready-made prompt file for each.
Back in the `quickstart-clone` session (reconnect with `sbx run quickstart-clone` if
you exited), give Claude the following prompt:

```
Implement two independent features at the same time, each in its own Git
worktree so they never share a working directory:

1. Run git fetch origin.
2. Create the worktree /home/agent/worktrees/add-search on a new branch
   add-search from origin/main, and follow prompts/implement-search.txt there.
3. Create the worktree /home/agent/worktrees/add-notif on a new branch
   add-notif from origin/main, and follow prompts/implement-notifications.txt there.

Use a separate subagent for each feature so they run in parallel. When a
feature's tests pass, commit it on its branch. Report both branch names when done.
```

Both branches start from `origin/main` (your fork on GitHub), so they don't include
the `fix-bugs` commits. Each subagent works in its own worktree, so they read and write
completely separate copies of the code with no conflicts. You'll still see only one
entry in `sbx ls`.

> **Tip**: you can also hand Claude its task when the session starts. Put Claude's
> permission flag **first** — when the first argument after `--` is a prompt rather
> than a flag, it replaces the agent's default startup flags, and Claude would start
> without `--dangerously-skip-permissions`:
>
> ```bash
> sbx run quickstart-clone -- --dangerously-skip-permissions "$(cat prompts/implement-search.txt)"
> ```
>
> The `"$(cat ...)"` must be quoted or the prompt won't be passed correctly to the
> sandbox.

When both are done, review each branch and open PRs (make sure you are in `~/sbx-quickstart`):

```bash
git checkout main

git fetch sandbox-quickstart-clone

git diff main..sandbox-quickstart-clone/add-search

git checkout -b add-search sandbox-quickstart-clone/add-search

git push -u origin add-search

gh pr create --head add-search \
  --title "Implement issue search" \
  --body "Implement issue search"  
```

```bash
git diff main..sandbox-quickstart-clone/add-notif

git checkout -b add-notif sandbox-quickstart-clone/add-notif

git push -u origin add-notif

gh pr create --head add-notif \
  --title "Implement status change notifications" \
  --body "Implement status change notifications"
```

### Clean up

All three branches are pushed, so you can remove the clone-mode sandbox. Switch back
to `main` first — the rest of the guide uses the direct-mode `quickstart` sandbox:

```bash
git checkout main

sbx stop quickstart-clone
sbx rm quickstart-clone
```

`sbx rm` deletes the sandbox, its clone, and the `sandbox-quickstart-clone` remote.

---

## 10. Docker Compose inside the sandbox

Each sandbox has its own private Docker daemon. Claude can run `docker compose up`,
build images, and start containers — none of which appear in your host's `docker ps`.

If necessary, reconnect to your sandbox:

```bash
sbx run quickstart
```

Give Claude the following prompt:

```
Start the full application stack using Docker Compose.
Once everything is healthy:
1. Confirm the backend API is responding at http://localhost:8000/health
2. Create a test user via the API
3. Create a project and two issues for that user
4. Report the final state (user, project, issues) as JSON

Use the API docs at http://localhost:8000/docs if helpful.
Make sure all servers bind to 0.0.0.0 so I can reach them via port forwarding.
```

Claude will `docker compose up --build -d`, wait for the `db` healthcheck to pass,
then hit the API endpoints with `curl` or `httpx`.

You can see the running containers in the sandbox. 

```bash
! docker ps
```

The containers Claude starts live entirely inside the sandbox. When you `sbx rm` the
sandbox, all images, containers, and Postgres data are deleted automatically.

---

## 11. Port forwarding with `sbx port`

Sandboxes are network-isolated — your browser can't reach a server inside one by
default. `sbx port publish` punches a hole from a host port to a sandbox port.

> **Terminal note**: `sbx port` is a host-side command. Run it in a new terminal tab
> while Claude is running

> **Note**: v0.54.0 replaced `sbx ports` with `sbx port publish`, `sbx port ls`, and
> `sbx port unpublish`. The old `sbx ports` command still works but is deprecated.

### Forward the API

With the Docker Compose stack running from the previous section:

```bash
sbx port publish quickstart 8080:8000
```

Open `http://localhost:8080/docs` — that's the live Swagger UI running inside the sandbox.

### Forward the frontend

```bash
sbx port publish quickstart 3001:3000
```

Open `http://localhost:3001` in your web browser to see the Web front-end

> **Tip**: leave out the host port (`sbx port publish quickstart 3000`) and the OS
> picks a free one. Use `sbx port ls` to see which port you got.

### Check active ports

```bash
sbx port ls quickstart
```

`sbx ls` also shows each sandbox's mappings in its `PORTS` column, in the form
`127.0.0.1:8080->8000/tcp4`. Published ports bind to `127.0.0.1` on your host by
default, so only your machine can reach them.

### Stop forwarding

Remove a single mapping:

```bash
sbx port unpublish quickstart 8080:8000
```

Or give only the sandbox port to remove every host mapping for it:

```bash
sbx port unpublish quickstart 8000
```

### Publish when you create a sandbox

If you know the ports up front, publish them at creation time:

```bash
sbx run --publish 8080:8000 --name my-sandbox claude
```

`--publish` only applies when `sbx run` creates the sandbox. It's ignored when you
reattach to an existing sandbox — use `sbx port publish` for that.

> **Gotcha**: services inside the sandbox must bind to `0.0.0.0` (or `[::]`), not
> `127.0.0.1`. Most dev servers default to `127.0.0.1` — that's why the prompt in
> section 10 explicitly asks Claude to bind to `0.0.0.0`.

> **Good to know**: published ports survive restarts — `sbx` re-publishes them when
> the sandbox or the daemon restarts. An explicit host port (`8080:8000`) is reused;
> an OS-picked host port changes on every start, so check `sbx port ls` after a
> restart. Removing the sandbox releases its ports.

---

## 12. Network policies

Every sandbox routes outbound HTTP/HTTPS through a host-side proxy that enforces
access rules you define. There are three built-in postures:

| Policy      | Description                                                                          |
|-------------|--------------------------------------------------------------------------------------|
| Open        | All traffic allowed — no restrictions                                                |
| Balanced    | Default deny, with a broad allow-list covering AI APIs, npm, pip, GitHub, registries |
| Locked Down | Everything blocked; you explicitly allow what you need                               |

> **Terminal note**: all `sbx policy` commands run on the **host**, not inside a
> sandbox session.

### Inspect current rules

```bash
sbx policy ls
```

You will see a long list of allowed websites. 

### See what the sandbox is actually hitting

```bash
sbx policy log
```

[output placeholder]

### Allow additional hosts

```bash
# Allow one host
sbx policy allow network smtp.mailgun.org

# Allow multiple at once
sbx policy allow network "smtp.mailgun.org,api.sendgrid.com"

# Allow all npm and PyPI (useful if Claude can't install packages)
sbx policy allow network "*.npmjs.org,*.pypi.org,files.pythonhosted.org"
```

### Block a host

```bash
sbx policy deny network ads.example.com
```

### Reaching host services from inside the sandbox

If you have a service running on your host machine (e.g. a local Ollama instance or a database), you can't reach it via `localhost` from inside the sandbox — that resolves to the VM itself. Use `host.docker.internal` instead, and add a policy rule if needed:

```bash
sbx policy allow network localhost:11434
```

Then inside the sandbox, point your client at `host.docker.internal:11434`.

---

## 13. Multiple workspaces

You can mount additional directories into a sandbox alongside the primary workspace.
Useful patterns: a shared `libs/` repo the agent can reference, `docs/` alongside
`app/`, or frontend and backend as separate repos both mounted at once.

Workspaces are configured at creation time — you can't add mounts to an existing sandbox. That means we need to recreate `quickstart`.

> **Warning**: `sbx rm` permanently deletes the sandbox VM and everything inside it — installed packages, containers, and images. Your source files in `~/sbx-quickstart` are untouched, and the branches from section 9 are already pushed to your fork.

```bash
sbx stop quickstart
sbx rm quickstart
```

Now recreate it with both workspaces mounted:

```bash
sbx run --name=quickstart claude ~/sbx-quickstart/backend ~/sbx-quickstart/frontend:ro
```

- `~/sbx-quickstart/backend` — primary workspace (read/write); agent starts here
- `~/sbx-quickstart/frontend:ro` — mounted read-only; agent can read but not write

Both appear inside the sandbox at their exact host paths, so relative paths in error
messages match what you see locally.

### Cross-repo exercise

With both workspaces mounted, give Claude the following prompt:

```
The frontend's api.ts sends search requests to GET /issues/search, but the
backend currently returns 501 for that endpoint. (Assume you've already
implemented the backend fix in a previous session.)

Review the frontend search flow in the frontend/src directory and make sure
the error handling is user-friendly when the endpoint isn't available yet.
The frontend is read-only — propose the change but don't apply it.
```

This pattern is common in monorepo setups where you want Claude to understand the full
picture but only write to specific parts.

---

## 14. Debugging with `sbx exec`

`sbx exec` opens a shell (or runs a one-off command) inside a running sandbox. Always
run it from a host terminal — it's not something you type inside the Claude session.

```bash
sbx exec -it quickstart bash
```

From inside the sandbox you can inspect the environment Claude is working in:

```bash
docker ps                                          # what containers are running?
pip list | grep fastapi                            # what's installed?
curl -s http://localhost:8000/health | python3 -m json.tool   # is the API up?
```

Type `exit` to leave. The Claude session keeps running.

### Run a one-off command without opening a shell

`sbx exec` starts in the sandbox's primary workspace. After section 13 that's
`backend/`, so you can run the tests directly:

```bash
sbx exec -it quickstart bash -c "pytest tests/ -v --tb=short"
```

### Set a persistent environment variable

Variables set inside the sandbox don't survive a restart. Write them to
`/etc/sandbox-persistent.sh` to make them permanent for the sandbox's lifetime:

```bash
sbx exec quickstart bash -c \
  "echo 'export SMTP_HOST=smtp.mailgun.org' >> /etc/sandbox-persistent.sh"
```

> **Note**: `sbx exec` always runs in the foreground — detached mode (`-d`) isn't
> supported. This command returns as soon as the line is written.

The file is sourced on every login shell, so Claude will see the variable in
subsequent sessions.

---

## 15. Connect your IDE over SSH

Up to now you've watched Claude's edits from an editor running on your host. That
works, but it means your editor — and everything it launches — runs on your host
with your privileges: extensions, language servers, linters, debuggers, the
integrated terminal, and any `.vscode/tasks.json` task. All of it reads and runs
files that an autonomous agent can write.

> **Why this matters**: Docker's sandbox security docs list IDE configuration
> (`.vscode/tasks.json`, `.idea/` run configurations) among the files that can
> execute code when you open a project in an IDE. An agent with write access to
> your workspace can create or change those files. If your IDE runs inside the
> sandbox, those tasks run in the microVM, not on your machine.

`sbx` can make every sandbox reachable over SSH as `<sandbox-name>.sbx`. Point an
IDE that does remote development over SSH, such as VS Code or Cursor, at that host.
The editor window stays on your host. Its server, extensions, terminals, and tasks
run inside the sandbox, next to Claude:

```
Your machine
├── IDE window (UI only)          ← you type and read here
├── ssh client ──► ProxyCommand "sbx ssh proxy"
│                          │  local socket / named pipe
│                          ▼  (no network port, no SSH key)
├── sbx daemon             ← SSH ends here; requires an active `sbx login`
│
└── Sandbox (microVM)
    ├── IDE server + extensions   ← run here
    ├── Integrated terminal       ← runs here, under your network policy
    ├── Private Docker daemon
    └── /your/workspace           ← the same mount Claude uses
```

> **Status**: SSH access is marked **experimental** in the `sbx` CLI. It requires
> Docker Sandboxes **0.37.0 or later**. This section was checked against **v0.54.1**.
> Commands and behavior may change between releases.

### How the connection works

You don't need to manage keys, ports, or an SSH server:

- **No network port and no SSH key.** A `ProxyCommand` sends the SSH stream to
  the `sbx` daemon over a local socket (a Unix domain socket on macOS, a named pipe
  on Windows). The daemon accepts the connection only while you're signed in with
  `sbx login`. Authentication is tied to that login, not to a stored key.
- **No SSH server in the sandbox.** SSH terminates at the daemon, which then
  opens your session inside the sandbox.
- **No host identity leaks.** You log in as the sandbox image's default user
  (`agent`, home `/home/agent`). Your host username is never sent.
- **Stopped sandboxes start themselves.** Connecting to a stopped sandbox starts
  it. The sandbox must already exist, though.
- **Your environment stays on the host.** The sandbox doesn't receive environment
  variables from your SSH client.

### Step 1 — Check your version

```bash
sbx version
```

If it's older than 0.37.0, upgrade first:

```bash
brew upgrade docker/tap/sbx       # macOS
```

```powershell
winget upgrade Docker.sbx         # Windows
```

**Windows only**: check that the OpenSSH client is installed:

```powershell
ssh -V
```

If `ssh` isn't found, install it from an elevated PowerShell. `sbx` works with
both native Windows OpenSSH and the OpenSSH bundled with Git Bash.

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0
```

### Step 2 — Enable SSH access (one time)

From a host terminal:

```bash
sbx setup ssh
```

This command starts the `sbx` daemon if it isn't running and configures your
SSH client. It writes a managed config block and adds an `Include` for it to your
SSH config: `~/.ssh/config` on macOS, `%USERPROFILE%\.ssh\config` on Windows. The
block looks like this (paths depend on your installation):

```text
# >>> docker sandboxes (managed) >>>
Match originalhost "*.sbx,!*'*"
    User _default_user_
    ProxyCommand "sbx" ssh proxy '%n'
    IdentityAgent none
    IdentityFile /dev/null
    IdentitiesOnly yes
    ControlMaster no
    ControlPath none
    UserKnownHostsFile "~/.ssh/sbx_known_hosts"
    KnownHostsCommand "sbx" ssh known-hosts %H
    StrictHostKeyChecking yes
# <<< docker sandboxes (managed) <<<
```

Don't edit this block by hand. Some details worth knowing:

- `Match originalhost "*.sbx,..."` applies to any hostname ending in `.sbx`.
  Because it's a `Match` block, not a `Host` entry, **IDE host pickers won't list
  your sandboxes**. Type the hostname yourself.
- `IdentityAgent none` and `IdentityFile /dev/null` mean your personal SSH keys
  are never offered to the sandbox connection.
- `KnownHostsCommand` checks the host key on every connection, so if the daemon
  rotates its key you won't get a host-key mismatch.

> **Important**: rerun `sbx setup ssh` after **every** `sbx` upgrade. v0.54.0
> hardened the generated config, so configs written by older versions must be
> regenerated. The command is idempotent, so running it again is safe.

### Step 3 — Connect from a terminal first

Test SSH from a plain terminal before you add an IDE. Then, if something fails,
you'll know whether the problem is SSH or the editor.

Confirm the sandbox exists. If you followed section 13, that's the recreated
`quickstart` with `backend` (read/write) and `frontend` (read-only) mounted:

```bash
sbx ls
```

Connect:

```bash
ssh quickstart.sbx
```

You're now in a shell inside the microVM. An interactive SSH shell starts in the
sandbox's **primary workspace**, just like `sbx exec`. Look around:

```bash
pwd          # the primary workspace path — write this down, you'll open it in your IDE
whoami       # agent — the sandbox user, not your host account
echo $HOME   # /home/agent — not your host home directory
docker ps    # the sandbox's private Docker daemon, not your host's
```

Type `exit` to leave. Like `sbx exec`, closing SSH doesn't stop the sandbox or
Claude's session.

You can also run a single command without opening a shell. This one prints the
microVM's own kernel, not your host's:

```bash
ssh quickstart.sbx -- uname -a
```

> **Note**: `.sbx` names exist only in your SSH config, not in DNS. They work in
> tools that use your OpenSSH config (`ssh`, VS Code, Cursor). They don't work in
> a browser or with `ping`.

### Step 4 — Connect VS Code

**Install the Remote - SSH extension** (`ms-vscode-remote.remote-ssh`) from the
Extensions view, or from a terminal:

```bash
code --install-extension ms-vscode-remote.remote-ssh
```

**Connect to the sandbox:**

1. Open the Command Palette (`Cmd+Shift+P` on macOS, `Ctrl+Shift+P` on Windows)
   and run **Remote-SSH: Connect to Host...**
2. Type `quickstart.sbx` and press `Enter`. It isn't in the list (see Step 2),
   so you have to type it.
3. If VS Code asks for the remote platform, choose **Linux**.
4. On the first connection, VS Code installs its server inside the sandbox,
   which takes a moment. Later connections are faster.

The remote indicator in the bottom-left corner now shows **SSH: quickstart.sbx**.

**Open the workspace folder:**

VS Code connects to the sandbox but may not open your project. It often opens
the sandbox user's home directory, `/home/agent`. Choose the folder yourself:

1. **File > Open Folder...**
2. Enter the path that `pwd` printed in Step 3. For this guide on macOS, that's
   `/Users/<you>/sbx-quickstart/backend`. Workspaces are mounted at their host
   path.
3. Click **OK**.

> **Gotcha**: don't type `~/sbx-quickstart/...` in the remote folder picker. Inside
> the sandbox, `~` is `/home/agent`, not your host home directory. Use the absolute
> path from `pwd`.

> **Tip (macOS)**: if the `code` command is on your PATH, open the project in
> the sandbox in one step from a host terminal. Your host shell expands `$HOME`
> to `/Users/<you>`, which is also the path the sandbox uses:
>
> ```bash
> code --remote ssh-remote+quickstart.sbx "$HOME/sbx-quickstart/backend"
> ```

### Step 5 — Prove you're inside the sandbox

Open a terminal in VS Code (**Terminal > New Terminal**) and run:

```bash
echo $HOME     # /home/agent
docker ps      # sandbox containers, not your host's
```

This terminal is a shell inside the microVM. Everything you run in it is subject
to the sandbox's network policy (section 12), the same as Claude.

**Work side by side with Claude.** In a host terminal, attach to the agent:

```bash
sbx run quickstart
```

Give Claude the following prompt:

```
Add a module-level docstring to app/models.py that summarizes each model
and its relationships. Don't change any code.
```

Open `app/models.py` in VS Code and watch the docstring appear. Claude and your
IDE are now both working inside the same sandbox on the same files. Nothing your
editor runs touches your host.

**See read-only mounts apply to your IDE too:**

1. **File > Add Folder to Workspace...** → `/Users/<you>/sbx-quickstart/frontend`
2. Open any file under `frontend/src`, change it, and save.

The save fails with a read-only file system error. The `:ro` mount from section 13
applies to everything in the sandbox, and your IDE is now in the sandbox. Undo the
change and close the file.

### Step 6 — Forward ports through SSH

Section 11 published sandbox ports with `sbx port publish`. That only works when
the server binds to `0.0.0.0`. SSH port forwarding is different: it connects to the
sandbox's **loopback** interface, so a dev server on `127.0.0.1` works without
changes.

In the VS Code terminal, start a throwaway server that listens **only** on
loopback:

```bash
python3 -m http.server 8765 --bind 127.0.0.1
```

VS Code usually detects the port and forwards it for you. If it doesn't, open the
**Ports** panel, click **Forward a Port**, and enter `8765`. Then open the
**Forwarded Address** shown in the panel in your host browser. VS Code may choose
a different host port if `8765` is already taken. Press `Ctrl-C` in the terminal to
stop the server.

The same works without an IDE, from a host terminal:

```bash
ssh -N -L 8765:127.0.0.1:8765 quickstart.sbx
```

| | `sbx port publish` (section 11) | SSH forwarding |
|---|---|---|
| Service must bind to | `0.0.0.0` | `127.0.0.1` / `localhost` / `::1` |
| Lasts | Until you unpublish it or remove the sandbox (re-published on restart) | Until the SSH connection closes |
| Best for | Long-running previews, sharing a stable port | Day-to-day IDE work |

> **Gotcha**: the daemon forwards only to loopback addresses **inside** the sandbox.
> Your SSH client chooses where the **host** side listens. Keep it on `127.0.0.1`.
> If you set your client to listen on a non-loopback address, other machines on
> your network may be able to reach the forwarded service.

When you're done, close the remote window (**File > Close Remote Connection**).
The sandbox keeps running.

### Other editors and apps

- **Cursor** is built on VS Code and connects the same way. Run
  **Remote-SSH: Connect to Host**, type `quickstart.sbx`, and open the workspace
  path from Step 3.
- **Other SSH-capable tools.** Once SSH is set up, `<name>.sbx` works like any
  other SSH host, so any tool that does remote development over your OpenSSH
  config can connect. Remember to type the hostname yourself.
- **Claude Desktop and ChatGPT** (macOS and Windows, v0.48.0 or later). Launch
  each app once, quit them, then run `sbx setup ssh --auto-create`. This adds a
  *Docker Sandbox (Claude)* connection (`claude.sbx`) and a *Docker Sandbox
  (Codex)* connection (`codex.sbx`). Each creates its own sandbox the first time
  you use it (`ssh-claude` and `ssh-codex`). These sandboxes have **no host
  workspace**: clone your project inside them and push your changes to get them
  out.
- **Cloud sandboxes** use a separate alias. Run `sbx --cloud setup ssh`, then
  `ssh <sandbox-id>@sbx_cloud` with the `sbx_`-prefixed ID from `sbx --cloud ls`.

### What SSH isolates — and what it doesn't

| | IDE on your host (sections 8–14) | IDE over SSH |
|---|---|---|
| Extensions, language servers, linters | Run on your host | Run in the sandbox |
| Terminal, tasks, debugger | Run on your host, with no network policy | Run in the sandbox, with your network policy |
| Your SSH keys and host home directory | Available to whatever the IDE runs | Not offered or mounted |
| Workspace files (direct mode) | Your host files | **Still your host files**, through the same mount |

Some things still cross the boundary:

- **Files.** SSH moves *execution* into the sandbox. It doesn't change the
  workspace mount. In direct mode, every edit made by you, your IDE, or Claude
  lands on your host disk immediately. To isolate the files too, see the next
  subsection.
- **Forwarded ports.** Each forwarded port opens a listener on your host.
- **Your SSH agent.** SSH agent forwarding into sandboxes is on by default, so
  Git over SSH and commit signing work. Your private keys stay on the host, but
  processes in the sandbox can ask your agent to sign. If you don't need that,
  turn it off:

  ```bash
  sbx settings set ssh.agentForwardingEnabled false
  sbx daemon restart
  ```

- **Extension and server downloads.** These go through your network policy. If
  the first connection hangs or extensions won't install, run `sbx policy log`
  and allow the VS Code hosts that were blocked (for example
  `update.code.visualstudio.com` or `marketplace.visualstudio.com`).

### Go further: isolate the files too (clone mode)

For the strongest setup, combine SSH with **clone mode** from section 9. Your repo
is mounted read-only, and the sandbox works on its own Git clone. Nothing reaches
your host working tree until you fetch it.

Clone mode is set when you create a sandbox, so create a fresh one (you removed
`quickstart-clone` at the end of section 9). Run this from your main checkout, not
from a worktree:

```bash
cd ~/sbx-quickstart
sbx create --clone --name quickstart-ide claude .
ssh quickstart-ide.sbx
```

Inside the sandbox, run `pwd`. The SSH shell starts in the primary workspace,
which in clone mode is the sandbox's own clone. Note the path, then `exit`.

Connect VS Code to `quickstart-ide.sbx` and open that path. Create a branch,
make an edit, and commit in the VS Code terminal:

```bash
git checkout -b ide-edits
git commit -am "Edit made from VS Code inside the sandbox"
```

> **Note**: if Git asks who you are, run `git config user.name "..."` and
> `git config user.email "..."` in the same terminal and commit again.

Back on the host, your working tree hasn't changed. Fetch the branch to review
it. The sandbox must be running:

```bash
git fetch sandbox-quickstart-ide
git diff main..sandbox-quickstart-ide/ide-edits
```

When you're done, remove the sandbox. This **deletes the clone**, so fetch or push
anything you want to keep first:

```bash
sbx rm quickstart-ide
```

### Turning SSH access off

To remove the managed SSH config and any desktop app connections:

```bash
sbx setup ssh remove
```

Your sandboxes aren't affected. Remove them separately with `sbx rm` if you no
longer need them.

---

## 16. Custom templates

> **Note**: This section is for reference only. Building custom templates requires Docker Desktop (or another Docker daemon) installed on your host machine — it is not something you do inside the sandbox.

Docker publishes sandbox templates as plain Docker images (`docker/sandbox-templates:<variant>`). You can extend one to pre-bake toolchains, language runtimes, config files, or any other dependencies your project needs. Claude won't have to install them at the start of every session.

> **Good to know**: built-in agents such as `claude` now run from Debian-based kits
> published at `docker.io/sbx/<name>:latest`. The Ubuntu-based
> `docker/sandbox-templates` images are older, separate images that still work as
> a `--template` base. For adding tools to a built-in agent, Docker now recommends
> kit mixins — see [Base images](https://docs.docker.com/ai/sandboxes/customize/author/base-images/)
> and [Customize sandboxes](https://docs.docker.com/ai/sandboxes/customize/).

### Create a Dockerfile

Start `FROM` an existing sandbox template and layer your additions on top. Switch between `root` (for system packages) and `agent` (for user-level tools) as needed:

```dockerfile
FROM docker/sandbox-templates:claude-code

USER root
RUN apt-get update && apt-get install -y protobuf-compiler

USER agent
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
```

> **Gotcha**: install agent-level tools as `agent`, not `root`. Installers run as
> `root` put files under `/root/`, where the agent can't use them.

### Build and push

```bash
docker build -t my-org/my-template:v1 --push .
```

Push to any registry Docker can pull from — Docker Hub, GHCR, ECR, etc.

For **private** images, Docker Hub pulls reuse your `sbx login` session. Other
registries need pull credentials stored first:

```bash
gh auth token | sbx secret set --registry ghcr.io --password-stdin
```

To skip the registry entirely, load a locally built image straight into the sandbox
runtime:

```bash
docker image save my-org/my-template:v1 -o my-template.tar
sbx template load my-template.tar
sbx run --pull never -t my-org/my-template:v1 claude
```

### Use your template

```bash
sbx run --template docker.io/my-org/my-template:v1 claude
```

Unlike `docker`, `sbx` doesn't add `docker.io` to image references for you, so always
use the full reference.

When `sbx` creates a sandbox from a registry template, it pulls the image by default,
so new sandboxes pick up an updated image pushed to the same tag. Existing sandboxes
keep the image they were created with — recreate them to get the update. Using a new
tag for each change (`v1`, `v2`, …) keeps this explicit.

> **Warning**: don't use `sbx reset` to refresh a template. It resets Docker
> Sandboxes to a freshly installed state: it deletes **all** sandboxes, stored
> secrets, policy rules, and the managed SSH config, and signs you out.

### Save a sandbox as a template (no Docker required)

If you've already set up a sandbox the way you like it, snapshot its container
filesystem instead of writing a Dockerfile:

```bash
sbx template save quickstart my-template:v1
sbx run -t my-template:v1 claude
```

`sbx` asks to stop the sandbox if it's running. The snapshot doesn't include mounted
workspaces or the sandbox's Docker images (`/var/lib/docker`). It **does** include
any credentials written to the sandbox filesystem, so keep secrets in `sbx secret`,
not in files, before you share a template.

### Available base images

| Image | Includes | Status |
|-------|----------|--------|
| `docker/sandbox-templates:claude-code` | Claude Code, standard toolchain | Rebuilt by Docker |
| `docker/sandbox-templates:claude-code-minimal` | Claude Code, minimal toolset (no Node.js, Python, Go, or Java) | Rebuilt by Docker |
| `docker/sandbox-templates:claude-code-docker` | Claude Code + Docker Engine | Rebuilt by Docker |
| `docker/sandbox-templates:gemini`, `gemini-docker` | Gemini CLI (`-docker` adds Docker Engine) | Rebuilt by Docker |
| `docker/sandbox-templates:codex`, `copilot`, `cursor-agent`, `devin`, `docker-agent`, `droid`, `kiro`, `opencode` | Other agents | Previously published; no longer rebuilt |
| `docker.io/sbx/shell-image:latest` | Development environment, no agent pre-installed | Current |
| `docker.io/sbx/shell-docker-image:latest` | Same, plus Docker Engine | Current |

The `docker/sandbox-templates` images are based on Ubuntu and run as a non-root `agent` user with sudo access. Most variants include Git, the Docker CLI, Node.js, Python, Go, and Java; full templates also include a C/C++ compiler toolchain.

---

## Appendix A: Prompt library

These prompts are designed to work well with Claude in a Docker Sandbox.

---

### Understand a new codebase

```
Explore this repository and produce a technical overview covering:
1. Architecture and tech stack (include the full dependency list)
2. How data flows through the system end to end
3. How to run the project and its test suite locally
4. Any patterns or conventions the team seems to follow
5. The top 3 areas you'd investigate first if you were debugging a production incident

Be specific — cite filenames and line numbers where relevant.
```

---

### Run tests and triage failures

```
Run the full test suite for the [backend/frontend] and produce a triage report:
- List every failing test with its full error output
- For each failure, identify whether it's a test bug, a code bug, or a missing feature
- Propose a fix for each code bug (don't implement yet — just describe the approach)
- Estimate the effort to fix each issue (small / medium / large)
```

---

### Fix a specific bug

```
Fix the bug exposed by [test name].

Constraints:
- Change only the minimal code needed to fix this specific bug
- Do not refactor unrelated code
- After each change, re-run [test name] to verify it passes
- When it passes, run the full suite to check for regressions
- Explain the root cause in a comment above the fix
```

---

### Implement a feature

```
Implement [feature description] in [file/module].

Requirements:
[paste the requirements or link to a spec]

Done criteria:
- Feature works end-to-end (manually demonstrate it with curl or a test)
- Tests cover the happy path and at least two edge cases
- No existing tests are broken
- Code follows the patterns already in the file
```

---

### Set up and run a service with Docker Compose

```
Start the full application stack with Docker Compose.

1. Run `docker compose up --build -d` and wait for all services to be healthy
2. Confirm each service is running with `docker compose ps`
3. Hit the health endpoint for each API service and report the response
4. If any service fails to start, show the full logs and diagnose the issue
5. Once everything is healthy, run a quick smoke test:
   - Create a user
   - Authenticate
   - Create a resource and retrieve it
   Report the responses as JSON.

Make sure any server you start binds to 0.0.0.0 (not 127.0.0.1) so I can
reach it via sbx port publish.
```

---

### Code review

```
Review the changes in [branch or file] as if this were a pull request.

Focus on:
1. Correctness — are there any bugs, edge cases, or logic errors?
2. Security — are there any injection risks, missing auth checks, or exposed secrets?
3. Performance — any obvious N+1 queries, blocking I/O, or unnecessary work?
4. Test coverage — are the important paths tested?
5. Code quality — readability, naming, consistency with the surrounding code

For each issue, provide: severity (P0/P1/P2), location (file + line), description,
and a suggested fix.
```

---

### Document a module

```
Write docstrings and inline comments for all public functions and classes in [file].

Rules:
- Follow the existing docstring style in the codebase
- Explain *why*, not just *what* — especially for non-obvious logic
- Add type hints to any function that's missing them
- Do not change any logic — documentation only
```

---

## Appendix B: CLI quick reference

```bash
# ── Lifecycle ──────────────────────────────────────────────────────────────────
sbx run --name=quickstart claude         # create and attach to a named sandbox
sbx run quickstart                       # reconnect to an existing sandbox
sbx create [AGENT] [WORKSPACE]           # create without attaching
sbx create --clone --name=quickstart-clone claude .   # clone mode — private Git clone
sbx run quickstart-clone -- --dangerously-skip-permissions \
  "$(cat p.txt)"                         # start with a prompt (flag first; quotes required)
git fetch sandbox-quickstart-clone       # fetch clone-mode commits  [host terminal]
sbx ls                                   # list sandboxes
sbx stop quickstart                      # pause (preserves installed packages)
sbx rm quickstart                        # delete sandbox + VM (clone mode: also its clone)

# ── Attach & shell ─────────────────────────────────────────────────────────────
sbx exec -it quickstart bash             # shell inside sandbox  [host terminal]
sbx exec quickstart bash -c "cmd"        # one-off command        [host terminal]

# ── SSH & IDE (experimental, v0.37.0+) ─────────────────────────────────────────
sbx setup ssh                            # one-time SSH config for <name>.sbx (rerun after upgrades)
ssh quickstart.sbx                       # shell inside sandbox over SSH
ssh quickstart.sbx -- uname -a           # one-off command over SSH
ssh -N -L 8765:127.0.0.1:8765 quickstart.sbx   # forward a sandbox loopback port
code --remote ssh-remote+quickstart.sbx /abs/path/to/workspace   # VS Code in sandbox
sbx setup ssh --auto-create              # Claude Desktop / ChatGPT connections (macOS/Windows)
sbx setup ssh remove                     # remove managed SSH config; keeps sandboxes
sbx --cloud setup ssh                    # cloud sandboxes → ssh <sandbox-id>@sbx_cloud

# ── Port forwarding ────────────────────────────────────────────────────────────
sbx port publish quickstart 8080:8000     # host:8080 → sandbox:8000
sbx port publish quickstart 3000          # OS picks host port
sbx port ls quickstart                    # show published ports
sbx port unpublish quickstart 8080:8000   # remove one mapping
sbx port unpublish quickstart 8000        # remove every mapping for sandbox port 8000

# ── Network policies ───────────────────────────────────────────────────────────
sbx policy ls                                       # list active rules
sbx policy log                                      # show connection log
sbx policy log quickstart                           # filter by sandbox
sbx policy allow network example.com                # allow a host
sbx policy allow network "*.npmjs.org,*.pypi.org"    # allow multiple
sbx policy deny network ads.example.com             # block a host
sbx policy rm network --resource example.com        # remove a rule
sbx policy reset                                    # wipe rules, restart daemon, reprompt
sbx policy init balanced                            # set global policy without prompt (CI)

# ── Credentials ────────────────────────────────────────────────────────────────
gh auth token | sbx secret set github    # store GitHub token (global by default)
sbx secret set anthropic --sandbox quickstart   # scope to one sandbox
sbx secret ls                            # list stored secrets
sbx secret rm github                     # remove a global secret

# ── Dashboard ──────────────────────────────────────────────────────────────────
sbx                                      # open interactive TUI  [host terminal]

# ── Reset ──────────────────────────────────────────────────────────────────────
sbx reset                                # ⚠ factory reset: deletes ALL sandboxes, secrets,
                                         #   policies, SSH config; signs you out
```

---

## Appendix C: Troubleshooting

### Agent can't install packages or reach an API

The network policy is blocking the domain. Check what's blocked:

```bash
sbx policy log
```

Then allow the domains you need:

```bash
sbx policy allow network "*.npmjs.org,*.pypi.org,files.pythonhosted.org,github.com"
```

Or allow everything temporarily:

```bash
sbx policy allow network "**"
```

---

### `You are not authenticated to Docker`

Your login session expired. Re-authenticate:

```bash
sbx login
```

---

### Agent can't reach the model provider

1. Check that `api.anthropic.com` is in the allow list: `sbx policy log`
2. If you're in Locked Down mode, explicitly allow it:
   ```bash
   sbx policy allow network api.anthropic.com
   ```
3. If you use an API key rather than `/login`, confirm it's stored with
   `sbx secret ls`. Secrets take effect in existing sandboxes without a restart, so
   you don't need to recreate the sandbox after adding one.

---

### Port forwarding isn't working

- Confirm the service binds to `0.0.0.0`, not `127.0.0.1`
- If you published without a host port, the host port changes on every start — check `sbx port ls quickstart`
- Check the sandbox is still running: `sbx ls`
- Make sure you're running `sbx port` from a host terminal, not from inside the sandbox session

---

### Sandbox clock is drifting (requests failing with timestamp errors)

Putting your laptop to sleep and waking it can cause the sandbox VM's clock to drift from the host. This shows up as expired token errors, TLS failures, or other time-sensitive request failures.

Fix it by stopping and restarting the sandbox:

```bash
sbx stop quickstart
sbx run quickstart
```

---

### Docker not available inside the sandbox (Windows)

Use the `-docker` template variant:

```bash
sbx run --template docker.io/docker/sandbox-templates:claude-code-docker --name=quickstart claude
```

---

### `ssh: Could not resolve hostname quickstart.sbx`

Your SSH client isn't using the managed `sbx` config. Generate it:

```bash
sbx setup ssh
```

Also check that:

- The sandbox name matches `sbx ls` exactly. The hostname is `<name>.sbx`.
- You're using an OpenSSH client that reads your user SSH config (`~/.ssh/config`
  on macOS, `%USERPROFILE%\.ssh\config` on Windows).

---

### SSH stopped working after upgrading `sbx`

Each release can change the generated SSH config, and v0.54.0 requires a refresh.
Regenerate it:

```bash
sbx setup ssh
```

---

### SSH connection is rejected

The daemon accepts SSH connections only while you have an active Docker login, and
only to sandboxes that already exist:

```bash
sbx login
sbx ls
```

---

### VS Code: `Host key verification failed` or a `KnownHostsCommand` error

VS Code can leave duplicate or broken managed blocks in your SSH config. Open
`~/.ssh/config` (macOS) or `%USERPROFILE%\.ssh\config` (Windows) and delete **every**
block between `# >>> docker sandboxes (managed) >>>` and
`# <<< docker sandboxes (managed) <<<`, including both marker lines. Then
regenerate it and reconnect:

```bash
sbx setup ssh
```

---

### VS Code keeps reconnecting (macOS)

Some VS Code versions get stuck in a reconnect loop on macOS. Add this to your
VS Code **user** settings (`settings.json`):

```json
{
  "remote.SSH.useLocalServer": false
}
```

See [microsoft/vscode-remote-release#11672](https://github.com/microsoft/vscode-remote-release/issues/11672).

---

### VS Code hangs on first connect, or extensions won't install in the sandbox

The VS Code server and extensions download from inside the sandbox, so your
network policy applies to them. Find the blocked hosts and allow them:

```bash
sbx policy log quickstart
sbx policy allow network update.code.visualstudio.com
```

---

### VS Code opens `/home/agent` instead of my project

This is expected. Editors often open the sandbox user's home directory. Use
**File > Open Folder...** and enter the absolute workspace path. To find it,
open an interactive SSH shell, which starts in the primary workspace, and run
`pwd`:

```bash
ssh quickstart.sbx
pwd
```
