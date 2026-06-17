# ui-visual-agent

AI-powered visual regression testing pipeline. Two modes:

- **Nightly** — runs defined scenarios, finds UI bugs, opens GitHub PRs with fixes
- **Chaos** — Claude autonomously explores the app, clicking wherever it finds interesting elements, fixes committed directly to `main`. Built for battle-hardening in the days before launch.

Reusable across projects. Point it at any running app, drop in a config file.

---

## How it works

### Nightly mode

```
every night (03:00 UTC)

Playwright                        Fix Agent
opens each scenario URL    →      reads open tickets from Postgres
+ viewport size                   → Claude generates code fix
     │                            → opens PR on bugfix branch
     ▼                            → ticket marked resolved
screenshot.png
     │
     ▼
Claude Vision
checks against 50-issue checklist
     │
  pass ──► nothing logged
  fail ──► bug ticket inserted into Postgres
           → fix agent runs automatically (AUTO_FIX=true)
```

### Chaos mode

```
every 5 min, pre-launch only (enable schedule in chaos.yml)

for each route in config.routes (or discovered from /sitemap.xml):

  step 1..N:
    screenshot
       │
       ├─ Claude Vision checks against 50-issue checklist → ticket if fail
       │
       └─ Claude decides what to click next
             │
          execute action (getByText → click / fill)
             │
          repeat

  fix agent runs after session → commits directly to main
```

---

## Connecting a new project (step-by-step)

### 1. Push this repo to GitHub

```bash
git add .
git commit -m "feat: initial setup"
git push origin main
```

### 2. Create a Postgres database (Neon — free, 2 min)

