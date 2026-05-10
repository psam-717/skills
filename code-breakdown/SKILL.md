---
name: code-breakdown
description: >
  Performs a thorough, beginner-friendly breakdown of the entire current project. Reads
  every file in the codebase and produces an extensive explanation of what the project is,
  how it is structured, how data flows through it, and what every major piece of code does.
  Use this skill when asked to explain a project, walk through a codebase, give an overview
  of how something works, or onboard onto an unfamiliar repo.
---

# Code Breakdown Skill

You are acting as a senior engineer giving a clear, high-level explanation of a project to
someone who has never seen it before. Your job is to summarise what the project is and
call out its major features — not to catalogue every file or line of code.

Follow the phases below in strict order.

---

## Phase 1 — Discover & Read

1. Identify the project's language(s), runtime(s), and framework(s) by reading:
   - Package manifests (`package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`,
     `pom.xml`, `Gemfile`, `composer.json`, `.csproj`, etc.)
   - Config files (`tsconfig.json`, `webpack.config.*`, `vite.config.*`, `babel.config.*`,
     `docker-compose.yml`, `Dockerfile`, `.env.example`, etc.)
   - CI/CD pipeline files (`.github/workflows/`, `Jenkinsfile`, `.gitlab-ci.yml`).
2. Identify the project type (e.g. REST API, CLI tool, web app, mobile app, library,
   monorepo, data pipeline, etc.).
3. Read every source file thoroughly enough to understand what the project does and what
   its major capabilities are. Do **not** skim.
4. Skip generated, compiled, and vendored directories (`dist/`, `build/`, `__pycache__/`,
   `*.min.js`, `node_modules/`, `vendor/`).

---

## Phase 2 — Project Summary

Present a **Project Summary** — a concise, plain-English description of the project:

```
## Project Summary

**Name:** <project name from manifest or folder>
**Type:** <REST API / Web App / CLI / Library / etc.>
**Language(s):** <e.g. TypeScript, Python 3.11>
**Framework(s):** <e.g. Express, FastAPI, React, Django>

**What it is:**
3–5 sentences explaining what this project does, what problem it solves, and who would
use it. Base this entirely on the actual code — do not invent or assume features.

**Tech Stack:**
A brief list of the key technologies and what role each plays (e.g. "PostgreSQL — primary
database", "Redis — session caching", "React — frontend UI").
```

---

## Phase 3 — Major Features

Identify and describe the project's **major features** — the significant, user-facing or
system-level capabilities that define what the project actually does.

For each feature:
- Give it a short, clear name.
- Write 2–4 sentences explaining what it does and how it works at a high level.
- Reference the key file(s) or module(s) that implement it (no need for line numbers —
  just the file path is enough).

Format:

```
## Major Features

### 1. User Authentication
Handles registration, login, and session management using JWT tokens. Users submit
credentials which are validated and checked against bcrypt-hashed passwords stored in the
database. On success a signed token is issued and must be included in subsequent requests.
Implemented in `src/services/auth.ts` and `src/controllers/auth.ts`.

### 2. Product Catalogue
Allows browsing, searching, and filtering products by category, price, and availability.
Results are paginated and cached in Redis to reduce database load on frequently accessed
queries. Implemented in `src/services/products.ts` and `src/routes/products.ts`.

### 3. Order Management
...
```

Derive features strictly from what exists in the code. Do not list a feature if there is
no meaningful implementation behind it. Aim for 3–8 features depending on the size of
the project.

---

## Phase 4 — How to Run It

Close with a brief **Quick-Start** so anyone reading can get the project running:

```
## Quick-Start

# Install dependencies
npm install

# Set up environment variables
cp .env.example .env
# Fill in required values (e.g. DATABASE_URL, JWT_SECRET)

# Run the project
npm run dev

# Run tests (if a test suite exists)
npm test
```

Tailor every command to what actually exists in the project (manifests, scripts, migration
tools, etc.). Only include steps that are evidenced by the codebase. If no test suite
exists, omit that step.

---

## General Rules

- Base every statement on actual code. Do not invent features, guess at intent, or pad
  with generic boilerplate.
- Write in plain English. Prefer clarity over technical jargon; explain terms if needed.
- Do not produce a file-by-file or module-by-module inventory — focus on the project as
  a whole and its major capabilities.
- If the project is a monorepo, summarise each package/app in one paragraph and then
  describe the major features across the whole repo.
- Do not suggest improvements, refactors, or bug fixes — this skill is for understanding
  only, not critique.
