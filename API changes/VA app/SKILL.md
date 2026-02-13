---
name: VA app
description: Analyze code changes between git branches (main vs feature) in the VHA Procurement Workflow System. Automatically detects API changes, database schema modifications, UI updates, and generates QA-ready changelogs with test case suggestions. Use when comparing branches, preparing QA handoffs, or documenting feature changes.
---
s
# VA API Change Analyzer

A skill for analyzing code differences between branches in the **VHA Procurement Workflow System** and generating structured, QA-ready documentation.

---

## Project Context

This skill operates on the **VHA (Veterans Health Administration) Procurement Workflow System** — a multi-tier procurement package management application for the Department of Veterans Affairs.

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js (App Router), React, TypeScript, Tailwind CSS, shadcn/ui |
| Backend | Node.js with Next.js API Routes (RESTful) |
| Database | PostgreSQL on Supabase, Prisma ORM |
| Auth | Supabase Auth (session-based, JWT, @va.gov email restriction) |
| State | Zustand (global), React Context (auth), React Query (server) |
| Forms | react-hook-form + Zod validation |

### Key Architecture Patterns

- **API Routes** live in `app/api/` using Next.js App Router route handlers
- **Database access** goes through Prisma ORM (`lib/prisma.ts`)
- **Auth** is enforced via middleware (`middleware.ts`) + `getCurrentUser()` helper
- **Form data** is auto-saved via `POST /api/packages/[id]/data` with 2s debounce
- **Workflow engine** drives the multi-tier form system (17 planned tiers, 2 active)
- **Conditional logic** controlled by `workflow_rules`, `rule_conditions`, `rule_actions` tables

### Critical File Locations

```
app/api/                    → All API route handlers
prisma/schema.prisma        → Database schema (41 tables)
prisma/migrations/          → Migration history
lib/api-client.ts           → Frontend API client
lib/auth-helpers.ts         → Auth utilities
lib/palt-calculator.ts      → PALT business logic
middleware.ts               → Route protection & session validation
components/tiers/           → Tier-specific form components
stores/                     → Zustand state stores
types/                      → TypeScript type definitions
hooks/                      → Custom React hooks
```

---

## When To Use This Skill

Use this skill when:
- You've created a **feature branch** from `main` and need to document what changed
- You need to generate a **QA handoff document** summarizing changes
- You want to **compare two branches** and understand the diff
- You need **test case suggestions** based on code changes
- You want a **structured changelog** for sprint reviews or PRs

Do NOT use this skill for:
- General code review or refactoring suggestions
- Performance optimization analysis
- Security audits (use dedicated security tools)

---

## How To Invoke

### Option 1: Provide Git Diff

Paste the output of `git diff main..feature-branch` and ask Claude to analyze it.

### Option 2: Provide File Lists

Tell Claude which files were changed, added, or deleted. Claude will ask for file contents as needed.

### Option 3: Provide Both Branch Versions

Upload or paste the before (main) and after (feature branch) versions of changed files.

### Option 4: Claude Code (Recommended)

If using Claude Code inside the repo, run:

```bash
# Generate the diff for Claude to analyze
git diff main..$(git branch --show-current) --stat
git diff main..$(git branch --show-current)
```

Or simply tell Claude Code:
> "Analyze the changes between main and my current branch using the va-api-change-analyzer skill"

Claude Code can directly read the repo, run git commands, and generate the full report.

---

## Analysis Workflow

When analyzing changes, follow this exact sequence:

### Step 1: Gather Change Information

```bash
# 1. Get list of changed files
git diff main..HEAD --name-status

# 2. Get full diff
git diff main..HEAD

# 3. Check for new migration files
ls -la prisma/migrations/ | tail -20

# 4. Check for new/modified API routes
git diff main..HEAD --name-status -- 'app/api/'

# 5. Check for schema changes
git diff main..HEAD -- prisma/schema.prisma

# 6. Check for type definition changes
git diff main..HEAD -- 'types/'

# 7. Check for new dependencies
git diff main..HEAD -- package.json
```

### Step 2: Classify Every Change

Categorize each changed file into one of these buckets:

| Category | Priority | Files |
|----------|----------|-------|
| **API Changes** | 🔴 Critical | `app/api/**/*.ts` |
| **Database Schema** | 🔴 Critical | `prisma/schema.prisma`, `prisma/migrations/` |
| **Auth/Security** | 🔴 Critical | `middleware.ts`, `lib/auth-helpers.ts`, `contexts/AuthContext.tsx` |
| **Business Logic** | 🟠 High | `lib/palt-calculator.ts`, `lib/*.ts`, `stores/*.ts` |
| **Type Definitions** | 🟠 High | `types/**/*.ts` |
| **UI Components** | 🟡 Medium | `components/**/*.tsx` |
| **Form/Tier Changes** | 🟡 Medium | `components/tiers/**/*.tsx` |
| **Styling** | 🟢 Low | `*.css`, `tailwind.config.*` |
| **Config** | 🟢 Low | `next.config.*`, `tsconfig.json`, `.env*` |
| **Docs/Tests** | 🟢 Low | `docs/`, `*.md`, `*.test.*` |

