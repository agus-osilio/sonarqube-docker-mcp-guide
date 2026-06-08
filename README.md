# 🐳 SonarQube with Docker + MCP for Claude Code — Complete Guide (Windows & Ubuntu)

> Covers two ways to use SonarQube: the full Docker image (web UI) and the MCP Server for Claude Code
>
> Includes instructions for **Windows** (PowerShell) and **Ubuntu** (bash). Look for the Ubuntu callouts at each step where the commands or requirements differ.

> [!WARNING]
> Always review commands before running them. Never execute a command you don't understand.

---

## 🔍 What is SonarQube and why use it?

**SonarQube** is an automated code quality and security analysis platform developed by SonarSource. It scans your codebase and detects **bugs, security vulnerabilities, and code smells** before they reach production — across more than 40 programming languages and frameworks.

It is used by over 7 million developers worldwide and can be deployed as a self-managed server (what this guide covers) or as SaaS via SonarQube Cloud.

**Languages supported in the free Community Build edition** (what this guide covers):
Java, C#, VB.NET, JavaScript, TypeScript, Python, Go, PHP, Kotlin, Ruby, Rust, Scala, CSS, HTML, XML, Flex — plus IaC languages: CloudFormation, Terraform, Kubernetes/Helm, Docker, and Azure Resource Manager — and cross-language secrets detection.

> C# / VB.NET and Java (via Maven or Gradle) require a dedicated scanner, not the generic SonarScanner CLI. See [Step A2](#step-a2--run-the-scanner) for details.

> [!IMPORTANT]
> **The MCP tool `analyze_code_snippet` supports a different, narrower set of languages** than the full Community Edition scanner. It works with: Java, Kotlin, Python, Ruby, Go, JavaScript, TypeScript, JSP, PHP, XML, HTML, CSS, CloudFormation, Kubernetes, Terraform, ARM, Ansible, Docker, and secrets detection. Languages like **C#, VB.NET, Rust, Scala, and Flex are not supported** by this tool — to analyze those, you need to run the full scanner from Path A.

### What problems does it solve?

| Problem | How SonarQube addresses it |
|---------|---------------------------|
| Bugs and defects in production | Catches them at development time, when fixes are cheap |
| Security vulnerabilities | Detects issues mapped to OWASP, CWE, and NIST SSDF — with remediation guidance |
| Inconsistent code standards across a team | Enforces a shared set of quality rules from local dev through CI/CD |
| AI-generated code risks | Validates code written or suggested by AI tools before it ships |
| Compliance burden | Automates proof of adherence to regulatory frameworks |

### Why use it as a standalone tool (Path A)?

Running SonarScanner against your project gives you:

- **Full project audit** — every file analyzed, not just what's open in your editor
- **Historical tracking** — each scan is recorded; you can see how quality evolves over time
- **Quality Gates** — configurable pass/fail thresholds you can enforce in CI/CD pipelines
- **Rich web dashboard** — filter issues by severity, type, file, and rule; drill into each finding with an explanation

This is the standard workflow for teams: scan on every pull request, block merges that break the quality gate, and track technical debt over time.

### Why use it with the MCP Server in Claude Code (Path B)?

The **SonarQube MCP Server** exposes SonarQube's analysis engine as tools that Claude Code can call directly during a conversation. This unlocks a different, complementary workflow:

- **No context switching** — you ask Claude to analyze a file or snippet and get findings immediately in the terminal, without opening a browser
- **Immediate feedback while writing code** — catch issues in the file you're working on right now, before committing
- **AI-driven remediation** — Claude sees the issue and can fix it on the spot, in the same turn
- **Access to project data** — if you've run a prior scan (Path A), Claude can also query stored issues, quality gate status, metrics, and security hotspots directly

The MCP does not replace Path A — it needs SonarQube running to function. Think of Path A as the source of truth for the whole project, and Path B as a fast, in-editor feedback loop while you're actively developing.

---

## 📋 Table of Contents

