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
11. [Port forwarding with `sbx ports`](#11-port-forwarding-with-sbx-ports)
12. [Network policies](#12-network-policies)
13. [Multiple workspaces](#13-multiple-workspaces)
14. [Debugging with `sbx exec`](#14-debugging-with-sbx-exec)
15. [Custom templates](#15-custom-templates)
16. [Appendix A: Prompt library](#appendix-a-prompt-library)
17. [Appendix B: CLI quick reference](#appendix-b-cli-quick-reference)
18. [Appendix C: Troubleshooting](#appendix-c-troubleshooting)

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

## 11. Port forwarding with `sbx ports`

Sandboxes are network-isolated — your browser can't reach a server inside one by
default. `sbx ports` punches a hole from a host port to a sandbox port.

> **Terminal note**: `sbx ports` is a host-side command. Run it in a new terminal tab
> while Claude is running

### Forward the API

With the Docker Compose stack running from the previous section:

```bash
sbx ports quickstart --publish 8080:8000
```

Open `http://localhost:8080/docs` — that's the live Swagger UI running inside the sandbox.

### Forward the frontend

```bash
sbx ports quickstart --publish 3001:3000
```

Open `http://localhost:3001` in your web browser to see the Web front-end


### Check active ports

```bash
sbx ports quickstart

HOST IP     HOST PORT   SANDBOX PORT   PROTOCOL
127.0.0.1   3001        3000           tcp
127.0.0.1   8080        8000           tcp
```

### Stop forwarding

```bash
sbx ports quickstart --unpublish 8080:8000
```

> **Gotcha**: services inside the sandbox must bind to `0.0.0.0`, not `127.0.0.1`.
> Most dev servers default to `127.0.0.1` — that's why the prompt in section 10
> explicitly asks Claude to bind to `0.0.0.0`.

> **Gotcha**: published ports don't survive a sandbox stop/restart. Re-run `sbx ports`
> after restarting.

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

```bash
sbx exec -it quickstart bash -c "cd backend && pytest tests/ -v --tb=short"
```

### Set a persistent environment variable

Variables set inside the sandbox don't survive a restart. Write them to
`/etc/sandbox-persistent.sh` to make them permanent for the sandbox's lifetime:

```bash
sbx exec -d quickstart bash -c \
  "echo 'export SMTP_HOST=smtp.mailgun.org' >> /etc/sandbox-persistent.sh"
```

The file is sourced on every login shell, so Claude will see the variable in
subsequent sessions.

---

## 15. Custom templates

> **Note**: This section is for reference only. Building custom templates requires Docker Desktop (or another Docker daemon) installed on your host machine — it is not something you do inside the sandbox.

Every built-in agent template (`claude-code`, `codex`, `gemini`, etc.) is a plain Docker image. You can extend any of them to pre-bake toolchains, language runtimes, config files, or any other dependencies your project needs. Claude won't have to install them at the start of every session.

### Create a Dockerfile

Start `FROM` an existing sandbox template and layer your additions on top. Switch between `root` (for system packages) and `agent` (for user-level tools) as needed:

```dockerfile
FROM docker/sandbox-templates:claude-code

USER root
RUN apt-get update && apt-get install -y protobuf-compiler

USER agent
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
```

### Build and push

```bash
docker build -t my-org/my-template:v1 --push .
```

Push to any registry Docker can pull from — Docker Hub, GHCR, ECR, etc.

### Use your template

```bash
sbx run --template docker.io/my-org/my-template:v1 claude
```

The image is pulled and cached locally on first use. Subsequent sandbox starts reuse the cached image. If you update the image and want the new version, run `sbx reset` to clear the cache, or use a new tag.

### Available base images

| Template | Includes |
|----------|----------|
| `docker/sandbox-templates:claude-code` | Claude Code, standard toolchain |
| `docker/sandbox-templates:claude-code-docker` | Claude Code + Docker Engine (required for Docker Compose on Windows) |
| `docker/sandbox-templates:codex` | OpenAI Codex |
| `docker/sandbox-templates:gemini` | Gemini CLI |
| `docker/sandbox-templates:shell` | Bare Bash — no agent pre-installed |

All base images include Ubuntu, Git, GitHub CLI, Node.js, Go, Python 3, and common package managers.

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
reach it via sbx ports forwarding.
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
sbx exec -d quickstart bash -c "cmd"     # one-off command        [host terminal]

# ── Port forwarding ────────────────────────────────────────────────────────────
sbx ports quickstart --publish 8080:8000   # host:8080 → sandbox:8000
sbx ports quickstart --publish 3000        # OS picks host port
sbx ports quickstart                       # show active forwarding rules
sbx ports quickstart --unpublish 8080:8000 # stop forwarding

# ── Network policies ───────────────────────────────────────────────────────────
sbx policy ls                                       # list active rules
sbx policy log                                      # show connection log
sbx policy log quickstart                           # filter by sandbox
sbx policy allow network example.com                # allow a host
sbx policy allow network "*.npmjs.org,*.pypi.org"    # allow multiple
sbx policy deny network ads.example.com             # block a host
sbx policy rm network --resource example.com        # remove a rule
sbx policy reset                                    # wipe rules, reprompt for policy
sbx policy set-default balanced                     # set default without prompt (CI)

# ── Credentials ────────────────────────────────────────────────────────────────
gh auth token | sbx secret set github    # store GitHub token (global by default)
sbx secret set anthropic --sandbox quickstart   # scope to one sandbox
sbx secret ls                            # list stored secrets
sbx secret rm github                     # remove a global secret

# ── Dashboard ──────────────────────────────────────────────────────────────────
sbx                                      # open interactive TUI  [host terminal]

# ── Reset ──────────────────────────────────────────────────────────────────────
sbx reset                                # stop all VMs, delete all sandbox data
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
- Published ports don't survive a restart — re-run `sbx ports` after `sbx stop`/`sbx run`
- Check the sandbox is still running: `sbx ls`
- Make sure you're running `sbx ports` from a host terminal, not from inside the sandbox session

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
