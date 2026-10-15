# Eventyay Local Development Setup Guide

A complete, step-by-step guide to cloning, configuring, running, and developing on **Eventyay** locally (especially Windows), including automation instructions for **Antigravity CLI (`agy`)** and your daily development routine.

---

## 📋 Table of Contents
1. [Prerequisites](#1-prerequisites)
2. [First-Time Setup (Step-by-Step)](#2-first-time-setup-step-by-step)
   - [Clone Fork & Configure Upstream](#step-21-clone-your-fork--configure-upstream)
   - [Configure Environment File](#step-22-configure-the-environment-file)
   - [Windows Critical Fix: Line Endings (CRLF to LF)](#step-23-windows-critical-fix-line-endings-crlf-to-lf)
   - [Start Docker Stack](#step-24-start-the-docker-stack)
   - [Create an Admin User](#step-25-create-an-admin-user)
   - [Verify Access](#step-26-verify-access)
3. [🔄 Regular Day-to-Day Development Routine](#3--regular-day-to-day-development-routine)
   - [Daily Start & Stop Commands](#31-daily-start--stop-commands)
   - [Live Reloading (No Rebuilds Needed)](#32-live-reloading-no-rebuilds-needed)
   - [When to Rebuild vs When Not to](#33-when-to-rebuild-vs-when-not-to)
   - [Running Tests](#34-running-tests)
   - [Daily Git Feature Branch Workflow](#35-daily-git-feature-branch-workflow)
4. [🚀 Quick Setup with Antigravity CLI (`agy`)](#4--quick-setup-with-antigravity-cli-agy)
5. [Common Commands Cheat Sheet](#5-common-commands-cheat-sheet)
6. [Troubleshooting & Windows Gotchas](#6-troubleshooting--windows-gotchas)

---

## 1. Prerequisites

Before starting on the target computer, ensure the following are installed:

| Tool | Recommended Version | Download / Installation |
| :--- | :--- | :--- |
| **Git** | 2.40+ | [git-scm.com](https://git-scm.com/) |
| **Docker Desktop** | Latest (WSL2 Backend) | [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/) |
| **GitHub CLI (`gh`)** | Latest | `winget install --id GitHub.cli` or [cli.github.com](https://cli.github.com/) |
| **Antigravity CLI (`agy`)** | Latest | Antigravity CLI installation |

> [!IMPORTANT]
> **Docker Desktop on Windows**: Make sure Docker Desktop is **launched and running** (showing green icon in system tray) before running docker commands. Ensure WSL2 integration is enabled in Docker Desktop Settings > Resources > WSL Integration.

---

## 2. First-Time Setup (Step-by-Step)

### Step 2.1: Clone Your Fork & Configure Upstream

Open PowerShell or terminal in your workspace directory (e.g. `C:\Projects`):

```powershell
# 1. Authenticate with GitHub (if not already authenticated)
gh auth login

# 2. Clone your personal fork
git clone https://github.com/<YOUR_GITHUB_USERNAME>/eventyay.git
cd eventyay

# 3. Add the official upstream repository
git remote add upstream https://github.com/fossasia/eventyay.git

# 4. Verify remotes
git remote -v
# Output should show:
# origin   https://github.com/<YOUR_USERNAME>/eventyay.git (fetch & push)
# upstream https://github.com/fossasia/eventyay.git (fetch & push)

# 5. Ensure you are on the dev branch and synchronized
git fetch upstream
git checkout dev
git pull upstream dev
```

---

### Step 2.2: Configure the Environment File

Eventyay provides a sample environment configuration for development:

```powershell
# Copy the sample development env file to .env.dev
Copy-Item deployment\env.dev.sample .env.dev
```

*(On macOS/Linux, use: `cp deployment/env.dev.sample .env.dev`)*

---

### Step 2.3: Windows Critical Fix: Line Endings (CRLF to LF)

> [!WARNING]
> **Crucial for Windows Users**:
> Git on Windows often checks out shell scripts with CRLF line endings (`\r\n`). In `docker-compose.yml`, `./app/entrypoint.sh` is mounted into the Linux container.
> If `entrypoint.sh` has CRLF line endings, the container will immediately crash with:
> `exec /usr/src/app/entrypoint.sh: no such file or directory`
>
> Run this PowerShell snippet to convert `app/entrypoint.sh` to clean Unix LF endings:

```powershell
$content = [System.IO.File]::ReadAllText("app\entrypoint.sh")
$content = $content.Replace("`r`n", "`n")
[System.IO.File]::WriteAllText("app\entrypoint.sh", $content, (New-Object System.Text.UTF8Encoding($false)))
```

---

### Step 2.4: Start the Docker Stack

Ensure **Docker Desktop** is running, then execute:

```powershell
docker compose up -d --build
```

#### What happens during the initial build:
1. uv installs Python 3.12 dependencies.
2. Static assets are collected (`collectstatic`).
3. Database migrations (PostgreSQL) are applied automatically.
4. Locales and gettext translations (`compilemessages`) are compiled.
5. Django SCSS/Stylus stylesheets (`updatestyles`) are generated.
6. Vite development servers for hot-reload frontends (`schedule-editor`, `video`, `schedule`) are started.
7. Django Daphne/ASGI server starts listening on port `8000`.

> [!NOTE]
> The initial build and migration process typically takes **5 to 8 minutes**. You can monitor progress with:
> ```powershell
> docker logs -f eventyay-next-web
> ```

---

### Step 2.5: Create an Admin User

Once the web server is running, create your administrator account.

#### Quick Non-Interactive Dev User (Recommended)
```powershell
docker exec -i eventyay-next-web python manage.py shell -c "from eventyay.base.models import User; u = User.objects.create_adminuser(email='admin@eventyay.local', password='EventyayAdmin123!'); print('Admin created:', u.email)"
```

---

### Step 2.6: Verify Access

Open your browser and navigate to:

- **Attendee / Public View**: [http://localhost:8000](http://localhost:8000)
- **Organizer / Admin Dashboard**: [http://localhost:8000/control/](http://localhost:8000/control/)
  - **Login Email**: `admin@eventyay.local`
  - **Password**: `EventyayAdmin123!`

---

## 3. 🔄 Regular Day-to-Day Development Routine

Because `app/eventyay` is **volume-mounted directly into the container**, you almost **never need to rebuild Docker** for everyday code changes.

### 3.1 Daily Start & Stop Commands

| Action | Command | What it does |
| :--- | :--- | :--- |
| **Start working (Morning)** | `docker compose up -d` *(or `docker compose start`)* | Starts all containers in 2-3 seconds. **No rebuild needed**. Your database data, users, and events are preserved. |
| **Pause / Stop working (Evening)** | `docker compose stop` | Pauses containers without deleting them. Next start is instantaneous. |
| **Clean stop (to free RAM)** | `docker compose down` | Stops and removes container instances, but **keeps database data** safely in named volumes. |

> [!CAUTION]
> **Golden Rule**: **NEVER run `docker compose down -v`** in your daily work! The `-v` flag removes the database volume, wiping all your local users, events, and test data.

---

### 3.2 Live Reloading (No Rebuilds Needed)

- **Backend & Templates** (`app/eventyay/**/*.py`, `*.html`):
  - The Django dev server runs with `StatReloader`.
  - The moment you save any file in your code editor, Django reloads automatically in 1–2 seconds. Simply refresh your browser tab.
- **Frontend Modules** (`app/eventyay/webapp/**`):
  - The Vite dev servers run with Hot-Module Replacement (HMR). Changes to Vue components reflect immediately in the browser.

---

### 3.3 When to Rebuild vs When Not to

| Scenario | What to run | Why |
| :--- | :--- | :--- |
| **Editing normal Python / HTML / JS** | *Nothing! Just save file.* | Mounted live into container. |
| **Pulled updates from upstream** (`git pull upstream dev`) | `docker compose up -d --build`<br>`docker exec -it eventyay-next-web python manage.py migrate` | If `pyproject.toml`, packages, or database migrations changed in upstream. |
| **Added or changed a Django model** | `docker exec -it eventyay-next-web python manage.py makemigrations`<br>`docker exec -it eventyay-next-web python manage.py migrate` | Creates and applies new database migrations. |
| **Server hung or needs quick reboot** | `docker restart eventyay-next-web` | Fast restart of web service only (~5 seconds). |
| **CSS changes not appearing** | `docker exec -it eventyay-next-web python manage.py compress --force` | Re-compiles compressed CSS stylesheets. |

---

### 3.4 Running Tests

Always run tests inside the container before committing changes:

```powershell
# Run the complete test suite
docker exec -it eventyay-next-web pytest tests/

# Run tests for a specific sub-folder (e.g. organizer speaker tests):
docker exec -it eventyay-next-web pytest tests/orga/
```

---

### 3.5 Daily Git Feature Branch Workflow

```powershell
# 1. Start your local containers
docker compose up -d

# 2. Make sure dev branch is in sync with upstream
git checkout dev
git pull upstream dev

# 3. Create a feature branch for your issue
git checkout -b feature/my-issue-fix

# 4. Make code edits in app/eventyay/ and test at http://localhost:8000

# 5. Run tests to ensure no regressions
docker exec -it eventyay-next-web pytest tests/

# 6. Commit with concise message and push to your fork
git add .
git commit -m "fix(orga): resolve issue with table sorting"
git push -u origin feature/my-issue-fix

# 7. Open PR on GitHub targeting fossasia/eventyay:dev
```

---

## 4. 🚀 Quick Setup with Antigravity CLI (`agy`)

When you open this project on another computer using `agy`, paste this single prompt to automate the entire first-time setup:

```text
Please set up my local Eventyay development environment:
1. Clone my fork https://github.com/codebytushar/eventyay.git and set upstream to https://github.com/fossasia/eventyay.git on dev branch.
2. Copy deployment/env.dev.sample to .env.dev.
3. Ensure app/entrypoint.sh has Unix LF line endings to prevent Windows Docker container crash.
4. Launch Docker Desktop if not running and execute 'docker compose up -d --build'.
5. Once migrations and compilemessages finish, create the local admin account admin@eventyay.local / EventyayAdmin123!.
6. Verify http://localhost:8000 responds with HTTP 200.
```

---

## 5. Common Commands Cheat Sheet

| Task | Command |
| :--- | :--- |
| **Start all containers** | `docker compose up -d` |
| **Stop all containers** | `docker compose stop` |
| **Check container health** | `docker ps` |
| **View web service logs** | `docker logs -f eventyay-next-web` |
| **View worker logs** | `docker logs -f eventyay-next-worker` |
| **Restart web container** | `docker restart eventyay-next-web` |
| **Run Django migrations** | `docker exec -it eventyay-next-web python manage.py migrate` |
| **Open Django shell** | `docker exec -it eventyay-next-web python manage.py shell` |
| **Reset DB & Redis completely** | `docker compose down -v` followed by `docker compose up -d --build` |

---

## 6. Troubleshooting & Windows Gotchas

### Issue 1: `exec /usr/src/app/entrypoint.sh: no such file or directory`
- **Cause**: Windows git CRLF line endings in `app/entrypoint.sh`.
- **Fix**: Re-run the PowerShell snippet in [Step 2.3](#step-23-windows-critical-fix-line-endings-crlf-to-lf), then run `docker compose up -d`.

### Issue 2: `failed to connect to the docker API at npipe:////./pipe/docker_engine`
- **Cause**: Docker Desktop is not running or the daemon is starting up.
- **Fix**: Open Docker Desktop from the Start Menu, wait until the Docker dashboard shows "Engine running" (green), and retry your command.

### Issue 3: Page Loads Unstyled (Plain Text HTML) or CSS MIME Type Error
- **Symptom**:
  - The login or admin page loads, but appears as broken, unstyled plain text without CSS.
  - Browser developer console (`F12` > Console) shows an error like:
    ```text
    Refused to apply style from 'http://localhost:8000/static/CACHE/css/output.xxx.css'
    because its MIME type ('text/html') is not a supported stylesheet MIME type.
    ```
- **Why it happens**:
  Django Compressor attempts to serve pre-compiled CSS from `/static/CACHE/css/output.xxx.css`. If that hash has not yet finished compiling, Django returns a 404 HTML page for the `.css` request, and Redis caches that 404 response. The browser receives HTML instead of CSS and refuses to apply it.
- **Step-by-Step Fix**:
  Run this quick 4-step sequence in PowerShell:
  ```powershell
  # 1. Flush the Redis cache
  docker exec -i eventyay-next-redis redis-cli FLUSHDB

  # 2. Collect static files
  docker exec -i eventyay-next-web python manage.py collectstatic --noinput

  # 3. Force re-compression of all CSS bundles
  docker exec -i eventyay-next-web python manage.py compress --force

  # 4. Restart the web service
  docker restart eventyay-next-web
  ```
  After restarting, go to your browser and perform a **Hard Refresh** (`Ctrl + Shift + R` on Windows/Linux or `Cmd + Shift + R` on Mac) to clear the browser's cached 404 page. The CSS will render immediately.