- [Before you start: understand what each piece does](#before-you-start-understand-what-each-piece-does)
  - [The pieces of the system](#the-pieces-of-the-system)
  - [The two paths](#the-two-paths)
  - [SonarQube without MCP vs SonarQube MCP for Claude Code](#sonarqube-without-mcp-vs-sonarqube-mcp-for-claude-code)
- [Prerequisites](#prerequisites)
- [Common setup — Steps 1 and 2](#common-setup--steps-1-and-2-both-paths)
  - [Step 1 — Start SonarQube Community Build](#step-1--start-sonarqube-community-build)
  - [Step 2 — Configure SonarQube](#step-2--configure-sonarqube)
- [Path A — SonarQube without MCP](#path-a--sonarqube-without-mcp)
  - [Step A1 — Follow the SonarQube onboarding](#step-a1--follow-the-sonarqube-onboarding)
  - [Step A2 — Run the scanner](#step-a2--run-the-scanner)
  - [Step A3 — View results in the UI](#step-a3--view-results-in-the-ui)
  - [Step A4 — Run the scanner again](#step-a4--run-the-scanner-again-next-times)
- [Path B — SonarQube MCP for Claude Code](#path-b--sonarqube-mcp-for-claude-code)
  - [Step B1 — Create a profile in Docker MCP Toolkit](#step-b1--create-a-profile-in-docker-mcp-toolkit)
  - [Step B2 — Add Claude Code as Client and SonarQube as Server](#step-b2--add-claude-code-as-client-and-sonarqube-as-server)
  - [Step B3 — Register SonarQube in Claude Code with `claude mcp add`](#step-b3--register-sonarqube-in-claude-code-with-claude-mcp-add)
  - [Step B4 — Verify the connection](#step-b4--verify-the-connection)
  - [Step B5 — Using it](#step-b5--using-it)
  - [Why does the MCP need a Project Key?](#why-does-the-mcp-need-a-project-key-if-im-not-uploading-any-scan)
  - [Tools available with local Community Edition (17)](#tools-available-with-local-community-edition-17)
- [Troubleshooting](#troubleshooting)
- [Restarting in future sessions](#restarting-in-future-sessions)
- [Flow summary](#flow-summary)
- [Limitations by SonarQube edition](#limitations-by-sonarqube-edition)

---

## 🧩 Before you start: understand what each piece does

### The pieces of the system

| Piece | What it is | What it does |
|-------|-----------|--------------|
| **SonarQube Community Build** | The analysis server | Stores projects, quality rules, and issue history. Has a web UI at `localhost:9000`. **Always required**, regardless of which path you choose. |
| **SonarScanner CLI** | The code scanner | Analyzes your entire project and uploads results to SonarQube. You run it manually or in CI/CD. |
| **SonarQube MCP Server** (`mcp/sonarqube`) | The bridge for Claude Code | Exposes SonarQube's capabilities as tools that Claude Code can use directly. |

### The two paths

```
┌─────────────────────────────────────────────────────────────────┐
│                     PATH A (Just SonarQube)                      │
│                                                                  │
│  Your project  →  SonarScanner  →  SonarQube  →  Browser UI      │
│  (code)           (Docker CLI)     (Docker)       localhost:9000 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                     PATH B (SonarQube MCP + Claude Code)         │
│                                                                  │
│  Claude Code  →  MCP Server  →  SonarQube                        │
│  (terminal)      (Docker)       (Docker)                         │
└─────────────────────────────────────────────────────────────────┘
```

### SonarQube without MCP vs SonarQube MCP for Claude Code

| | **SonarQube without MCP** | **SonarQube MCP for Claude Code** |
|---|---|---|
| **How you analyze** | Run SonarScanner on your project from the terminal | Ask Claude Code to analyze a snippet or file |
| **Where you see results** | In the browser at `localhost:9000` | Directly in the conversation with Claude, without leaving the terminal |
| **Are results saved** | Yes — in SonarQube, with history and metrics over time | Only if you previously ran SonarScanner. Snippet analysis is not saved. |
| **Requires Claude Code** | No | Yes |
| **Workflow** | Write → scan → browser → review → back to editor | Write → ask Claude for analysis → Claude fixes it on the spot |
| **Best for** | Full project audits, CI/CD, technical debt history | Immediate feedback while developing, without context switching |

> [!IMPORTANT]
> The MCP does not replace SonarQube — it needs it. The SonarQube container must be running in both paths.

---

## 📦 Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) version **4.62 or later** (the MCP Toolkit Profiles UI described in this guide requires 4.62+)
- PowerShell (comes with Windows)
- [Claude Code](https://claude.ai/download) — only if you're using Path B

> [!NOTE]
> **Ubuntu:** Install [Docker Desktop for Ubuntu](https://docs.docker.com/desktop/setup/install/linux/ubuntu/) (version 4.62 or later). Docker Desktop is required — Docker Engine alone does not include the MCP Toolkit. Use your terminal (bash or zsh) instead of PowerShell; the Claude Code CLI is the same.

---

## ⚙️ Common setup — Steps 1 and 2 (both paths)

These steps are the same regardless of which path you choose.

---

### Step 1 — Start SonarQube Community Build

```bash
docker run -d --name sonarqube -p 9000:9000 sonarqube:community
```

SonarQube takes **1-2 minutes** to initialize. Open your browser at `http://localhost:9000`. When you see the login screen, it's ready. If you see a 502 error, wait another minute.

> **Persistence:** Without a volume, all data is lost if you delete the container. For real projects:
> ```bash
> docker run -d --name sonarqube -p 9000:9000 -v sonarqube_data:/opt/sonarqube/data sonarqube:community
> ```
>
> **Understanding the `-v` flag:** The format is `HOST_SIDE:CONTAINER_SIDE`.
> - `sonarqube_data` (left) is the **name of the Docker volume** — you can change this to whatever you want (e.g. `my-sonar-vol`). Docker manages where it's actually stored on your machine.
> - `/opt/sonarqube/data` (right) is the **path inside the container** where SonarQube writes its data. This is fixed by the official image — do not change it.
>
> If you prefer to store data in a specific folder instead of a Docker-managed volume, use a bind mount:
>
> **Windows:**
> ```powershell
> docker run -d --name sonarqube -p 9000:9000 -v "C:\Users\YourName\sonarqube-data:/opt/sonarqube/data" sonarqube:community
> ```
>
> **Ubuntu:**
> ```bash
> docker run -d --name sonarqube -p 9000:9000 -v ~/sonarqube-data:/opt/sonarqube/data sonarqube:community
> ```

---

### Step 2 — Configure SonarQube

#### Initial login

| Field | Value |
|-------|-------|
| Username | `admin` |
| Password | `admin` |

On first login, you'll be prompted to change your password. Choose one and save it.

#### Create a project

1. On the main screen, click **Create Project** → **Local project**
2. Fill in:
   - **Project display name:** whatever you want (e.g. `my-project`)
   - **Project key:** auto-filled — **write this value down**
3. Click **Next** → **Create project**

> [!NOTE]
> The Project Key is the unique identifier for the project inside SonarQube. You'll need it for both Path A (in the scanner command) and Path B (when asking Claude Code for analysis).

#### Generate an authentication token

Both the scanner and the MCP need a token to authenticate. You need a **User Token** — generated here, not from the project onboarding wizard.

1. Click your **avatar** (top right) → **My Account**
2. Go to the **Security** tab
3. Under **Generate Tokens**:
   - **Name:** whatever you want (e.g. `local-dev`)
   - **Type:** `User Token`
   - **Expires in:** your preference
4. Click **Generate**
5. **Copy the token** — you won't be able to see it again

A User Token starts with `squ_`: `squ_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`

> **Why not the token the project onboarding generates?**
>
> When you click through "Analyze locally" after creating a project, the wizard offers to generate a token for you — that token is a **Project Analysis Token** (you can tell because it starts with `sqp_`). It is fine for running the scanner (Path A), but **does not work for the MCP**: analysis tokens only have permission to push scan results to SonarQube; they cannot call the Web API to read issues, metrics, or quality gate status. The MCP needs that read access, so it requires a User Token.

**Token types compared**

| Type | Generated from | What it can do | Scanner (Path A) | MCP (Path B) |
|------|----------------|----------------|:----------------:|:------------:|
| **User Token** | My Account → Security | Full Web API — same permissions as your account | ✅ | ✅ |
| **Global Analysis Token** | My Account → Security | Push scan results to any project | ✅ | ❌ |
| **Project Analysis Token** | My Account → Security, or the project onboarding wizard | Push scan results to one specific project | ✅ | ❌ |

> **Why can't analysis tokens read the API?** They follow the least-privilege principle: their only purpose is to let a build system upload results. The MCP queries the API to retrieve stored issues, metrics, and security hotspots — operations that require the broader access a User Token provides.

---

## 🖥️ Path A — SonarQube without MCP

You use SonarScanner to analyze your project and view the results in SonarQube's web UI.

---

### Step A1 — Follow the SonarQube onboarding

When you create the project in SonarQube, the UI walks you through an onboarding that generates the exact command to run the scanner. You don't need to create any configuration file.

1. After creating the project, SonarQube asks **how you want to analyze your repository** — choose the option that applies (Locally, with a CI, etc.)
2. Generate an analysis token when prompted (or use the one you already created in Step 2)
3. SonarQube shows you the **ready-to-use command to copy and paste**

---

### Step A2 — Run the scanner

The SonarQube onboarding asks **"What option best describes your project?"** and shows tabs: Maven, Gradle, JS/TS & Web, .NET, Python, and Other (for Go, PHP, Ruby...). **Select the one that matches your project and copy the command it generates** — each tab produces a completely different command.

**Why does each project type have its own scanner?**

Most languages are supported by more than one scanner — the distinction isn't which languages each one covers, it's **how deeply each one can analyze them**. For Maven, Gradle, and .NET projects, the dedicated scanner hooks directly into the build process and gains access to the compiled output, classpath, and dependency graph. SonarQube uses that information to perform accurate type resolution and data flow analysis. The generic CLI (`sonar-scanner`) runs independently of any build system and cannot access any of that — the official documentation explicitly warns: *"Don't use the SonarScanner CLI for projects built with Maven, Gradle, or .NET. Doing so will degrade the quality of your analysis."* For Python and JS/TS, the generic CLI works fine — the onboarding offers `pysonar` (pip) and `@sonar/scan` (npm) as convenient alternatives: `pysonar` is officially documented as a wrapper around the CLI that handles Java provisioning automatically, and `@sonar/scan` is an npm-native package, but neither is required if you prefer to use the CLI directly. The generic CLI is also the right choice for Go, PHP, Ruby, and similar languages.

It's also worth noting that `@sonar/scan` and `pysonar` are not restricted to the language their tab suggests — since they invoke the same underlying analysis engine, they work as general-purpose scanners for any language supported by Community Build. If your project mixes Python, HTML, CSS, and JavaScript for example, running `@sonar/scan` or `sonar-scanner` from the project root will analyze all of them in a single scan. **The analysis quality is identical across all three options** (`sonar-scanner`, `pysonar`, `@sonar/scan`) — the official documentation describes `pysonar` explicitly as *"a wrapper around SonarScanner CLI"* whose differences are purely operational (pip install, `pyproject.toml` support, automatic JRE provisioning). The underlying analysis engine and the findings it produces are the same. *Source: [SonarScanner for Python — official docs](https://docs.sonarsource.com/sonarqube-community-build/analyzing-source-code/scanners/sonarscanner-for-python/)*

> [!IMPORTANT]
> **Do not use the generic `sonar-scanner` CLI for Maven, Gradle, or .NET projects.**
> - **Maven / Gradle:** The CLI has no access to the compiled classpath or dependencies, so SonarQube receives incomplete information. The scan will still report `EXECUTION SUCCESS`, giving no indication that the analysis quality was degraded.
> - **C# / VB.NET (.NET):** The CLI cannot invoke the .NET compiler. Those files are skipped entirely and SonarQube receives no findings for them, even though the scan reports success.

The onboarding wizard generates the **exact command ready to paste** for whichever tab you select. For the **Other** tab (Go, PHP, Ruby, and similar) — and as a general-purpose scanner for any supported language — there are two installation options:

**Option 1 — via npm (same command on Windows and Ubuntu):**

```bash
npm install -g @sonar/scan
```

Then run from the root of your project:

**Windows** (PowerShell — single line):
```powershell
sonar -Dsonar.host.url=http://localhost:9000 -Dsonar.token=YOUR_TOKEN -Dsonar.projectKey=PROJECT-KEY
```

**Ubuntu** (bash — backslash `\` for line continuation):
```bash
sonar \
  -Dsonar.host.url=http://localhost:9000 \
  -Dsonar.token=YOUR_TOKEN \
  -Dsonar.projectKey=YOUR_PROJECT_KEY
```

> [!NOTE]
> **Why `localhost` here and not `host.docker.internal`?** Because all these scanners run natively on your machine, not inside a Docker container. From your machine, `localhost:9000` reaches SonarQube without any issue. `host.docker.internal` is only needed when the connecting process runs *inside* Docker.

**Option 2 — via direct download (SonarScanner CLI binary):**

Download the binary for your OS from the [SonarScanner CLI official docs](https://docs.sonarsource.com/sonarqube-community-build/analyzing-source-code/scanners/sonarscanner/), extract it, and add the `bin/` directory to your PATH. Then the command is `sonar-scanner` instead of `sonar`, with the same parameters.

Open a terminal at the **root of your project** and run the command for your project type.

The scanner displays logs in the terminal. At the end you should see:

```
INFO: EXECUTION SUCCESS
INFO: Total time: Xs
```

If you see `EXECUTION FAILURE`, check the Troubleshooting section.

---

### Step A3 — View results in the UI

1. Open your browser at `http://localhost:9000`
2. Click on your project on the main screen

The project's left sidebar has these sections:

| Tab | What it shows |
|-----|--------------|
| **Overview** | Opens by default. Shows Quality Gate status, a metrics summary, and analysis history. |
| **Issues** | List of all detected problems. You can filter by type, severity, file, etc. |
| **Measures** | Detailed project metrics (coverage, complexity, technical debt, duplications, etc.). |
| **Code** | Project structure file by file, with issues marked inline. |
| **Activity** | History of all scans run and how metrics evolved over time. |

To see the detected problems, go to **Issues** in the left sidebar.

Click on any number to see individual issues — each one shows the file, line number, severity, and problem description.

---

### Step A4 — Run the scanner again (next times)

Every time you want to update the analysis, simply run the same command from Step A2 again. SonarQube saves the history and you can track quality evolution over time.

---

## 🤖 Path B — SonarQube MCP for Claude Code

You connect Claude Code to SonarQube's MCP Server to request analysis directly from the terminal.

> [!IMPORTANT]
> Claude Code with the MCP analyzes your code on the fly and returns the result in the conversation. **Nothing is uploaded to SonarQube** — results do not appear in the `localhost:9000` dashboard and are not saved in the project history. If you need results persisted in SonarQube, you need to run the traditional scanner from Path A.

---

### Step B1 — Create a profile in Docker MCP Toolkit

Docker Desktop has a feature called **MCP Toolkit** that lets you manage which MCP servers are available to each AI client (like Claude Code), through a graphical interface without touching the terminal.

> [!IMPORTANT]
> The SonarQube container must be running, with login done and the token valid (Steps 1 and 2 of the common setup) before continuing. If SonarQube is not running, the MCP Server won't be able to connect.

1. Open **Docker Desktop**
2. In the left sidebar, click **MCP Toolkit**
3. Go to the **Profiles** tab
4. Click **Create profile**
5. Name it `default`

> [!NOTE]
> The profile is named `default` because **you can only have one profile per client** — meaning Claude Code can only be associated with one profile at a time. There's no reason to create a profile with a different name for this use case.

---

### Step B2 — Add Claude Code as Client and SonarQube as Server

#### Add Claude Code as a client

Inside the `default` profile:

1. Find the **Clients** section
2. Click **+**
3. Select **Claude Code**
4. Click **Connect**

#### Add SonarQube as a server

1. Go to the **Catalog** tab in MCP Toolkit
2. Search for **SonarQube** and click **+ Add**
3. Select the `default` profile when prompted
4. In the server configuration section, fill in:
   - `SONARQUBE_URL`: `http://host.docker.internal:9000`
   - `SONARQUBE_TOKEN`: the **User Token** (`squ_...`) you generated in Step 2 — project and global analysis tokens will not work here
5. Save the configuration

Once configured, the MCP Toolkit automatically starts the MCP Server container (`mcp/sonarqube`) when Claude Code needs it. **This is separate from the SonarQube container** (`localhost:9000`) — you still need to start that one yourself with `docker start sonarqube`.

> [!NOTE]
> **Why `host.docker.internal` and not `localhost`?** The MCP Server runs inside a Docker container. From inside Docker, `localhost` points to the container itself, not your machine. `host.docker.internal` is the hostname Docker provides to reach the host machine, where SonarQube is listening on port 9000.

#### Optional configuration variables

| Variable | What it does | Example |
|----------|-------------|---------|
| `SONARQUBE_PROJECT_KEY` | Default Project Key. If set, you don't need to repeat it in every request to Claude. | `my-project` |
| `SONARQUBE_TOOLSETS` | Enables only certain groups of tools. If not set, all are enabled. | `analysis,issues,quality-gates` |
| `SONARQUBE_READ_ONLY` | When `true`, disables tools that modify data in SonarQube. | `true` |
| `SONARQUBE_DEBUG_ENABLED` | Enables verbose logging. Useful for troubleshooting. | `true` |

---

### Step B3 — Register SonarQube in Claude Code with `claude mcp add`

Configuring the profile in Docker Desktop is not enough for Claude Code to recognize the MCP server — **you also need to register it** with the following command. Without this step, the server may not appear in `claude mcp list` or `/mcp`.

**Windows** — open PowerShell and run this in **a single line** (replace the values):

```powershell
claude mcp add --scope local sonarqube --env SONARQUBE_TOKEN="YOUR_TOKEN_HERE" --env SONARQUBE_URL="http://host.docker.internal:9000" -- docker run --init --pull=always -i --rm -e SONARQUBE_TOKEN -e SONARQUBE_URL -v "C:\path\to\your\project:/app/mcp-workspace:rw" mcp/sonarqube
```

Replace:
- `YOUR_TOKEN_HERE` → the **User Token** (`squ_...`) from Step 2. The official MCP docs state explicitly that project and global analysis tokens will not function properly here.
- `C:\path\to\your\project` → absolute path to your working folder (e.g. `C:\Users\YourName\Desktop\my-project`)

**Ubuntu** — open your terminal and run this in **a single line** (replace the values):

```bash
claude mcp add --scope local sonarqube --env SONARQUBE_TOKEN="YOUR_TOKEN_HERE" --env SONARQUBE_URL="http://host.docker.internal:9000" -- docker run --init --pull=always -i --rm -e SONARQUBE_TOKEN -e SONARQUBE_URL -v "/home/your-user/your-project:/app/mcp-workspace:rw" mcp/sonarqube
```

Replace:
- `YOUR_TOKEN_HERE` → the **User Token** (`squ_...`) from Step 2.
- `/home/your-user/your-project` → absolute path to your project folder (e.g. `/home/agus/Desktop/my-project`). You can also use `$HOME/my-project` or the output of `pwd` from inside the folder.

> [!NOTE]
> **Ubuntu — `host.docker.internal`:** This works with Docker Desktop for Linux because Docker Desktop runs a VM internally (the same mechanism as on Windows and Mac). If you were using Docker Engine without Docker Desktop, `host.docker.internal` would not resolve — but since this guide requires Docker Desktop for the MCP Toolkit, no extra flags are needed.

> Each `--env NAME="value"` before the server name saves the variable in Claude Code's config. Each `-e NAME` after `--` (inside the Docker command) passes it to the MCP container when it starts.

#### Command breakdown

**General structure**

```
claude mcp add [options] [name] -- [MCP server command]
```

**Part 1: `claude mcp add`**

Registers a new MCP server in Claude Code.

| Flag | Value | Meaning |
|------|-------|---------|
| `--scope local` | local | Only available in this project (not global) |
| `--env SONARQUBE_TOKEN="..."` | your User Token (`squ_...`) | Environment variable passed to the server — must be a User Token |
| `--env SONARQUBE_URL="..."` | SonarQube URL | URL of your SonarQube instance |
| `sonarqube` | — | Name Claude uses to identify this MCP |

**Part 2: `docker run` (the MCP server itself)**

Windows (PowerShell — backtick `` ` `` for line continuation):
```powershell
docker run --init --pull=always -i --rm `
  -e SONARQUBE_TOKEN `
  -e SONARQUBE_URL `
  -v "C:\path\to\your\project:/app/mcp-workspace:rw" `
  mcp/sonarqube
```

Ubuntu (bash — backslash `\` for line continuation):
```bash
docker run --init --pull=always -i --rm \
  -e SONARQUBE_TOKEN \
  -e SONARQUBE_URL \
  -v "/home/your-user/your-project:/app/mcp-workspace:rw" \
  mcp/sonarqube
```

| Flag | Meaning |
|------|---------|
| `--init` | Uses an init process to handle signals correctly |
| `--pull=always` | Always pulls the latest image |
| `-i` | Interactive mode (required for MCP communication over stdin/stdout) |
| `--rm` | Removes the container when it exits |
| `-e SONARQUBE_TOKEN` | Injects the environment variable into the container |
| `-e SONARQUBE_URL` | Injects the SonarQube URL into the container |
| `mcp/sonarqube` | The Docker image for the MCP server |

**The `-v` volume explained**

Windows:
```
C:\path\to\your\project  →  /app/mcp-workspace  (rw)
     │                            │                │
Your project on Windows    Path inside          Read and
                           the container        write
```

Ubuntu:
```
/home/your-user/your-project  →  /app/mcp-workspace  (rw)
         │                              │                │
   Your project on Linux         Path inside          Read and
                                 the container        write
```

This gives the SonarQube MCP server access to your source code so it can analyze it.

> [!IMPORTANT]
> **Using the correct path reduces token usage.**
> When the workspace is mounted correctly, the MCP server reads files directly from disk using the `filePath` parameter, keeping file content out of the agent context window. If the path is left as the placeholder (Windows: `C:\path\to\your\project` / Ubuntu: `/home/your-user/your-project`) or points to a non-existent folder, the mount fails silently and the MCP has no file access — forcing Claude to read each file in full and pass its content via the `fileContent` parameter, consuming significantly more tokens.
>
> *Source: [SonarQube MCP Server Tools — official docs](https://docs.sonarsource.com/sonarqube-mcp-server/using/tools)*

**Full flow**

```
Claude Code
    │
    ▼
claude mcp add  →  registers the server
    │
    ▼
Docker container (mcp/sonarqube)
    │  reads your code from
    ▼
your-project/  ←→  /app/mcp-workspace
(Windows: C:\path\to\your\project)
(Ubuntu:  /home/your-user/your-project)
    │
    ▼
SonarQube at http://host.docker.internal:9000
```

> `host.docker.internal` is the special hostname Docker Desktop uses on Windows, Mac, and Linux to refer to the host's `localhost` from inside a container.

---

**Where is the configuration saved?**

By default, `claude mcp add` uses `local` scope:

- The config is saved in `~/.claude.json` (your home folder — `C:\Users\YourName\.claude.json` on Windows, `/home/your-user/.claude.json` on Ubuntu)
- But it is **indexed under the project path** from which you ran the command
- Result: the MCP only appears when you open Claude Code from that folder — in any other project, it doesn't exist

In other words, **it's not global** (it doesn't appear in all your projects), but it's also not a file inside the project (nothing is created in your working folder). The `~/.claude.json` file acts as a central registry where each project has its own section.

If you want the MCP available in **all your projects**, add `--scope user` before the server name:

Windows:
```powershell
claude mcp add --scope user sonarqube --env SONARQUBE_TOKEN="YOUR_TOKEN_HERE" --env SONARQUBE_URL="http://host.docker.internal:9000" -- docker run --init --pull=always -i --rm -e SONARQUBE_TOKEN -e SONARQUBE_URL -v "C:\path\to\your\project:/app/mcp-workspace:rw" mcp/sonarqube
```

Ubuntu:
```bash
claude mcp add --scope user sonarqube --env SONARQUBE_TOKEN="YOUR_TOKEN_HERE" --env SONARQUBE_URL="http://host.docker.internal:9000" -- docker run --init --pull=always -i --rm -e SONARQUBE_TOKEN -e SONARQUBE_URL -v "/home/your-user/your-project:/app/mcp-workspace:rw" mcp/sonarqube
```

---

### Step B4 — Verify the connection

> `claude mcp list` — lists all MCP servers registered in Claude Code and their connection status.

```bash
claude mcp list
```

You should see:
```
sonarqube: docker run ... - ✓ Connected
```

Or from inside Claude Code, type `/mcp`:
```
sonarqube · ✔ connected · N tools
```

> [!NOTE]
> The number of tools varies depending on the SonarQube edition and token permissions. With local Community Edition, seeing 17 is normal. See the tools table below for details.

---

### Step B5 — Using it

#### Analyze a code snippet

Use the `analyze_code_snippet` tool. You can pass the code directly in the message:

```
Please use the analyze_code_snippet SonarQube MCP tool to review sample.py for any issues, using the project key <sample-prokect-key>
```

```
Use the analyze_code_snippet tool to analyze this Python code using project key <sample-prokect-key>:

def login(user, password):
    query = "SELECT * FROM users WHERE user='" + user + "'"
    db.execute(query)
```

```
Use analyze_code_snippet to check this JavaScript function for security issues, project key <sample-prokect-key>:

function getUser(id) {
    return db.query("SELECT * FROM users WHERE id = " + id);
}
```

#### Analyze a file from the mounted workspace

If you mounted your folder with `-v`, Claude can read the file directly from disk without you pasting the content:

```
Use analyze_code_snippet to audit the file "sample.py" and search for issues, using the project key <sample-prokect-key>
```

```
Use analyze_code_snippet with filePath "src/auth.py" and project key <sample-prokect-key> to find all security issues
```

```
Use analyze_code_snippet on filePath "controllers/UserController.java" with project key <sample-prokect-key>
```

#### View the details of a specific rule

```
Use show_rule to get the details of rule python:S2076
```

#### Search for security issues in the project (requires prior Path A scan)

```
Use search_sonar_issues_in_projects with project key <sample-prokect-key> to list all VULNERABILITY issues
```

```
Use get_project_quality_gate_status for project key <sample-prokect-key>
```

```
Use search_security_hotspots for project key <sample-prokect-key>
```

#### View project metrics (requires prior Path A scan)

```
Use get_component_measures for project key <sample-prokect-key> to get coverage and complexity metrics
```

---

### Why does the MCP need a Project Key if I'm not uploading any scan?

This is a common question. The answer: **the Project Key tells the analysis engine which rules and quality profile to apply**.

SonarQube does not always apply the same rules. Each project has a **Quality Profile** that defines which rules are active (out of the thousands available) and at what severity. Without a Project Key, the engine doesn't know what criteria to use to analyze your code.

**Two types of tools in the MCP depending on whether they need a prior scan:**

**Do not require a prior scan — analyze on the fly:**
- `analyze_code_snippet` — analyzes a snippet or full file using the project's rules.

**Require a prior scan — query historical data:**
- `search_sonar_issues_in_projects` — searches for issues already detected by the scanner.
- `get_project_quality_gate_status` — checks whether the project passed its quality gate.
- `get_component_measures` — retrieves metrics (coverage, technical debt, etc.).
- `search_security_hotspots` — searches for security hotspots in the project.

> [!NOTE]
> If you've never run the scanner (Path A), the tools in the second group will return empty results. The first group works regardless — it performs the analysis on the spot.

---

### Tools available with local Community Edition (17)

The MCP exposes up to 25 tools in total, but with local SonarQube Community Edition only 17 are available. The rest require higher editions, administrator permissions, or external services.

**✅ Available in Community Edition**

| Tool | What it does | Requires prior scan |
|------|-------------|:-------------------:|
| `analyze_code_snippet` | Analyzes a snippet or file with SonarQube's engine | No |
| `search_sonar_issues_in_projects` | Searches for detected issues in the project | Yes |
| `change_sonar_issue_status` | Changes an issue's status (false positive, resolved, etc.) | Yes |
| `search_my_sonarqube_projects` | Lists all your projects | No |
| `list_quality_gates` | Lists configured quality gates | No |
| `get_project_quality_gate_status` | Checks whether a project passed its quality gate | Yes |
| `get_component_measures` | Retrieves metrics (coverage, complexity, technical debt) | Yes |
| `search_metrics` | Searches available metrics | No |
| `show_rule` | Shows the details of a specific rule | No |
| `show_security_hotspot` | Shows the details of a security hotspot | Yes |
| `search_security_hotspots` | Searches for security hotspots in the project | Yes |
| `change_security_hotspot_status` | Changes the status of a hotspot | Yes |
| `get_duplications` | Retrieves code duplication information | Yes |
| `search_duplicated_files` | Searches for files with duplicated code | Yes |
| `get_file_coverage_details` | Retrieves coverage details for a file | Yes |
| `search_files_by_coverage` | Searches for files by coverage level | Yes |
| `list_pull_requests` | Lists pull requests for the project | Yes |

**❌ Not available in local Community Edition**

| Tool | Why it's unavailable |
|------|---------------------|
| `analyze_file_list`, `toggle_automatic_analysis` | Require SonarQube for IDE running locally |
| `run_advanced_code_analysis` | SonarQube Cloud only + paid entitlement |
| `search_dependency_risks` | Enterprise Edition (Server 2025.4+) with Advanced Security |
| `list_enterprises`, `list_portfolios` | Enterprise / SonarQube Cloud only |
| `ping_system`, `get_system_health/status/info/logs` | Require administrator role on the server |
| `create_webhook`, `list_webhooks` | Require administrator permissions |
| `list_languages`, `get_raw_source`, `get_scm_info` | Should work in Community — likely filtered by `SONARQUBE_TOOLSETS` in your config |
| Context Augmentation (10 tools) | Require paid SonarQube Cloud entitlement |

**Languages supported for local analysis with `analyze_code_snippet`:** Java, Kotlin, Python, Ruby, Go, JavaScript, TypeScript, JSP, PHP, XML, HTML, CSS, CloudFormation, Kubernetes, Terraform, ARM, Ansible, Docker, and secrets detection.

---

## 🔧 Troubleshooting

---

### `EXECUTION FAILURE` when running the scanner — Path A

The scanner finished with an error. Check these causes in order:

| Cause | How to fix it |
|-------|--------------|
| SonarQube is still starting up | Wait a minute and retry |
| The Project Key doesn't match (case-sensitive) | Copy it exactly from `localhost:9000` |
| Wrong token type | For the MCP, must be a **User Token** (Global and Project Analysis tokens lack read permissions). For the scanner only, all three types work. |
| Used `localhost` in the scanner URL via Docker | Replace with `host.docker.internal` |

---

### Scanner finishes with `EXECUTION SUCCESS` but nothing appears in SonarQube — Path A

The scan ran correctly but results don't appear in `localhost:9000`. The most common cause is that the Project Key in the command doesn't exactly match the one you created in Step 2 — check capitalization, hyphens, and dots.

---

### `sonarqube already exists in local config` — Path B

An entry with that name already exists in Claude Code's config. The following command tells Claude Code to remove it:

> `claude mcp remove sonarqube` — deletes the MCP server named `sonarqube` from Claude Code's local configuration.

```bash
claude mcp remove sonarqube
```

Then run the command from Step B3 again.

---

### `Failed to connect` / `-32000` — Path B

The MCP Server can't connect to SonarQube. Check in order:

1. **Is SonarQube running?**

   > `docker ps` — lists all currently active Docker containers. If `sonarqube` doesn't appear, the container is stopped.

   ```bash
   docker ps
   ```

2. **Did you use `host.docker.internal` in the URL?** — `localhost` doesn't work from inside the MCP container.

3. **Is the token valid?** — Go to `localhost:9000`, generate a new one, and update the command.

---

### `EOF` on initialization — Path B

The MCP Server started but received an empty token. Check two things:

- The token must be in quotes: `--env SONARQUBE_TOKEN="squ_xxx..."` — without quotes it may break if the token contains special characters.
- The `-e SONARQUBE_TOKEN` flag must be present in the Docker part of the command (after `--`): this flag tells the container to take the variable from the environment Claude Code passes to it. Without it, the token never arrives.

---

### The MCP appears connected but project tools return empty results — Path B

This is expected behavior if you've never run the scanner from Path A. Tools that query issues, metrics, or quality gates read historical data that SonarQube only has after at least one complete scan. `analyze_code_snippet` still works because it performs the analysis on the spot.

---

### Fewer tools than expected appear in `/mcp` — Path B

Two possible causes:

- You configured `SONARQUBE_TOOLSETS` when adding the MCP, which limits active toolsets to only those you listed.
- The token doesn't have sufficient permissions — some tools require an administrator role in SonarQube.

---

### SonarQube doesn't load at `localhost:9000` — Both paths

> `docker ps` — lists all active Docker containers. Use it to check whether SonarQube is running.

```bash
docker ps
```

If `sonarqube` doesn't appear in the list, the container is stopped. The following command restarts it without losing data (unlike `docker run`, which would create a brand new container from scratch):

> `docker start sonarqube` — restarts an existing container that was stopped.

```bash
docker start sonarqube
```

If it appears in the list but `localhost:9000` doesn't load, wait 1-2 minutes — the container is active but SonarQube is still initializing internally.

> [!NOTE]
> **Ubuntu — container starts but crashes immediately:** SonarQube uses embedded Elasticsearch, which requires `vm.max_map_count ≥ 524288` on the Linux kernel. **With Docker Desktop for Linux this is managed automatically** (Docker Desktop runs containers inside its own VM with a kernel it controls, same as on Mac and Windows). If the crash happens anyway, the VM's value may be insufficient — run this on the host to override it:
>
> ```bash
> sudo sysctl -w vm.max_map_count=524288
> ```
>
> Then restart the container with `docker start sonarqube`. To make it permanent across reboots:
>
> ```bash
> echo "vm.max_map_count=524288" | sudo tee /etc/sysctl.d/99-sonarqube.conf
> sudo sysctl -p /etc/sysctl.d/99-sonarqube.conf
> ```
>
> *Note: this is only relevant for Docker Engine on Linux (bare metal, no Docker Desktop). If you are using Docker Desktop as required by this guide, you should not need it.*

---

### 502 error at `localhost:9000` — Both paths

SonarQube is in the middle of starting up. Wait 1-2 minutes and reload the page. This is normal the first time you start the container or after restarting your PC.

---

## 🔄 Restarting in future sessions

If you restarted your machine or closed Docker Desktop:

> `docker start sonarqube` — restarts the stopped SonarQube container.

```bash
docker start sonarqube
```

For Path B, Claude Code already has the MCP configuration saved in `~/.claude.json`. You don't need to run `claude mcp add` again.

> `docker ps` — confirms the container is running. `claude mcp list` — confirms Claude Code can see the MCP server.

```bash
# Verify everything is active
docker ps
claude mcp list
```

---

## 🗺️ Flow summary

```
[First time — both paths]
  1. docker run sonarqube:community        → SonarQube at localhost:9000
  2. Create project at localhost:9000      → Get Project Key
  3. Generate User Token at localhost:9000 → Get squ_xxx...

[Path A — every time you analyze]
  1. docker start sonarqube               → If not already running
  2. Run scanner command                  → Scanner analyzes and uploads results
  3. Open localhost:9000                  → View dashboard with issues and metrics

[Path B — one-time setup]
  claude mcp add --scope local sonarqube --env ...  → Saved in ~/.claude.json scoped to this folder

[Path B — every work session]
  1. docker start sonarqube              → If not already running
  2. Open Claude Code                    → MCP already configured and connected
  3. Ask Claude for analysis             → Immediate feedback in the terminal
```

---

## 📊 Limitations by SonarQube edition

| Feature | Community Build | Server 2025.1+ | Cloud |
|---------|:---------------:|:--------------:|:-----:|
| Snippet analysis (MCP) | ✅ | ✅ | ✅ |
| Issues, metrics, quality gates | ✅ | ✅ | ✅ |
| Dependency risks (SCA) | ❌ | Enterprise 2025.4+ only | ✅ |
| Context Augmentation | ❌ | ❌ | Paid entitlement only |

---

*Based on official SonarSource documentation and real setup experience on Windows (Docker Desktop + PowerShell) and Ubuntu (Docker Desktop + bash).*
