# Phase 0 — Foundations Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up the monorepo, backend, frontend, database, auth, observability, and deployments so that a hosted end-to-end "hello world" streaming demo runs against real infrastructure at `uncover.robinching.com`.

**Architecture:** Turborepo monorepo with `apps/web` (Next.js 15) and `apps/api` (Python FastAPI on Modal). Supabase Postgres with Row-Level Security stores runs and run_steps; Supabase Auth with Google OAuth handles identity. A toy demo agent streams status updates via Server-Sent Events from FastAPI → Next.js so the real agent work in P1 slots into proven plumbing.

**Tech Stack:** pnpm, Turborepo, Next.js 15 (App Router), TypeScript, Tailwind, vitest, Python 3.12, FastAPI, uv, pytest, httpx, Supabase (Postgres + Auth + RLS), Modal (Python hosting), Vercel (Next.js hosting), LangSmith (observability).

**Constraints:**
- No Supabase CLI or `gh` CLI locally — all Supabase and GitHub operations happen via dashboard / web UI
- Hard monthly cost ceiling: $5. P0 stays within free tiers.
- Spec reference: `docs/specs/2026-04-15-design.md`

---

## File Structure

Files created during P0:

```
uncover/
├── package.json                              # Turborepo root, pnpm workspace
├── pnpm-workspace.yaml
├── turbo.json
├── .nvmrc                                    # Node version lock
├── apps/
│   ├── web/                                  # Next.js 15 app
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── next.config.mjs
│   │   ├── tailwind.config.ts
│   │   ├── postcss.config.mjs
│   │   ├── vitest.config.ts
│   │   ├── app/
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx                      # Landing placeholder
│   │   │   ├── globals.css
│   │   │   └── demo/
│   │   │       └── page.tsx                  # SSE demo consumer
│   │   ├── lib/
│   │   │   ├── env.ts                        # Env var validation
│   │   │   └── sse.ts                        # SSE client helper
│   │   └── __tests__/
│   │       └── sse.test.ts                   # vitest unit test
│   └── api/                                  # Python FastAPI app
│       ├── pyproject.toml                    # uv-managed
│       ├── uv.lock
│       ├── modal_app.py                      # Modal deployment entrypoint
│       ├── .python-version
│       ├── app/
│       │   ├── __init__.py
│       │   ├── main.py                       # FastAPI app factory
│       │   ├── config.py                     # Settings via pydantic-settings
│       │   ├── db/
│       │   │   ├── __init__.py
│       │   │   └── supabase_client.py        # Singleton Supabase client
│       │   ├── logging/
│       │   │   ├── __init__.py
│       │   │   └── run_logger.py             # runs / run_steps helper
│       │   ├── agents/
│       │   │   ├── __init__.py
│       │   │   └── demo/
│       │   │       ├── __init__.py
│       │   │       └── counter.py            # Toy async generator agent
│       │   └── routes/
│       │       ├── __init__.py
│       │       ├── health.py                 # /health
│       │       └── stream.py                 # /run/demo + /stream/:run_id (SSE)
│       └── tests/
│           ├── __init__.py
│           ├── conftest.py
│           ├── test_run_logger.py
│           ├── test_counter.py
│           └── test_stream.py
├── supabase/
│   ├── migrations/
│   │   └── 20260415_p0_foundations.sql
│   └── README.md                             # How to apply migrations via dashboard
└── docs/
    ├── specs/
    │   └── 2026-04-15-design.md              # Already exists
    └── plans/
        └── 2026-04-15-phase-0-foundations.md # This file
```

---

## Prerequisites (Before Task 1)

These are manual one-time setup steps you must do yourself. They don't produce repo commits.

- [ ] **Install pnpm globally** (if not already): `npm install -g pnpm`
- [ ] **Install uv** (fast Python package manager): `pip install uv` (inside any Python venv, or via the standalone installer from uv docs)
- [ ] **Have a Google Cloud account** ready (for creating an OAuth app in Task 5)
- [ ] **Have a Supabase account** ready
- [ ] **Have a Vercel account** ready and connected to your GitHub
- [ ] **Have a Modal account** ready (`pip install modal` + `modal token new` to authenticate)
- [ ] **Have a LangSmith account** ready
- [ ] **Create an empty GitHub repo** `uncover` under your GitHub account via the web UI. Note the remote URL.
- [ ] **Add the GitHub remote to the local repo** and push `main`:

```bash
cd C:/Users/robin/Documents/search-agents/uncover
git remote add origin https://github.com/<your-username>/uncover.git
git push -u origin main
```

Once these prerequisites are done, proceed to Task 1.

---

## Task 1: Bootstrap Turborepo Monorepo

**Files:**
- Create: `package.json`
- Create: `pnpm-workspace.yaml`
- Create: `turbo.json`
- Create: `.nvmrc`

- [ ] **Step 1: Create `.nvmrc` with Node 20**

Write `.nvmrc`:
```
20
```

- [ ] **Step 2: Create root `package.json`**

Write `package.json`:
```json
{
  "name": "uncover",
  "version": "0.0.0",
  "private": true,
  "packageManager": "pnpm@9.12.0",
  "scripts": {
    "dev": "turbo run dev",
    "build": "turbo run build",
    "test": "turbo run test",
    "lint": "turbo run lint",
    "typecheck": "turbo run typecheck"
  },
  "devDependencies": {
    "turbo": "^2.1.0"
  },
  "engines": {
    "node": ">=20"
  }
}
```

- [ ] **Step 3: Create `pnpm-workspace.yaml`**

Write `pnpm-workspace.yaml`:
```yaml
packages:
  - "apps/*"
  - "packages/*"
```

- [ ] **Step 4: Create `turbo.json`**

Write `turbo.json`:
```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "!.next/cache/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "test": {
      "dependsOn": ["^build"]
    },
    "lint": {},
    "typecheck": {
      "dependsOn": ["^build"]
    }
  }
}
```

- [ ] **Step 5: Install Turborepo**

Run: `pnpm install`

Expected: creates `pnpm-lock.yaml` and installs `turbo` as a devDependency.

- [ ] **Step 6: Verify**

Run: `pnpm turbo --version`

Expected: prints a version number `2.x.x`.

- [ ] **Step 7: Commit**

```bash
git add package.json pnpm-workspace.yaml turbo.json .nvmrc pnpm-lock.yaml
git commit -m "feat(p0): bootstrap turborepo monorepo with pnpm"
```

---

## Task 2: Initialize Next.js 15 App Shell

**Files:**
- Create: `apps/web/package.json`
- Create: `apps/web/tsconfig.json`
- Create: `apps/web/next.config.mjs`
- Create: `apps/web/tailwind.config.ts`
- Create: `apps/web/postcss.config.mjs`
- Create: `apps/web/app/layout.tsx`
- Create: `apps/web/app/page.tsx`
- Create: `apps/web/app/globals.css`
- Create: `apps/web/lib/env.ts`
- Create: `apps/web/vitest.config.ts`
- Create: `apps/web/__tests__/smoke.test.ts`

- [ ] **Step 1: Create `apps/web/package.json`**