### Step 3: Deep-Dive API Changes

For every modified file in `app/api/`, document:

1. **Endpoint**: Method + Path (e.g., `POST /api/packages/[id]/data`)
2. **Change Type**: New | Modified | Deleted | Breaking
3. **Request Changes**: New/removed/modified fields in request body or params
4. **Response Changes**: New/removed/modified fields in response body
5. **Auth Changes**: Any changes to authentication or authorization checks
6. **Business Logic Changes**: Validation rules, calculations, conditional logic
7. **Error Handling**: New error codes or changed error responses
8. **Breaking Change?**: Yes/No — would existing clients break?

### Step 4: Deep-Dive Database Changes

For any Prisma schema changes, document:

1. **New Tables**: Table name, purpose, key columns
2. **Modified Tables**: Column additions/removals/type changes
3. **New Relations**: Foreign keys, join tables
4. **Migrations**: Migration name, what it does, is it reversible?
5. **Data Impact**: Does existing data need transformation?
6. **Index Changes**: New/removed indexes

### Step 5: Generate QA Test Cases

For each API change, generate test cases following this template:

```
TEST CASE: [TC-XXX] [Brief Description]
Priority: Critical | High | Medium | Low
Preconditions: [Setup required]
Steps:
  1. [Action]
  2. [Action]
Expected Result: [What should happen]
Edge Cases:
  - [Edge case 1]
  - [Edge case 2]
Related Endpoint: [METHOD /path]
```

#### VA-Specific Test Considerations

Always include these VA-specific test scenarios when relevant:

- **Email validation**: Only `@va.gov` emails allowed
- **Session timeout**: 28-min warning, 30-min auto-logout
- **Package ownership**: Users can only access their own packages
- **PALT validation**: Need dates must meet minimum lead times
- **FITARA gate**: IT acquisitions require FITARA approval
- **Form type logic**: MICRO (<$25K), SHORT ($25K-$7M), LONG (≥$7M)
- **Conditional fields**: J&A Type (if Sole Source=Yes), FITARA Form ID (if FITARA=Yes)
- **Auto-save**: 2-second debounce, data persistence across sessions
- **Package status flow**: draft → in_progress → awaiting_signature → complete → archived
- **Failed login lockout**: Max 5 attempts

### Step 6: Generate Output Documents

Produce **two documents**:

---

## Output Document 1: QA Changelog (CHANGELOG-QA.md)

```markdown
# QA Changelog: [Feature Branch Name]
**Date**: [Date]
**Author**: [Developer Name]
**Branch**: `feature/xxx` → `main`
**Sprint/Ticket**: [Reference]

---

## Summary
[1-2 sentence overview of what this feature branch does]

## Risk Assessment
- **Overall Risk**: 🔴 High | 🟠 Medium | 🟢 Low
- **Breaking Changes**: Yes/No
- **DB Migration Required**: Yes/No
- **Env Variable Changes**: Yes/No

---

## API Changes

### New Endpoints
| Method | Path | Description | Auth Required |
|--------|------|-------------|---------------|
| POST | /api/xyz | Does something | Yes |

### Modified Endpoints
| Method | Path | What Changed | Breaking? |
|--------|------|--------------|-----------|
| PATCH | /api/packages/[id] | Added `newField` to request body | No |

### Removed Endpoints
| Method | Path | Reason |
|--------|------|--------|
| — | — | — |

### Detailed API Changes

#### `POST /api/endpoint` (New)
**Purpose**: [What it does]

**Request Body**:
```json
{
  "field1": "string (required)",
  "field2": "number (optional)"
}
```

**Response (200)**:
```json
{
  "id": "string",
  "status": "created"
}
```

**Error Responses**:
- `400`: Invalid input
- `401`: Unauthorized
- `404`: Resource not found

---

## Database Changes

### New Tables
| Table | Purpose | Key Columns |
|-------|---------|-------------|
| — | — | — |

### Modified Tables
| Table | Change | Details |
|-------|--------|---------|
| — | — | — |

### Migration
- **Migration Name**: `YYYYMMDDHHMMSS_description`
- **Reversible**: Yes/No
- **Data Impact**: [None / Requires backfill / etc.]

---

## Business Logic Changes
- [Description of logic change 1]
- [Description of logic change 2]

## UI Changes
- [Component/page change 1]
- [Component/page change 2]

## Configuration Changes
- [New env vars, config changes, dependency updates]

---

## Test Cases

### Critical (Must Pass Before Merge)
| ID | Description | Endpoint | Steps | Expected Result |
|----|-------------|----------|-------|-----------------|
| TC-001 | [Name] | POST /api/xyz | [Steps] | [Result] |

### High Priority
| ID | Description | Endpoint | Steps | Expected Result |
|----|-------------|----------|-------|-----------------|
| TC-010 | [Name] | PATCH /api/xyz | [Steps] | [Result] |

### Regression Checks
| Area | What to Verify |
|------|----------------|
| Auth | Login, session timeout, route protection still work |
| Packages | CRUD operations unaffected |
| Auto-save | 2s debounce still working |
| Tier navigation | Progress sidebar still updates |

---

## Deployment Notes
- [ ] Run database migration: `npx prisma migrate deploy`
- [ ] Update environment variables: [list any new ones]
- [ ] Clear Prisma cache: `npx prisma generate`
- [ ] Verify Supabase RLS policies (if applicable)
- [ ] Smoke test critical flows post-deploy
```