1. Sign up at [neon.tech](https://neon.tech)
2. "New project" → pick a name → region **EU Central**
3. Dashboard → **Connection string** → copy the string:
   ```
   postgresql://user:pass@ep-xxx.eu-central-1.aws.neon.tech/neondb?sslmode=require
   ```

### 3. Create a GitHub PAT for the fix agent

The fix agent needs write access to the repo it will open PRs in (e.g. `artboxes`).

1. [github.com/settings/tokens](https://github.com/settings/tokens) → **Fine-grained tokens** → "Generate new token"
2. **Resource owner**: your org (e.g. `NullStateLabs`)
3. **Repository access**: "Only select repositories" → select the target repo (`artboxes`)
4. **Permissions**: `Contents` → Read and write, `Pull requests` → Read and write
5. Generate → copy the token (shown only once)

### 4. Add Secrets and Variables to this repo on GitHub

Go to: **github.com/YOUR-ORG/ui-visual-agent → Settings → Secrets and variables → Actions**

**Secrets tab:**

| Name | Value |
|---|---|
| `ANTHROPIC_API_KEY` | Anthropic API key |
| `DATABASE_URL` | Neon connection string from step 2 |
| `UI_FIX_GITHUB_TOKEN` | PAT from step 3 |
| `BASE_URL` | Public URL of the app to test, e.g. `https://demo.boxes.art` |
| `AUTH_STATE` | Base64-encoded auth state (see step 4a — optional, only for auth-protected pages) |

**Variables tab:**

| Name | Value |
|---|---|
| `REPO_OWNER` | e.g. `NullStateLabs` |
| `REPO_NAME` | e.g. `artboxes` |
| `REPO_BRANCH` | `main` |

### 4a. Set up auth for protected pages (optional)

If your app has login-protected pages, the agent needs a saved auth state to access them. Works with any provider that stores session data in cookies or localStorage: Privy, NextAuth, Clerk, Supabase, etc.

**Step 1 — install browsers (once):**

```bash
pnpm exec playwright install chromium
```

**Step 2 — capture auth state interactively:**

```bash
pnpm auth:save -- --url=https://your-app.com
```

A browser window opens. Log in as you normally would, then close the window. The script captures cookies + localStorage and saves them to `auth/state.json`.

**Step 3 — encode and upload to GitHub:**

```bash
base64 -i auth/state.json | pbcopy   # macOS — copies to clipboard
```

Go to **Settings → Secrets → Actions → New repository secret**, name it `AUTH_STATE`, paste and save.

**Step 4 — reference the file in your config:**

```ts
const config: AgentConfig = {
  auth: {
    storageState: "auth/state.json",
  },
  scenarios: [ ... ],
};
```

`AgentConfig.auth` applies to every scenario and every chaos session. Per-scenario overrides are also supported:

```ts
scenarios: [
  { label: "home — public", url: "/", auth: null },   // opt out for public pages
  { label: "profile",       url: "/profile" },         // inherits config.auth
  { label: "admin",         url: "/admin", auth: { storageState: "auth/admin.json" } },
]
```

**How CI injects the secret:**

Both `nightly.yml` and `chaos.yml` decode the secret into `auth/state.json` before tests run:

```yaml
- name: Restore auth state
  env:
    AUTH_STATE: ${{ secrets.AUTH_STATE }}
  run: |
    if [ -n "$AUTH_STATE" ]; then
      mkdir -p auth
      echo "$AUTH_STATE" | base64 -d > auth/state.json
    fi
```

If `AUTH_STATE` is not set the step is a no-op and the agent runs without auth.

**Token expiry:** Most auth providers (Privy, Clerk, etc.) issue tokens that expire in 1–7 days. Re-run `pnpm auth:save` and update the `AUTH_STATE` secret whenever tests start failing on protected pages.

### 5. Create the config file for your project

**Option A — let Claude generate it for you (recommended)**

Open [examples/generate-config-prompt.md](examples/generate-config-prompt.md),
copy the prompt inside, and paste it into Claude **in the context of your target project**.
Claude will explore the codebase and return a complete `ui-agent.config.ts` with
real file paths, viewports, and skip rules already filled in.

**Option B — copy the reference example and edit manually**

```bash
cp examples/artboxes-config.ts ui-agent.config.ts
```

Edit `ui-agent.config.ts` — update the import path and adapt the routes to your app:

```diff
- import type { AgentConfig } from "../src/runner/types.js";
+ import type { AgentConfig } from "./src/runner/types.js";
```

Then commit it:

```bash
git add ui-agent.config.ts
git commit -m "feat: add project config"
git push origin main
```

### 6. Create the database table

Run once — either locally with your `DATABASE_URL` in `.env`, or trigger it manually in CI:

```bash
pnpm migrate
```

### 7. Test the pipeline manually

1. Go to: **github.com/YOUR-ORG/ui-visual-agent → Actions → "Nightly Visual QA"**
2. Click **"Run workflow"** → "Run workflow"
3. Watch the logs live

Expected output if no issues found:
```
Playwright runs 5 scenarios…
Fix Agent starting… (mode: bugfix-branch)
Found 0 open ticket(s)
```

Expected output if issues are found:
```
[HIGH] Bug ticket #1: flex children appear side-by-side on mobile viewport
Fix Agent starting… (mode: bugfix-branch)
Branch 'bugfix' created from main
PR opened: https://github.com/NullStateLabs/artboxes/pull/42
```

### 8. Verify results

- **Screenshots**: Actions → workflow run → Artifacts → `screenshots-*`
- **Bug tickets**: check your Neon DB — `SELECT * FROM ui_bug_tickets;`
- **Fix PRs**: check the target repo (`artboxes`) for a PR from the `bugfix` branch

### 9. Activate chaos mode (pre-launch only)

Uncomment the schedule in [.github/workflows/chaos.yml](.github/workflows/chaos.yml):

```yaml
# Before:
# schedule:
#   - cron: "*/5 * * * *"

# After:
schedule:
  - cron: "*/5 * * * *"
```

Push → chaos mode fires every 5 minutes, commits fixes directly to `main` of the target repo.
Re-comment after launch.

---

## Local setup (without CI)

```bash
pnpm install
pnpm exec playwright install chromium
cp .env.example .env   # fill in BASE_URL, ANTHROPIC_API_KEY, DATABASE_URL
pnpm migrate
```

| Command | What it does |
|---|---|
| `MOCK_LLM=true pnpm test` | Full pipeline dry-run — no API calls, always finds 1 mock issue |
| `pnpm test` | Real nightly run against BASE_URL |
| `pnpm test:ci` | Same + auto-triggers fix agent (opens PR on `bugfix` branch) |
| `pnpm test:chaos` | Chaos exploration run |
| `pnpm test:chaos:ci` | Same + commits fixes to `test-chaos` branch |
| `pnpm fix` | Manually process any open tickets → open PR |
| `pnpm migrate` | Create `ui_bug_tickets` table |
| `pnpm auth:save -- --url=<URL>` | Open browser → log in → save `auth/state.json` |

### Environment variables

| Variable | Description |
|---|---|
| `BASE_URL` | URL of the app to test, e.g. `https://artboxes.io` |
| `ANTHROPIC_API_KEY` | Anthropic API key |
| `DATABASE_URL` | Postgres connection string |
| `UI_FIX_GITHUB_TOKEN` | GitHub PAT with `repo` scope (Contents + Pull requests write) |
| `REPO_OWNER` | Target repo owner, e.g. `NullStateLabs` |
| `REPO_NAME` | Target repo name, e.g. `artboxes` |
| `REPO_BRANCH` | Base branch, default `main` |
| `BUILD_COMMAND` | Shell command to verify the build before pushing (e.g. `pnpm install --frozen-lockfile && pnpm build`). Runs inside a shallow clone of the target repo. Push is aborted if the command fails. Leave unset to skip. |
| `MOCK_LLM` | Set `true` to skip real API calls during pipeline testing |
| `AUTH_STATE` | (CI only) Base64-encoded `auth/state.json`. Decoded into the file by the workflow before tests run. |

---

## Usage

| Command | What it does |
|---|---|
| `pnpm test` | Run nightly scenarios, print findings |
| `pnpm test:ci` | Same + auto-trigger fix agent (opens PRs on `bugfix` branch) |
| `pnpm test:chaos` | Run chaos exploration, print findings |
| `pnpm test:chaos:ci` | Same + auto-trigger fix agent (commits to `test-chaos` branch) |
| `pnpm fix` | Manually process open tickets → open PRs |
| `pnpm migrate` | Create `ui_bug_tickets` table |
| `pnpm auth:save -- --url=<URL>` | Capture auth state interactively → `auth/state.json` |

---

## Config file

`ui-agent.config.ts` in the repo root:

```ts
import type { AgentConfig } from "./src/runner/types.js";

const config: AgentConfig = {
  // ── Auth (optional) ───────────────────────────────────────────────
  // Applied to every scenario and chaos session by default.
  // Generate auth/state.json with: pnpm auth:save -- --url=https://your-app.com
  auth: {
    storageState: "auth/state.json",
  },

  // ── Skip false positives that fire on every page ──────────────────
  // Issue IDs from src/checklists/common-ui-issues.ts
  globalSkipIssueIds: [8, 49],

  // ── Nightly scenarios ─────────────────────────────────────────────
  scenarios: [
    {
      label: "home — mobile",
      url: "/",
      filePath: "app/page.tsx",          // source file the fix agent will edit
      viewport: { width: 375, height: 812 },
      auth: null,                        // public page — skip auth
    },
    {
      label: "home — desktop",
      url: "/",
      filePath: "app/page.tsx",
      viewport: { width: 1280, height: 800 },
      severityThreshold: "medium",
      auth: null,
    },
    {
      label: "profile — mobile",
      url: "/profile",
      filePath: "app/profile/page.tsx",
      viewport: { width: 375, height: 812 },
      // inherits config.auth automatically
    },
    {
      label: "connect wallet modal",
      url: "/",
      filePath: "components/modals/ConnectWalletModal.tsx",
      viewport: { width: 375, height: 812 },
      steps: [
        { action: "click", selector: "button:has-text('Connect')" },
        { action: "wait", ms: 600 },
      ],
      auth: null,
    },
    {
      label: "404 page",
      url: "/this-page-does-not-exist",
      filePath: "app/not-found.tsx",
      viewport: { width: 375, height: 812 },
      auth: null,
    },
  ],

  // ── Chaos mode routes ─────────────────────────────────────────────
  // If omitted, routes are discovered from /sitemap.xml automatically.
  routes: [
    "/",
    "/collections",
    "/mint",
    "/profile",
  ],

  // ── Chaos mode options ────────────────────────────────────────────
  chaosConfig: {
    steps: 12,
    explorationMode: "chaos",          // "chaos" = random clicks, "explore" = strategic
    severityThreshold: "medium",
  },
};

export default config;
```

### Scenario options (nightly)

| Field | Type | Description |
|---|---|---|
| `label` | `string` | Name shown in test output and bug tickets |
| `url` | `string` | Path relative to `BASE_URL` |
| `filePath` | `string` | Source file path in the target repo (e.g. `app/upcoming/page.tsx`). Required for auto-fix — the fix agent edits this file. If omitted, tickets are logged but cannot be auto-fixed. |
| `viewport` | `{ width, height }` | Defaults to `375×812` (mobile) |
| `steps` | `StepAction[]` | Interactions before the screenshot (click, fill, hover, wait, scroll, press) |
| `skipIssueIds` | `number[]` | Checklist issue IDs to ignore for this scenario |
| `severityThreshold` | `"high" \| "medium" \| "low"` | Only report issues at or above this level. Default: `"low"` |
| `auth` | `ScenarioAuth \| null` | Override the global `AgentConfig.auth` for this scenario. `null` disables auth even when a global default is set. |

### Top-level config options

| Field | Type | Description |
|---|---|---|
| `auth` | `ScenarioAuth` | Default auth applied to every scenario and chaos session. Contains `storageState` (path to JSON file) and/or `localStorage` (key/value pairs). |
| `globalSkipIssueIds` | `number[]` | Checklist issue IDs to skip across **all** scenarios. Use for known false positives that apply site-wide. Per-scenario `skipIssueIds` are merged on top. |
| `routes` | `string[]` | Starting routes for chaos exploration. Falls back to `/sitemap.xml` then `["/"]`. |
| `chaosConfig` | `ChaosConfig` | Chaos mode options (see table below). |

### Chaos options

| Field | Type | Description |
|---|---|---|
| `steps` | `number` | Exploration steps per session. Default: `12` |
| `viewport` | `{ width, height }` | Viewport for chaos sessions. Default: `375×812` |
| `severityThreshold` | `"high" \| "medium" \| "low"` | Only report issues at or above this level. Default: `"medium"` |
| `explorationMode` | `"chaos" \| "explore"` | `"chaos"` (default) picks clicks randomly — simulates an unpredictable user. `"explore"` picks strategically and avoids repeating the same element. |

---

## No element IDs required

The chaos runner does not require `data-testid` or IDs on clickable elements. Claude reads the screenshot visually, identifies interesting elements by their visible text, and Playwright locates them via `getByText()`. Any standard button, link, input, tab or nav item is automatically discoverable.

---

## GitHub Actions

### Nightly (`.github/workflows/nightly.yml`)
Runs at 03:00 UTC. Flow per run:

1. Generate all code fixes in memory
2. Clone target repo → apply fixes → run `BUILD_COMMAND`
3. If build passes → stage N commits (one per ticket) → **one push**
4. Open / update PR on the `bugfix` branch

Build fails → push is aborted, nothing reaches GitHub, no Vercel preview.

### Chaos (`.github/workflows/chaos.yml`)
Manual trigger (`workflow_dispatch`) by default. The `schedule` block is commented out — **uncomment it only during the pre-launch battle-hardening window**, then re-comment after launch.

```yaml
# Uncomment to activate chaos cron (pre-launch only):
# schedule:
#   - cron: "*/5 * * * *"
```

Chaos fixes are committed **directly to `main`**, not a branch. Only enable this when you trust the pipeline.

---

## Required GitHub Secrets / Variables

| Secret | Required | Value |
|---|---|---|
| `DATABASE_URL` | yes | Postgres connection string |
| `ANTHROPIC_API_KEY` | yes | Anthropic API key |
| `UI_FIX_GITHUB_TOKEN` | yes | GitHub PAT with `Contents` + `Pull requests` write |
| `BASE_URL` | yes | URL of the deployed app |
| `AUTH_STATE` | optional | Base64-encoded `auth/state.json` (see step 4a) |

| Variable | Required | Value |
|---|---|---|
| `REPO_OWNER` | yes | e.g. `NullStateLabs` |
| `REPO_NAME` | yes | e.g. `artboxes` |
| `REPO_BRANCH` | yes | e.g. `main` |

---

## UI Issues Checklist

Claude Vision checks each screenshot against 50 common UI problems across 10 categories.

| Category | Issues |
|---|---|
| Layout | overflow, unwanted side-by-side on mobile, z-index overlap, fixed-width breakage, misalignment, missing padding, inconsistent spacing |
| Typography | clipped text, overflow, ellipsis, tiny font, tight line-height, no visual hierarchy |
| Contrast | invisible text, invisible icon, low-contrast button label, unreadable placeholder |
| Images | broken image, stretched aspect ratio, unintended clip, placeholder visible |
| Interactive | button label overflow, tap target too small, overlapping controls, icon-only with no label |
| Forms | input too narrow, label misassociation, error message overlap, inconsistent widths |
| Navigation | links cut off, nav overlapping content, awkward breadcrumb wrap |
| Cards | content overflow, inconsistent grid heights, icon/text misalignment |
| Modals | clipped by screen edge, hidden close button, transparent overlay |
| States | raw JSON / stack trace visible, frozen spinner, empty state with no message |

Full list: [src/checklists/common-ui-issues.ts](src/checklists/common-ui-issues.ts)

---

## File structure

```
ui-visual-agent/
├── .github/workflows/
│   ├── nightly.yml               # 03:00 UTC cron → scenario tests → PRs
│   └── chaos.yml                 # every 5 min (pre-launch) → exploration → direct commits
├── src/
│   ├── agent/
│   │   └── fix-agent.ts          # runFixAgent({ mode }) — bugfix-branch PR or direct commit
│   ├── checklists/
│   │   └── common-ui-issues.ts   # 50-item checklist
│   ├── helpers/
│   │   ├── auth.ts               # applyAuth() — injects cookies + localStorage before each scenario
│   │   ├── db-ticket.ts          # Postgres CRUD
│   │   ├── llm-vision.ts         # analyzeScreenshotWithChecklist, suggestNextAction, generateCodeFix
│   │   ├── migrate.ts            # CREATE TABLE
│   │   └── sitemap.ts            # fetch /sitemap.xml → route list
│   └── runner/
│       ├── chaos-runner.ts       # autonomous exploration session
│       ├── scenario-runner.ts    # deterministic scenario execution
│       └── types.ts              # Scenario / AgentConfig / ScenarioAuth / ChaosConfig
├── specs/
│   ├── visual-checklist.spec.ts  # nightly spec
│   └── chaos.spec.ts             # chaos spec
├── scripts/
│   └── save-auth.ts              # interactive script: opens browser, saves auth/state.json
├── examples/
│   ├── artboxes-config.ts
│   └── section-header-mobile.spec.ts
├── auth/                         # gitignored — contains saved auth state
│   └── state.json                # generated by pnpm auth:save
├── screenshots/
│   └── chaos/                    # chaos session screenshots (step-01-home.png …)
├── playwright.config.ts
├── .env.example
└── package.json
```

---

## Database schema

```sql
CREATE TABLE ui_bug_tickets (
  id              SERIAL PRIMARY KEY,
  component       TEXT        NOT NULL,
  file_path       TEXT        NOT NULL,
  assertion       TEXT        NOT NULL,
  reasoning       TEXT        NOT NULL,
  screenshot_path TEXT        NOT NULL,
  status          TEXT        NOT NULL DEFAULT 'open',
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  resolved_at     TIMESTAMPTZ
);
```