Write `apps/web/package.json`:
```json
{
  "name": "@uncover/web",
  "version": "0.0.0",
  "private": true,
  "scripts": {
    "dev": "next dev --port 3000",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "typecheck": "tsc --noEmit",
    "test": "vitest run"
  },
  "dependencies": {
    "next": "15.0.3",
    "react": "19.0.0-rc-66855b96-20241106",
    "react-dom": "19.0.0-rc-66855b96-20241106"
  },
  "devDependencies": {
    "@types/node": "^22.7.0",
    "@types/react": "^18.3.12",
    "@types/react-dom": "^18.3.1",
    "autoprefixer": "^10.4.20",
    "postcss": "^8.4.47",
    "tailwindcss": "^3.4.14",
    "typescript": "^5.6.3",
    "vitest": "^2.1.4",
    "eslint": "^9.13.0",
    "eslint-config-next": "15.0.3"
  }
}
```

- [ ] **Step 2: Create `apps/web/tsconfig.json`**

Write `apps/web/tsconfig.json`:
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [{ "name": "next" }],
    "paths": { "@/*": ["./*"] }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

- [ ] **Step 3: Create `apps/web/next.config.mjs`**

Write `apps/web/next.config.mjs`:
```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,
  env: {
    NEXT_PUBLIC_API_URL: process.env.NEXT_PUBLIC_API_URL
  }
};

export default nextConfig;
```

- [ ] **Step 4: Create `apps/web/tailwind.config.ts`**

Write `apps/web/tailwind.config.ts`:
```typescript
import type { Config } from "tailwindcss";

const config: Config = {
  content: [
    "./app/**/*.{js,ts,jsx,tsx}",
    "./components/**/*.{js,ts,jsx,tsx}",
    "./lib/**/*.{js,ts,jsx,tsx}"
  ],
  theme: { extend: {} },
  plugins: []
};

export default config;
```

- [ ] **Step 5: Create `apps/web/postcss.config.mjs`**

Write `apps/web/postcss.config.mjs`:
```javascript
export default {
  plugins: {
    tailwindcss: {},
    autoprefixer: {}
  }
};
```

- [ ] **Step 6: Create `apps/web/app/globals.css`**

Write `apps/web/app/globals.css`:
```css
@tailwind base;
@tailwind components;
@tailwind utilities;

:root {
  color-scheme: dark;
}

html, body {
  background: #0a0a0a;
  color: #f5f5f5;
  font-family: system-ui, -apple-system, sans-serif;
}
```

- [ ] **Step 7: Create `apps/web/app/layout.tsx`**

Write `apps/web/app/layout.tsx`:
```tsx
import "./globals.css";
import type { Metadata } from "next";

export const metadata: Metadata = {
  title: "Uncover",
  description: "A 5-agent AI suite for SEO, GEO, and AI search optimization"
};

export default function RootLayout({
  children
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}
```

- [ ] **Step 8: Create `apps/web/app/page.tsx`**

Write `apps/web/app/page.tsx`:
```tsx
export default function LandingPage() {
  return (
    <main className="min-h-screen flex items-center justify-center p-8">
      <div className="max-w-2xl text-center">
        <h1 className="text-5xl font-bold mb-4">Uncover</h1>
        <p className="text-xl text-neutral-400">
          A 5-agent AI suite for SEO, GEO, and AI search optimization.
        </p>
        <p className="mt-8 text-sm text-neutral-500">Coming soon.</p>
      </div>
    </main>
  );
}
```

- [ ] **Step 9: Create `apps/web/lib/env.ts`**

Write `apps/web/lib/env.ts`:
```typescript
function required(name: string): string {
  const value = process.env[name];
  if (!value) {
    throw new Error(`Missing required environment variable: ${name}`);
  }
  return value;
}

export const env = {
  NEXT_PUBLIC_API_URL: required("NEXT_PUBLIC_API_URL")
};
```

- [ ] **Step 10: Create `apps/web/vitest.config.ts`**

Write `apps/web/vitest.config.ts`:
```typescript
import { defineConfig } from "vitest/config";
import path from "node:path";

export default defineConfig({
  resolve: {
    alias: {
      "@": path.resolve(__dirname, ".")
    }
  },
  test: {
    environment: "node",
    include: ["__tests__/**/*.test.ts"]
  }
});
```

- [ ] **Step 11: Create `apps/web/__tests__/smoke.test.ts`**

Write `apps/web/__tests__/smoke.test.ts`:
```typescript
import { describe, it, expect } from "vitest";

describe("web app smoke test", () => {
  it("math still works", () => {
    expect(1 + 1).toBe(2);
  });
});
```

- [ ] **Step 12: Install dependencies**

Run: `pnpm install`

Expected: downloads Next.js 15, React 19 RC, tailwind, vitest, etc.

- [ ] **Step 13: Run the typecheck and test**

Run: `pnpm --filter @uncover/web typecheck && pnpm --filter @uncover/web test`

Expected: typecheck passes with no errors; smoke test passes.

- [ ] **Step 14: Run the dev server and visit locally**

Run: `NEXT_PUBLIC_API_URL=http://localhost:8000 pnpm --filter @uncover/web dev`

Open `http://localhost:3000` in a browser. Expected: the Uncover landing page renders with the headline and "Coming soon." Stop the dev server (Ctrl+C) before continuing.

- [ ] **Step 15: Commit**

```bash
git add apps/web pnpm-lock.yaml
git commit -m "feat(p0): initialize next.js 15 app shell with tailwind and vitest"
```

---

## Task 3: Initialize FastAPI App with uv

**Files:**
- Create: `apps/api/pyproject.toml`
- Create: `apps/api/.python-version`
- Create: `apps/api/app/__init__.py`
- Create: `apps/api/app/main.py`
- Create: `apps/api/app/config.py`
- Create: `apps/api/app/routes/__init__.py`
- Create: `apps/api/app/routes/health.py`
- Create: `apps/api/tests/__init__.py`
- Create: `apps/api/tests/conftest.py`
- Create: `apps/api/tests/test_health.py`

- [ ] **Step 1: Create `apps/api/.python-version`**

Write `apps/api/.python-version`:
```
3.12
```

- [ ] **Step 2: Create `apps/api/pyproject.toml`**

Write `apps/api/pyproject.toml`:
```toml
[project]
name = "uncover-api"
version = "0.0.0"
description = "Uncover Python agent backend"
requires-python = ">=3.12"
dependencies = [
  "fastapi>=0.115.0",
  "uvicorn[standard]>=0.32.0",
  "pydantic>=2.9.0",
  "pydantic-settings>=2.6.0",
  "supabase>=2.9.0",
  "httpx>=0.27.0",
  "python-dotenv>=1.0.1",
]

[dependency-groups]
dev = [
  "pytest>=8.3.0",
  "pytest-asyncio>=0.24.0",
  "ruff>=0.7.0",
  "mypy>=1.13.0",
]

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
pythonpath = ["."]

[tool.uv]
package = false
```

- [ ] **Step 3: Create `apps/api/app/__init__.py`**

Write `apps/api/app/__init__.py`:
```python
```

(Empty file — marks `app` as a package.)

- [ ] **Step 4: Create `apps/api/app/config.py`**

Write `apps/api/app/config.py`:
```python
from functools import lru_cache

from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    supabase_url: str = ""
    supabase_service_role_key: str = ""
    supabase_anon_key: str = ""
    langsmith_api_key: str = ""
    langsmith_project: str = "uncover"
    environment: str = "development"
    cors_origins: str = "http://localhost:3000"

    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",
    )


@lru_cache
def get_settings() -> Settings:
    return Settings()
```

- [ ] **Step 5: Create `apps/api/app/routes/__init__.py`**