---

## Output Document 2: Developer README (README-CHANGES.md)

```markdown
# Feature: [Feature Name]
**Branch**: `feature/xxx`
**Based on**: `main` @ commit [hash]

## What This Feature Does
[2-3 paragraph description of the feature, why it was built, and how it works]

## Files Changed

### Added
- `path/to/new/file.ts` — [Brief description]

### Modified
- `path/to/modified/file.ts` — [What changed and why]

### Deleted
- `path/to/deleted/file.ts` — [Why it was removed]

## How to Test Locally

1. Pull the branch: `git checkout feature/xxx`
2. Install deps: `npm install`
3. Run migrations: `npx prisma migrate dev`
4. Set env vars: [list any new ones]
5. Start dev server: `npm run dev`
6. Test flow: [Step-by-step of the main happy path]

## API Documentation

[Detailed API docs for any new/changed endpoints with curl examples]

### Example Requests

```bash
# Create new [resource]
curl -X POST http://localhost:3000/api/endpoint \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{"field": "value"}'
```

## Known Limitations
- [Limitation 1]
- [Limitation 2]

## Dependencies Added/Updated
| Package | Version | Purpose |
|---------|---------|---------|
| — | — | — |
```

---

## Existing API Reference (Baseline)

Use this as the baseline when comparing changes. These are the **current production endpoints on main**:

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/packages` | List user's packages |
| POST | `/api/packages` | Create new package |
| GET | `/api/packages/[id]` | Get package details |
| PATCH | `/api/packages/[id]` | Update package metadata |
| DELETE | `/api/packages/[id]` | Delete package |
| GET | `/api/packages/[id]/data` | Get package form data |
| POST | `/api/packages/[id]/data` | Save/update form data (auto-save) |
| GET | `/api/workflows/[workflowId]/steps` | Get workflow steps |
| GET | `/api/workflows/[workflowId]/steps/[stepId]/fields` | Get step fields + conditional rules |

### Existing Database Schema (41 tables)

**9 categories**: Workflow Definition (10), Data Points (2), User & Org (4), Package Data (9), Output Documents (3), COR & Signatures (2), AI Assistance (2), Reference/Lookup (5), System/Audit (3)

Key tables to watch for changes:
- `packages` — Main package records
- `package_data` — Form field values (typed columns)
- `workflow_steps` — Step definitions
- `step_fields` — Form field definitions
- `workflow_rules` / `rule_conditions` / `rule_actions` — Conditional logic
- `data_points` — Master data point registry

---

## Quality Checklist

Before finalizing the QA changelog, verify:

- [ ] Every API change is documented with request/response shapes
- [ ] Breaking changes are clearly flagged with migration path
- [ ] Database changes include migration steps
- [ ] Auth/security changes are highlighted as critical
- [ ] Test cases cover happy path + error cases + edge cases
- [ ] VA-specific business rules are validated (PALT, FITARA, form types)
- [ ] Regression test areas are identified
- [ ] Deployment steps are complete and ordered
- [ ] New environment variables are documented
- [ ] Package.json changes (new deps) are noted

---

## Example Usage

### With Claude Code (in repo)

```
> Analyze changes between main and my current branch. Generate QA changelog and README using the va-api-change-analyzer skill.
```

### With Claude Chat (paste diff)

```
Here's my git diff from main to feature/add-tier2:

[paste diff]

Please analyze these changes using the va-api-change-analyzer skill and generate:
1. QA Changelog (CHANGELOG-QA.md)
2. Developer README (README-CHANGES.md)
```

### Quick Mode (just list changes)

```
> Just give me a quick summary of API changes between main and my branch — no test cases needed.
```

---

## Tips for Best Results

1. **Provide the full diff** — partial diffs lead to incomplete analysis
2. **Include schema.prisma changes** — database changes are often the most critical
3. **Mention the ticket/story** — helps Claude add context to the changelog
4. **Specify your QA team's preferences** — some teams want curl examples, others want Postman collections
5. **Run from Claude Code when possible** — it can read the full repo and produce more accurate results
6. **Review and edit** — Claude's analysis is a starting point; always review before sending to QA