Write `apps/api/app/routes/__init__.py`:
```python
```

(Empty file.)

- [ ] **Step 6: Create `apps/api/app/routes/health.py`**

Write `apps/api/app/routes/health.py`:
```python
from fastapi import APIRouter

router = APIRouter()


@router.get("/health")
async def health() -> dict[str, str]:
    return {"status": "ok"}
```

- [ ] **Step 7: Create `apps/api/app/main.py`**

Write `apps/api/app/main.py`:
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.config import get_settings
from app.routes import health


def create_app() -> FastAPI:
    settings = get_settings()
    app = FastAPI(title="Uncover API", version="0.0.0")

    app.add_middleware(
        CORSMiddleware,
        allow_origins=[o.strip() for o in settings.cors_origins.split(",")],
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

    app.include_router(health.router)

    return app


app = create_app()
```

- [ ] **Step 8: Create `apps/api/tests/__init__.py`**

Write `apps/api/tests/__init__.py`:
```python
```

(Empty file.)

- [ ] **Step 9: Create `apps/api/tests/conftest.py`**

Write `apps/api/tests/conftest.py`:
```python
import pytest
from fastapi.testclient import TestClient

from app.main import create_app


@pytest.fixture
def client() -> TestClient:
    app = create_app()
    return TestClient(app)
```

- [ ] **Step 10: Write the failing health test**

Write `apps/api/tests/test_health.py`:
```python
from fastapi.testclient import TestClient


def test_health_endpoint_returns_ok(client: TestClient) -> None:
    response = client.get("/health")
    assert response.status_code == 200
    assert response.json() == {"status": "ok"}
```

- [ ] **Step 11: Install Python deps and run the test**

```bash
cd apps/api
uv sync --all-groups
uv run pytest tests/test_health.py -v
```

Expected: `test_health_endpoint_returns_ok PASSED`.

- [ ] **Step 12: Run the dev server and hit health manually**

```bash
cd apps/api
uv run uvicorn app.main:app --reload --port 8000
```

In another terminal: `curl http://localhost:8000/health`
Expected: `{"status":"ok"}`. Stop the server (Ctrl+C) before continuing.

- [ ] **Step 13: Commit**

```bash
git add apps/api
git commit -m "feat(p0): initialize fastapi app with uv + health endpoint"
```

---

## Task 4: Write and Apply Supabase P0 Schema Migration

**Files:**
- Create: `supabase/migrations/20260415_p0_foundations.sql`
- Create: `supabase/README.md`

**Manual prerequisite:** Create a new Supabase project at [supabase.com/dashboard](https://supabase.com/dashboard). Name it `uncover`. Choose the nearest region. Record the project URL, anon key, and service role key for Task 7.

- [ ] **Step 1: Write the P0 migration SQL**

Write `supabase/migrations/20260415_p0_foundations.sql`:
```sql
-- Phase 0 foundational schema for Uncover.
-- Applied manually via the Supabase SQL editor (no local CLI).

-- =========================================================================
-- Tables
-- =========================================================================

create table if not exists public.google_connections (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  google_email text not null,
  gsc_sites jsonb not null default '[]'::jsonb,
  ga4_properties jsonb not null default '[]'::jsonb,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create table if not exists public.runs (
  run_id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users(id) on delete set null,
  agent_type text not null,
  status text not null default 'queued'
    check (status in ('queued', 'running', 'succeeded', 'failed', 'budget_exceeded')),
  started_at timestamptz,
  finished_at timestamptz,
  cost_usd numeric(10, 4) default 0,
  input jsonb,
  error text,
  created_at timestamptz not null default now()
);

create index if not exists runs_user_id_idx on public.runs(user_id);
create index if not exists runs_agent_type_idx on public.runs(agent_type);
create index if not exists runs_status_idx on public.runs(status);

create table if not exists public.run_steps (
  id bigserial primary key,
  run_id uuid not null references public.runs(run_id) on delete cascade,
  step_name text not null,
  status text not null default 'running'
    check (status in ('running', 'succeeded', 'failed', 'skipped')),
  started_at timestamptz not null default now(),
  finished_at timestamptz,
  payload jsonb
);

create index if not exists run_steps_run_id_idx on public.run_steps(run_id);

create table if not exists public.case_studies (
  slug text primary key,
  title text not null,
  agent_type text not null,
  target_brand text,
  run_id uuid references public.runs(run_id),
  summary_md text,
  published_at timestamptz
);

-- =========================================================================
-- Row-Level Security
-- =========================================================================

alter table public.google_connections enable row level security;
alter table public.runs enable row level security;
alter table public.run_steps enable row level security;
alter table public.case_studies enable row level security;

-- google_connections: strict per-user isolation
create policy "users read own google connections"
  on public.google_connections for select
  using (auth.uid() = user_id);

create policy "users write own google connections"
  on public.google_connections for all
  using (auth.uid() = user_id)
  with check (auth.uid() = user_id);

-- runs: users read and write their own runs
create policy "users read own runs"
  on public.runs for select
  using (auth.uid() = user_id);

create policy "users insert own runs"
  on public.runs for insert
  with check (auth.uid() = user_id);

create policy "users update own runs"
  on public.runs for update
  using (auth.uid() = user_id);

-- service role bypasses RLS, so the FastAPI backend using service_role_key
-- can write runs without a user context. We still enforce reads through
-- user JWT at the API layer.

-- run_steps: readable/writable only when the parent run is owned
create policy "users read own run steps"
  on public.run_steps for select
  using (
    exists (
      select 1 from public.runs
      where runs.run_id = run_steps.run_id
        and runs.user_id = auth.uid()
    )
  );

create policy "users insert own run steps"
  on public.run_steps for insert
  with check (
    exists (
      select 1 from public.runs
      where runs.run_id = run_steps.run_id
        and runs.user_id = auth.uid()
    )
  );

-- case_studies: public-readable (anon can SELECT), writes restricted to service role
create policy "anyone reads published case studies"
  on public.case_studies for select
  to anon, authenticated
  using (published_at is not null);

-- =========================================================================
-- Helper function: keep updated_at fresh
-- =========================================================================

create or replace function public.set_updated_at()
returns trigger
language plpgsql
as $$
begin
  new.updated_at = now();
  return new;
end;
$$;

create trigger google_connections_set_updated_at
  before update on public.google_connections
  for each row execute function public.set_updated_at();
```

- [ ] **Step 2: Write `supabase/README.md` documenting manual migration workflow**

Write `supabase/README.md`:
```markdown
# Supabase migrations — Uncover

Migrations are applied **manually via the Supabase dashboard SQL editor** because no local Supabase CLI is installed.

## Apply a new migration

1. Open the Supabase dashboard for the `uncover` project.
2. Go to **SQL Editor** → **New query**.
3. Copy-paste the full contents of the migration file from `supabase/migrations/`.
4. Run the query. Verify no errors.
5. Commit the migration file to the repo so the history is versioned.

## File naming

`YYYYMMDD_short_description.sql`

## Current migrations

- `20260415_p0_foundations.sql` — Phase 0 schema (runs, run_steps, google_connections, case_studies) with RLS policies.
```

- [ ] **Step 3: Apply the migration via the Supabase dashboard**

Manual action:
1. Open the Supabase dashboard for the `uncover` project.
2. Go to SQL Editor → New query.
3. Copy-paste the full contents of `supabase/migrations/20260415_p0_foundations.sql`.
4. Click **Run**.

Expected: "Success. No rows returned." Verify all 4 tables exist under the **Table Editor**: `google_connections`, `runs`, `run_steps`, `case_studies`.

- [ ] **Step 4: Verify RLS is enabled**

In the Supabase dashboard, go to **Authentication → Policies**. You should see:
- `google_connections`: 2 policies (read own, write own)
- `runs`: 3 policies (read own, insert own, update own)
- `run_steps`: 2 policies (read own, insert own)
- `case_studies`: 1 policy (anyone reads published)

If any are missing, re-run the migration SQL.

- [ ] **Step 5: Commit**

```bash
git add supabase/migrations/20260415_p0_foundations.sql supabase/README.md
git commit -m "feat(p0): add supabase schema migration with rls policies"
```

---

## Task 5: Configure Supabase Auth with Google OAuth

**Files:** None in this task — entirely dashboard configuration. Records credentials you'll use in Task 7.

- [ ] **Step 1: Create a Google Cloud OAuth app**

1. Open [console.cloud.google.com](https://console.cloud.google.com).
2. Create a new project named `uncover-auth` (or reuse an existing one).
3. Navigate to **APIs & Services → Credentials → Create Credentials → OAuth client ID**.
4. Configure the OAuth consent screen:
   - User type: External
   - App name: Uncover
   - User support email: your email
   - Scopes: `.../auth/userinfo.email`, `.../auth/userinfo.profile`, `openid`
   - Test users: add your own email
5. Create the OAuth client ID:
   - Application type: **Web application**
   - Name: Uncover
   - Authorized JavaScript origins:
     - `http://localhost:3000`
     - `https://uncover.robinching.com` (will resolve after Task 13)
   - Authorized redirect URIs:
     - `https://<your-supabase-project-ref>.supabase.co/auth/v1/callback`
6. Record the **Client ID** and **Client Secret**.

- [ ] **Step 2: Enable Google provider in Supabase**

1. In the Supabase dashboard: **Authentication → Providers → Google**.
2. Toggle Google on.
3. Paste the **Client ID** and **Client Secret** from Step 1.
4. Copy the Supabase **Callback URL** shown on the page and confirm it matches what you entered in Google Cloud (Step 1.5).
5. Click **Save**.

- [ ] **Step 3: Configure Site URL and redirect URLs in Supabase**

1. **Authentication → URL Configuration**.
2. **Site URL**: `http://localhost:3000` (for now — updated to production in Task 13).
3. **Redirect URLs**: add both:
   - `http://localhost:3000/**`
   - `https://uncover.robinching.com/**`
4. Save.

- [ ] **Step 4: Verify the Google sign-in flow**

Quickest test: use Supabase's built-in **Authentication → Users → Invite** UI or visit `https://<project-ref>.supabase.co/auth/v1/authorize?provider=google` in a browser. You should be redirected to Google's consent screen, and after consent, to the callback URL (which will 404 for now because the frontend isn't wired to handle it — that's fine, we just need to confirm Google accepts the request).

Expected: Google displays the consent screen without errors. A new user row appears in **Authentication → Users** if you complete the flow.

- [ ] **Step 5: No commit**

This task made no file changes. Move to Task 6.

---

## Task 6: Verify Row-Level Security with Two Test Users

**Files:** None — verification-only via Supabase dashboard SQL editor.

- [ ] **Step 1: Create two test users**

In the Supabase dashboard: **Authentication → Users → Add user → Create new user**.
- User A: email `test-a@uncover.test`, password any strong value, auto-confirm email
- User B: email `test-b@uncover.test`, password any strong value, auto-confirm email

Record both user IDs (UUIDs) from the users list.

- [ ] **Step 2: Insert a run as User A (via SQL editor, impersonated)**

In the Supabase SQL editor, run:
```sql
-- Switch to user A's context
set local role authenticated;
set local request.jwt.claims = '{"sub": "<USER_A_UUID>", "role": "authenticated"}';

-- Insert a run owned by user A
insert into public.runs (user_id, agent_type, input)
values ('<USER_A_UUID>'::uuid, 'demo', '{"message": "rls test"}'::jsonb)
returning run_id;
```

Replace `<USER_A_UUID>` with the actual UUID. Expected: returns one `run_id`. Record it.

- [ ] **Step 3: Attempt to read the run as User B**

Run:
```sql
set local role authenticated;
set local request.jwt.claims = '{"sub": "<USER_B_UUID>", "role": "authenticated"}';

select run_id, user_id, agent_type
from public.runs;
```

Expected: **zero rows returned**. User B should not see User A's run.

- [ ] **Step 4: Confirm User A can see their own run**

Run:
```sql
set local role authenticated;
set local request.jwt.claims = '{"sub": "<USER_A_UUID>", "role": "authenticated"}';

select run_id, user_id, agent_type
from public.runs;
```

Expected: **one row returned** with the `run_id` from Step 2.

- [ ] **Step 5: Clean up the test row**

Run:
```sql
-- Back to service role (superuser) to clean up
reset role;
delete from public.runs where agent_type = 'demo' and input->>'message' = 'rls test';
```

- [ ] **Step 6: Document RLS verification in a note**

Write a one-line note at the top of `supabase/README.md` (append, don't overwrite):
```markdown

## RLS verification

RLS isolation verified on 2026-04-15 via the two-user test in `docs/plans/2026-04-15-phase-0-foundations.md` Task 6. Re-run before launch (see spec Section 10 Definition of Done).
```

- [ ] **Step 7: Commit**

```bash
git add supabase/README.md
git commit -m "docs(p0): record rls isolation verification"
```

---

## Task 7: Build run_logger Helper (TDD)

**Files:**
- Create: `apps/api/app/db/__init__.py`
- Create: `apps/api/app/db/supabase_client.py`
- Create: `apps/api/app/logging/__init__.py`
- Create: `apps/api/app/logging/run_logger.py`
- Create: `apps/api/tests/test_run_logger.py`

**Prerequisite:** Before running tests, create `apps/api/.env` with:
```
SUPABASE_URL=https://<your-project-ref>.supabase.co
SUPABASE_SERVICE_ROLE_KEY=<your-service-role-key>
SUPABASE_ANON_KEY=<your-anon-key>
```

Add `apps/api/.env` to `.gitignore` verification — it's already covered by the root `.gitignore`'s `.env` rule.

- [ ] **Step 1: Create `apps/api/app/db/__init__.py`**

Write `apps/api/app/db/__init__.py`:
```python
```

(Empty file.)

- [ ] **Step 2: Create `apps/api/app/db/supabase_client.py`**

Write `apps/api/app/db/supabase_client.py`:
```python
from functools import lru_cache

from supabase import Client, create_client

from app.config import get_settings


@lru_cache
def get_supabase_client() -> Client:
    settings = get_settings()
    if not settings.supabase_url or not settings.supabase_service_role_key:
        raise RuntimeError(
            "SUPABASE_URL and SUPABASE_SERVICE_ROLE_KEY must be set"
        )
    return create_client(settings.supabase_url, settings.supabase_service_role_key)
```

- [ ] **Step 3: Create `apps/api/app/logging/__init__.py`**

Write `apps/api/app/logging/__init__.py`:
```python
```

(Empty file.)

- [ ] **Step 4: Write the failing run_logger test**

Write `apps/api/tests/test_run_logger.py`:
```python
from unittest.mock import MagicMock
from uuid import UUID

import pytest

from app.logging.run_logger import RunLogger


@pytest.fixture
def fake_client() -> MagicMock:
    client = MagicMock()
    # Mock the chained builder pattern used by supabase-py
    runs_table = MagicMock()
    run_steps_table = MagicMock()

    def table(name: str) -> MagicMock:
        if name == "runs":
            return runs_table
        if name == "run_steps":
            return run_steps_table
        raise ValueError(name)

    client.table.side_effect = table

    # Simulate returning a row with a run_id
    runs_insert_result = MagicMock()
    runs_insert_result.data = [
        {"run_id": "11111111-1111-1111-1111-111111111111"}
    ]
    runs_table.insert.return_value.execute.return_value = runs_insert_result

    # run_steps insert returns its row too
    step_insert_result = MagicMock()
    step_insert_result.data = [{"id": 1}]
    run_steps_table.insert.return_value.execute.return_value = step_insert_result

    return client


def test_create_run_returns_run_id(fake_client: MagicMock) -> None:
    logger = RunLogger(client=fake_client)
    run_id = logger.create_run(agent_type="demo", user_id=None, input={"x": 1})
    assert isinstance(run_id, UUID)
    assert str(run_id) == "11111111-1111-1111-1111-111111111111"
    fake_client.table.assert_any_call("runs")


def test_start_step_inserts_row(fake_client: MagicMock) -> None:
    logger = RunLogger(client=fake_client)
    run_id = UUID("11111111-1111-1111-1111-111111111111")
    step_id = logger.start_step(run_id=run_id, step_name="count_to_three")
    assert step_id == 1
    fake_client.table.assert_any_call("run_steps")


def test_finish_step_updates_row(fake_client: MagicMock) -> None:
    update_result = MagicMock()
    update_result.data = [{"id": 42}]
    fake_client.table("run_steps").update.return_value.eq.return_value.execute.return_value = update_result

    logger = RunLogger(client=fake_client)
    logger.finish_step(step_id=42, payload={"value": 3})
    fake_client.table.assert_any_call("run_steps")


def test_finish_run_updates_status(fake_client: MagicMock) -> None:
    update_result = MagicMock()
    update_result.data = [{"run_id": "11111111-1111-1111-1111-111111111111"}]
    fake_client.table("runs").update.return_value.eq.return_value.execute.return_value = update_result

    logger = RunLogger(client=fake_client)
    logger.finish_run(
        run_id=UUID("11111111-1111-1111-1111-111111111111"),
        status="succeeded",
    )
    fake_client.table.assert_any_call("runs")
```

- [ ] **Step 5: Run the test to verify it fails**

```bash
cd apps/api
uv run pytest tests/test_run_logger.py -v
```

Expected: `ModuleNotFoundError: No module named 'app.logging.run_logger'` or `ImportError: cannot import name 'RunLogger'`.

- [ ] **Step 6: Implement `run_logger.py`**

Write `apps/api/app/logging/run_logger.py`:
```python
from datetime import datetime, timezone
from typing import Any, Literal
from uuid import UUID

from supabase import Client

from app.db.supabase_client import get_supabase_client

RunStatus = Literal["queued", "running", "succeeded", "failed", "budget_exceeded"]
StepStatus = Literal["running", "succeeded", "failed", "skipped"]


class RunLogger:
    def __init__(self, client: Client | None = None) -> None:
        self._client = client or get_supabase_client()

    def create_run(
        self,
        *,
        agent_type: str,
        user_id: UUID | None,
        input: dict[str, Any],
    ) -> UUID:
        payload = {
            "agent_type": agent_type,
            "user_id": str(user_id) if user_id else None,
            "input": input,
            "status": "running",
            "started_at": datetime.now(timezone.utc).isoformat(),
        }
        result = self._client.table("runs").insert(payload).execute()
        row = result.data[0]
        return UUID(row["run_id"])

    def start_step(self, *, run_id: UUID, step_name: str) -> int:
        payload = {
            "run_id": str(run_id),
            "step_name": step_name,
            "status": "running",
        }
        result = self._client.table("run_steps").insert(payload).execute()
        return int(result.data[0]["id"])

    def finish_step(
        self,
        *,
        step_id: int,
        payload: dict[str, Any] | None = None,
        status: StepStatus = "succeeded",
    ) -> None:
        update = {
            "status": status,
            "finished_at": datetime.now(timezone.utc).isoformat(),
        }
        if payload is not None:
            update["payload"] = payload
        self._client.table("run_steps").update(update).eq("id", step_id).execute()

    def finish_run(
        self,
        *,
        run_id: UUID,
        status: RunStatus,
        error: str | None = None,
        cost_usd: float = 0.0,
    ) -> None:
        update: dict[str, Any] = {
            "status": status,
            "finished_at": datetime.now(timezone.utc).isoformat(),
            "cost_usd": cost_usd,
        }
        if error is not None:
            update["error"] = error
        self._client.table("runs").update(update).eq("run_id", str(run_id)).execute()
```

- [ ] **Step 7: Run tests to verify they pass**

```bash
cd apps/api
uv run pytest tests/test_run_logger.py -v
```

Expected: all 4 tests PASS.

- [ ] **Step 8: Commit**

```bash
git add apps/api/app/db apps/api/app/logging apps/api/tests/test_run_logger.py
git commit -m "feat(p0): add run_logger helper with tdd unit tests"
```

---

## Task 8: Build Toy Demo Agent (TDD)

**Files:**
- Create: `apps/api/app/agents/__init__.py`
- Create: `apps/api/app/agents/demo/__init__.py`
- Create: `apps/api/app/agents/demo/counter.py`
- Create: `apps/api/tests/test_counter.py`

- [ ] **Step 1: Create `apps/api/app/agents/__init__.py` and `apps/api/app/agents/demo/__init__.py`**

Write both files as empty:
```python
```

- [ ] **Step 2: Write the failing counter test**

Write `apps/api/tests/test_counter.py`:
```python
import pytest

from app.agents.demo.counter import counter_agent


@pytest.mark.asyncio
async def test_counter_yields_three_events() -> None:
    events = []
    async for event in counter_agent(delay_seconds=0):
        events.append(event)

    assert len(events) == 3
    assert events[0]["value"] == 1
    assert events[1]["value"] == 2
    assert events[2]["value"] == 3
    assert all(event["step_name"] == "count" for event in events)
    assert all(event["status"] == "succeeded" for event in events)


@pytest.mark.asyncio
async def test_counter_respects_delay() -> None:
    import time

    start = time.monotonic()
    async for _ in counter_agent(delay_seconds=0.05):
        pass
    elapsed = time.monotonic() - start
    # 3 events * 50ms = 150ms minimum
    assert elapsed >= 0.12
```

- [ ] **Step 3: Run test to verify it fails**

```bash
cd apps/api
uv run pytest tests/test_counter.py -v
```

Expected: `ModuleNotFoundError: No module named 'app.agents.demo.counter'`.

- [ ] **Step 4: Implement `counter.py`**

Write `apps/api/app/agents/demo/counter.py`:
```python
import asyncio
from collections.abc import AsyncIterator
from typing import Any


async def counter_agent(
    *,
    delay_seconds: float = 0.5,
) -> AsyncIterator[dict[str, Any]]:
    """A toy async agent that yields 3 status events with a delay between each.

    Used by Phase 0 to prove the SSE streaming pipeline end-to-end without
    depending on LLM APIs. Real agents in P1-P4 follow the same yield shape:
    each event has step_name, status, and an opaque payload.
    """
    for i in range(1, 4):
        await asyncio.sleep(delay_seconds)
        yield {
            "step_name": "count",
            "status": "succeeded",
            "value": i,
            "message": f"Step {i} of 3",
        }
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
cd apps/api
uv run pytest tests/test_counter.py -v
```

Expected: both tests PASS.

- [ ] **Step 6: Commit**

```bash
git add apps/api/app/agents apps/api/tests/test_counter.py
git commit -m "feat(p0): add toy counter_agent async generator with tests"
```

---

## Task 9: Build SSE Endpoint Wired to run_logger

**Files:**
- Create: `apps/api/app/routes/stream.py`
- Modify: `apps/api/app/main.py`
- Create: `apps/api/tests/test_stream.py`

- [ ] **Step 1: Write the failing stream test**

Write `apps/api/tests/test_stream.py`:
```python
from unittest.mock import MagicMock, patch
from uuid import UUID

import pytest
from fastapi.testclient import TestClient

from app.main import create_app


@pytest.fixture
def app_with_mocked_logger():
    mock_logger = MagicMock()
    mock_logger.create_run.return_value = UUID("22222222-2222-2222-2222-222222222222")
    mock_logger.start_step.return_value = 42

    with patch("app.routes.stream.RunLogger", return_value=mock_logger):
        app = create_app()
        yield TestClient(app), mock_logger


def test_demo_stream_emits_three_sse_events(app_with_mocked_logger) -> None:
    client, mock_logger = app_with_mocked_logger

    with client.stream("POST", "/run/demo") as response:
        assert response.status_code == 200
        assert response.headers["content-type"].startswith("text/event-stream")
        lines = [line for line in response.iter_lines() if line]

    # Three "data:" lines plus the "done" terminator
    data_lines = [l for l in lines if l.startswith("data:")]
    assert len(data_lines) >= 3

    mock_logger.create_run.assert_called_once()
    assert mock_logger.start_step.call_count == 3
    assert mock_logger.finish_step.call_count == 3
    mock_logger.finish_run.assert_called_once()
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd apps/api
uv run pytest tests/test_stream.py -v
```

Expected: `ModuleNotFoundError: No module named 'app.routes.stream'` or 404.

- [ ] **Step 3: Implement `stream.py`**

Write `apps/api/app/routes/stream.py`:
```python
import json
from collections.abc import AsyncIterator

from fastapi import APIRouter
from fastapi.responses import StreamingResponse

from app.agents.demo.counter import counter_agent
from app.logging.run_logger import RunLogger

router = APIRouter()


def _sse_format(data: dict) -> str:
    return f"data: {json.dumps(data)}\n\n"


async def _demo_event_stream() -> AsyncIterator[str]:
    logger = RunLogger()
    run_id = logger.create_run(agent_type="demo", user_id=None, input={})

    yield _sse_format({"event": "run_started", "run_id": str(run_id)})

    try:
        async for event in counter_agent(delay_seconds=0.3):
            step_id = logger.start_step(run_id=run_id, step_name=event["step_name"])
            logger.finish_step(step_id=step_id, payload=event)
            yield _sse_format({"event": "step", **event, "run_id": str(run_id)})
    except Exception as exc:  # noqa: BLE001 - top-level boundary
        logger.finish_run(run_id=run_id, status="failed", error=str(exc))
        yield _sse_format({"event": "run_failed", "run_id": str(run_id), "error": str(exc)})
        return

    logger.finish_run(run_id=run_id, status="succeeded")
    yield _sse_format({"event": "run_succeeded", "run_id": str(run_id)})


@router.post("/run/demo")
async def run_demo() -> StreamingResponse:
    return StreamingResponse(
        _demo_event_stream(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no",
        },
    )
```

- [ ] **Step 4: Wire the stream router into `main.py`**

Modify `apps/api/app/main.py`. Replace the existing `create_app` function with:
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.config import get_settings
from app.routes import health, stream


def create_app() -> FastAPI:
    settings = get_settings()
    app = FastAPI(title="Uncover API", version="0.0.0")

    app.add_middleware(
        CORSMiddleware,
        allow_origins=[o.strip() for o in settings.cors_origins.split(",")],
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

    app.include_router(health.router)
    app.include_router(stream.router)

    return app


app = create_app()
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
cd apps/api
uv run pytest tests/test_stream.py tests/test_health.py tests/test_run_logger.py tests/test_counter.py -v
```

Expected: all tests PASS.

- [ ] **Step 6: Commit**

```bash
git add apps/api/app/routes/stream.py apps/api/app/main.py apps/api/tests/test_stream.py
git commit -m "feat(p0): add sse /run/demo endpoint wired to run_logger"
```

---

## Task 10: Build Frontend SSE Consumer Page

**Files:**
- Create: `apps/web/lib/sse.ts`
- Create: `apps/web/app/demo/page.tsx`
- Create: `apps/web/__tests__/sse.test.ts`

- [ ] **Step 1: Write the failing SSE client test**

Write `apps/web/__tests__/sse.test.ts`:
```typescript
import { describe, it, expect } from "vitest";
import { parseSSELine } from "@/lib/sse";

describe("parseSSELine", () => {
  it("parses a valid data line", () => {
    const result = parseSSELine('data: {"event":"step","value":1}');
    expect(result).toEqual({ event: "step", value: 1 });
  });

  it("returns null for empty lines", () => {
    expect(parseSSELine("")).toBeNull();
  });

  it("returns null for non-data lines", () => {
    expect(parseSSELine("event: message")).toBeNull();
  });

  it("returns null for malformed JSON", () => {
    expect(parseSSELine("data: {not json")).toBeNull();
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
cd apps/web
pnpm test
```

Expected: `Cannot find module '@/lib/sse'`.

- [ ] **Step 3: Implement `lib/sse.ts`**

Write `apps/web/lib/sse.ts`:
```typescript
export type SSEEvent = {
  event: string;
  [key: string]: unknown;
};

export function parseSSELine(line: string): SSEEvent | null {
  if (!line.startsWith("data:")) {
    return null;
  }
  const jsonStr = line.slice(5).trim();
  if (!jsonStr) {
    return null;
  }
  try {
    return JSON.parse(jsonStr) as SSEEvent;
  } catch {
    return null;
  }
}

export async function* streamSSE(
  url: string,
  init?: RequestInit
): AsyncGenerator<SSEEvent> {
  const response = await fetch(url, {
    ...init,
    method: init?.method ?? "POST",
    headers: {
      ...init?.headers,
      Accept: "text/event-stream"
    }
  });

  if (!response.ok || !response.body) {
    throw new Error(`SSE request failed: ${response.status}`);
  }

  const reader = response.body.getReader();
  const decoder = new TextDecoder();
  let buffer = "";

  while (true) {
    const { value, done } = await reader.read();
    if (done) {
      break;
    }
    buffer += decoder.decode(value, { stream: true });
    const parts = buffer.split("\n\n");
    buffer = parts.pop() ?? "";
    for (const part of parts) {
      for (const line of part.split("\n")) {
        const parsed = parseSSELine(line);
        if (parsed) {
          yield parsed;
        }
      }
    }
  }
}
```

- [ ] **Step 4: Run the test to verify it passes**

```bash
cd apps/web
pnpm test
```

Expected: 4 `parseSSELine` tests + the existing smoke test all PASS.

- [ ] **Step 5: Create the demo page**

Write `apps/web/app/demo/page.tsx`:
```tsx
"use client";

import { useState } from "react";
import { streamSSE, type SSEEvent } from "@/lib/sse";

const API_URL = process.env.NEXT_PUBLIC_API_URL ?? "http://localhost:8000";

export default function DemoPage() {
  const [events, setEvents] = useState<SSEEvent[]>([]);
  const [running, setRunning] = useState(false);
  const [error, setError] = useState<string | null>(null);

  async function runDemo() {
    setEvents([]);
    setError(null);
    setRunning(true);

    try {
      for await (const event of streamSSE(`${API_URL}/run/demo`)) {
        setEvents((prev) => [...prev, event]);
      }
    } catch (err) {
      setError(err instanceof Error ? err.message : String(err));
    } finally {
      setRunning(false);
    }
  }

  return (
    <main className="min-h-screen p-8 max-w-2xl mx-auto">
      <h1 className="text-3xl font-bold mb-6">Uncover — P0 SSE Demo</h1>
      <p className="text-neutral-400 mb-6">
        A toy agent that counts 1 → 2 → 3. Proves the SSE pipeline end-to-end
        before real agents plug in.
      </p>

      <button
        onClick={runDemo}
        disabled={running}
        className="px-4 py-2 bg-white text-black font-medium rounded disabled:opacity-50"
      >
        {running ? "Running…" : "Run demo agent"}
      </button>

      {error && (
        <p className="mt-4 text-red-400">Error: {error}</p>
      )}

      <ul className="mt-6 space-y-2">
        {events.map((event, i) => (
          <li
            key={i}
            className="p-3 bg-neutral-900 rounded font-mono text-sm"
          >
            <pre>{JSON.stringify(event, null, 2)}</pre>
          </li>
        ))}
      </ul>
    </main>
  );
}
```

- [ ] **Step 6: Commit**

```bash
git add apps/web/lib/sse.ts apps/web/app/demo apps/web/__tests__/sse.test.ts
git commit -m "feat(p0): add sse client helper and demo consumer page"
```

---

## Task 11: Local End-to-End Verification

**Files:** None — verification only.

- [ ] **Step 1: Start the FastAPI dev server**

In terminal 1:
```bash
cd apps/api
uv run uvicorn app.main:app --reload --port 8000
```

Expected: `Uvicorn running on http://127.0.0.1:8000`.

- [ ] **Step 2: Start the Next.js dev server**

In terminal 2:
```bash
cd apps/web
NEXT_PUBLIC_API_URL=http://localhost:8000 pnpm dev
```

Expected: `Local: http://localhost:3000`.

- [ ] **Step 3: Visit the demo page in a browser**

Open `http://localhost:3000/demo`. Click **Run demo agent**.

Expected:
- Three `step` events stream in with `value: 1`, `2`, `3` (approximately 0.3 seconds apart)
- A final `run_succeeded` event
- No errors in the Next.js or FastAPI consoles

- [ ] **Step 4: Verify database rows**

In the Supabase dashboard SQL editor, run:
```sql
select run_id, agent_type, status, started_at, finished_at
from public.runs
order by started_at desc
limit 5;
```

Expected: the most recent row has `agent_type = 'demo'`, `status = 'succeeded'`, both timestamps set.

Then:
```sql
select id, run_id, step_name, status, payload
from public.run_steps
where run_id = '<MOST_RECENT_RUN_ID>'
order by id;
```

Expected: 3 rows, each with `step_name = 'count'`, `status = 'succeeded'`, and `payload.value` = 1, 2, 3.

- [ ] **Step 5: Stop both servers**

Ctrl+C in both terminals.

- [ ] **Step 6: No commit**

This task is verification-only.

---

## Task 12: Deploy FastAPI to Modal

**Files:**
- Create: `apps/api/modal_app.py`

- [ ] **Step 1: Authenticate Modal locally** (prerequisite — only once)

Run:
```bash
uv pip install modal
modal token new
```

Follow the browser prompt. Expected: `Token created successfully`.

- [ ] **Step 2: Create `apps/api/modal_app.py`**

Write `apps/api/modal_app.py`:
```python
import modal

image = (
    modal.Image.debian_slim(python_version="3.12")
    .pip_install_from_pyproject("pyproject.toml")
    .add_local_python_source("app")
)

app = modal.App("uncover-api", image=image)

secrets = [
    modal.Secret.from_name("uncover-supabase"),
    modal.Secret.from_name("uncover-langsmith"),
]


@app.function(secrets=secrets, min_containers=0, timeout=300)
@modal.asgi_app()
def fastapi_app():
    from app.main import create_app

    return create_app()
```

- [ ] **Step 3: Create Modal secrets**

Run each command, replacing placeholder values with real values:
```bash
modal secret create uncover-supabase \
  SUPABASE_URL=https://<project-ref>.supabase.co \
  SUPABASE_SERVICE_ROLE_KEY=<service-role-key> \
  SUPABASE_ANON_KEY=<anon-key> \
  CORS_ORIGINS=https://uncover.robinching.com,http://localhost:3000
```

```bash
modal secret create uncover-langsmith \
  LANGSMITH_API_KEY=<will-fill-in-task-14> \
  LANGSMITH_PROJECT=uncover
```

For LangSmith, use a placeholder value now (e.g., `pending`); Task 14 creates the real key and you'll re-run the secret create to update.

- [ ] **Step 4: Deploy to Modal**

```bash
cd apps/api
modal deploy modal_app.py
```

Expected: Modal prints a deployed URL like `https://<your-username>--uncover-api-fastapi-app.modal.run`. Record this URL.

- [ ] **Step 5: Verify the deployed health endpoint**

Run:
```bash
curl https://<your-modal-url>/health
```

Expected: `{"status":"ok"}`.

- [ ] **Step 6: Verify the deployed SSE endpoint**

Run:
```bash
curl -N -X POST https://<your-modal-url>/run/demo
```

Expected: three `data:` lines stream in over ~1 second, followed by a `run_succeeded` line. Check Supabase for a new `runs` row.

- [ ] **Step 7: Commit**

```bash
git add apps/api/modal_app.py
git commit -m "feat(p0): deploy fastapi to modal via modal_app.py"
```

---

## Task 13: Deploy Next.js to Vercel + Configure `uncover.robinching.com`

**Files:** None in the repo (Vercel dashboard configuration). The code is already deploy-ready from Task 2.

**Prerequisite:** Task 12 must be complete so you have the Modal URL.

- [ ] **Step 1: Import the GitHub repo into Vercel**

1. Push the latest commits to GitHub: `git push`
2. Open [vercel.com/new](https://vercel.com/new).
3. Click **Import Git Repository** and select `uncover`.
4. **Framework preset:** Next.js (auto-detected).
5. **Root directory:** `apps/web`
6. **Build command:** `cd ../.. && pnpm turbo run build --filter=@uncover/web` (or Vercel's default — Vercel auto-detects pnpm monorepos).
7. **Install command:** `pnpm install`
8. **Environment variables:**
   - `NEXT_PUBLIC_API_URL` = the Modal URL from Task 12 (e.g., `https://<user>--uncover-api-fastapi-app.modal.run`)
9. Click **Deploy**.

Expected: a successful production build at a `*.vercel.app` URL. Record it.

- [ ] **Step 2: Configure `uncover.robinching.com` subdomain**

1. In the Vercel project settings: **Domains → Add Domain**.
2. Enter `uncover.robinching.com`.
3. Vercel displays the DNS record to add (usually a CNAME to `cname.vercel-dns.com`).
4. Go to your `robinching.com` DNS provider (wherever you manage DNS for the domain).
5. Add the CNAME record exactly as Vercel specifies.
6. Wait 1–5 minutes for DNS propagation. Vercel shows a green check when verified.

- [ ] **Step 3: Update Modal CORS to allow the production domain**

If `CORS_ORIGINS` in the `uncover-supabase` Modal secret already includes `https://uncover.robinching.com` (it does per Task 12 Step 3), no action is needed. Otherwise, re-run the secret create command with the production URL included, then redeploy:
```bash
cd apps/api
modal deploy modal_app.py
```

- [ ] **Step 4: Update Supabase Auth URL configuration**

In the Supabase dashboard: **Authentication → URL Configuration**:
- **Site URL:** `https://uncover.robinching.com`
- Keep `http://localhost:3000/**` in the redirect URLs for dev
- Ensure `https://uncover.robinching.com/**` is in the redirect URLs
- Save

- [ ] **Step 5: Visit the production demo page**

Open `https://uncover.robinching.com/demo` in a browser. Click **Run demo agent**.

Expected: three SSE events stream in, a `run_succeeded` event appears, and Supabase shows a new `runs` row plus 3 `run_steps` rows.

- [ ] **Step 6: No commit**

This task made no repo file changes.

---

## Task 14: Configure LangSmith Project and Secrets

**Files:**
- Modify: `apps/api/app/main.py`

- [ ] **Step 1: Create a LangSmith project**

1. Open [smith.langchain.com](https://smith.langchain.com).
2. Create a new project named `uncover`.
3. Go to **Settings → API Keys → Create API Key**. Record it.

- [ ] **Step 2: Update the Modal secret with the real LangSmith API key**

```bash
modal secret create uncover-langsmith \
  LANGSMITH_API_KEY=<real-api-key> \
  LANGSMITH_PROJECT=uncover \
  --force
```

(The `--force` flag overwrites the placeholder from Task 12.)

- [ ] **Step 3: Wire LangSmith env vars into `main.py` startup**

Modify `apps/api/app/main.py`. Replace the top imports and `create_app` with:
```python
import os

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.config import get_settings
from app.routes import health, stream


def create_app() -> FastAPI:
    settings = get_settings()

    # Wire LangSmith tracing for any LangGraph agents added in P1+.
    # Setting env vars is sufficient — LangChain/LangGraph pick these up
    # automatically. No imports required in P0.
    if settings.langsmith_api_key and settings.langsmith_api_key != "pending":
        os.environ["LANGCHAIN_TRACING_V2"] = "true"
        os.environ["LANGCHAIN_API_KEY"] = settings.langsmith_api_key
        os.environ["LANGCHAIN_PROJECT"] = settings.langsmith_project

    app = FastAPI(title="Uncover API", version="0.0.0")

    app.add_middleware(
        CORSMiddleware,
        allow_origins=[o.strip() for o in settings.cors_origins.split(",")],
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

    app.include_router(health.router)
    app.include_router(stream.router)

    return app


app = create_app()
```

- [ ] **Step 4: Redeploy**

```bash
cd apps/api
modal deploy modal_app.py
```

Expected: successful redeploy.

- [ ] **Step 5: Verify LangSmith env vars are set in Modal**

Check Modal dashboard for the `uncover-api` app → Secrets. Both `uncover-supabase` and `uncover-langsmith` should be attached. LangSmith traces won't appear until P1 adds a real LangGraph agent — that's expected.

- [ ] **Step 6: Commit**

```bash
git add apps/api/app/main.py
git commit -m "feat(p0): wire langsmith env vars for p1 agents"
```

---

## Task 15: Production End-to-End Verification

**Files:** None — final verification of the full P0 surface.

- [ ] **Step 1: Hit the production landing page**

Open `https://uncover.robinching.com` in a browser.

Expected: the "Uncover — A 5-agent AI suite…" landing page renders with proper styling.

- [ ] **Step 2: Hit the production demo page**

Open `https://uncover.robinching.com/demo`. Click **Run demo agent**.

Expected:
- Three `step` events stream in with approximately 0.3-second gaps
- A `run_succeeded` event completes
- No CORS errors in the browser console
- No errors in Modal or Vercel logs

- [ ] **Step 3: Verify database rows from production run**

In the Supabase SQL editor:
```sql
select run_id, agent_type, status, started_at, finished_at
from public.runs
where agent_type = 'demo'
order by started_at desc
limit 1;
```

Record the `run_id`, then:
```sql
select id, step_name, status, payload
from public.run_steps
where run_id = '<RUN_ID>'
order by id;
```

Expected: 3 rows with `status = 'succeeded'` and payloads containing `value` 1, 2, 3.

- [ ] **Step 4: Check Modal logs for the production run**

In the Modal dashboard, open the `uncover-api` app → latest function invocation. Look for:
- No exceptions
- Request logged for `POST /run/demo`
- Function completes cleanly

- [ ] **Step 5: Run the full test suite one more time**

```bash
cd C:/Users/robin/Documents/search-agents/uncover
pnpm --filter @uncover/web test
cd apps/api
uv run pytest -v
```

Expected: all Python tests PASS; all TypeScript tests PASS.

- [ ] **Step 6: Final commit (if anything drifted)**

```bash
cd C:/Users/robin/Documents/search-agents/uncover
git status
```

If there are uncommitted changes, commit them:
```bash
git add .
git commit -m "chore(p0): final cleanup before phase 0 sign-off"
```

Otherwise, no commit needed.

- [ ] **Step 7: Push everything to GitHub**

```bash
git push
```

- [ ] **Step 8: Mark Phase 0 complete**

Update `README.md` status line from "Design phase. Not yet implemented." to:
```markdown
## Status

Phase 0 (Foundations) complete. Hosted demo live at https://uncover.robinching.com/demo.
Next: Phase 1 — GEO Citation Tracker.
```

Commit:
```bash
git add README.md
git commit -m "docs(p0): mark phase 0 complete in readme"
git push
```

---

## Phase 0 — Done Definition

Phase 0 is complete when ALL of these are true:

- [ ] Turborepo monorepo exists with `apps/web` and `apps/api` packages
- [ ] Next.js 15 app renders a landing page at `https://uncover.robinching.com`
- [ ] FastAPI app exposes `/health` and `/run/demo` on Modal
- [ ] Supabase has the P0 schema applied with RLS policies active
- [ ] RLS verified: a second test user cannot read the first user's runs
- [ ] Supabase Auth Google OAuth provider is enabled and the consent screen loads
- [ ] `run_logger` helper writes `runs` and `run_steps` rows and is unit-tested
- [ ] The toy `counter_agent` yields 3 events over SSE end-to-end in production
- [ ] A `POST /run/demo` against production creates one `runs` row and three `run_steps` rows
- [ ] LangSmith project exists and `LANGCHAIN_*` env vars are wired into FastAPI (no traces expected until P1)
- [ ] All Python and TypeScript tests pass locally and on fresh checkout
- [ ] `README.md` reflects Phase 0 complete status

Once all boxes are checked, invoke the `writing-plans` skill again to draft the Phase 1 (Citation Tracker) plan.
