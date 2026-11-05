---
name: nextjs-prisma-crud
description: Build, change, or review CRUD features in Next.js App Router apps that use Prisma, Zod, and Server Actions. Use when adding or editing a Prisma model, Zod schema, Server Action, admin page, form, or table; when a change touches tenant scope, permissions, transactions, balances, inventory, audit logs, or deletes; or when auditing that code for authorization gaps.
---

# Next.js + Prisma CRUD

The repository is the spec. Mirror its conventions; this skill fills the gaps and keeps new
code consistent with what is already there. Where the repo and this skill disagree, the repo
wins, except for the **Server-side rules**, which hold in every project.

Introduce only conventions the repo already has. In particular, add a shared
`ActionResponse<T>` type, `Prisma.*GetPayload` types, soft delete, UUID keys, a new folder
name, or Zod helpers (`pick`, `omit`, `partial`, `merge`, `superRefine`, `transform`) only
when the repo already uses them. Keep the change inside the requested feature.

## 1. Discover

Read `package.json` and the lockfile, `tsconfig.json`, `next.config.*`,
`prisma/schema.prisma` (including the `generator` block), the schema, actions, and `app/`
folders, test and CI config, and any agent instructions already in the repo (`AGENTS.md`,
`CLAUDE.md`, `.github/copilot-instructions.md`, other skills).

**Done when** you can state each of these, or have marked it unknown and asked the user:

- Package manager (from the lockfile) and the versions of Next.js, Prisma, Zod, and React Hook Form.
- Layout: code at the root or under `src/`.
- Database provider and the Prisma Client import path (Prisma 7's `prisma-client` generator
  outputs to a custom path; enums and types come from there, not from `@prisma/client`).
- Auth helper, permission mechanism, and **tenant scope** field (`companyId`, `accountId`,
  `studioId`, or whatever this repo uses; some repos have none).
- Audit helper, delete strategy, and the `package.json` scripts for typecheck, lint, and tests.

## 2. Read the analog

Find the closest existing entity, the **analog**, and read its full stack: Prisma model and
related models, Zod schema, actions, page, manager, form, and table.

**Done when** you can name the analog's file names, response shape, create/update style
(`upsert` or explicit branching), audit call, and revalidation targets. Copy those patterns.

## 3. Build

Each layer owns one job:

| Layer | Owns |
| --- | --- |
| Prisma schema | Models, relations, enums, keys, indexes, constraints, `onDelete`, tenant fields |
| Zod schema | Input validation, form contracts, nested payloads, cross-field rules |
| Server Action | Auth, permissions, validation, scoping, transactions, audit, revalidation |
| Server page | Awaits `params` / `searchParams`, checks auth, loads data and catalogs, passes serializable props |
| Manager (client) | Selected entity, dialog state, search, pagination, delete confirmation, toasts |
| Form (client) | React Hook Form + Zod resolver, default values, payload, submit, loading state |
| Table / list | Renders rows and emits callbacks (`onEdit`, `onDelete`); persistence stays in actions |

Work through the layers in order:

1. Decide the entity's tenant scope: the tenant field, a scoped parent record, or none
   (then name the risk in the report).
2. Edit the Prisma schema only when the task asks for it. Write the change and give the user
   the migration command to run.
3. Write the Zod schema in the repo's schema folder.
4. Write the actions in the analog's file layout, following the lifecycle in
   [references/actions.md](references/actions.md).
5. Build the page, manager, form, and table from the analog's components.
6. Revalidate only the paths or tags that show the changed data.

## Server-side rules

- Every Server Action and Route Handler authenticates, checks permission, and parses input
  with the Zod schema (`safeParse`) on the server. Hidden buttons, protected layouts, and
  client-side role checks are UX, never authorization.
- Every query is **scoped**: reads and writes filter by the tenant field, and every related
  ID in the payload (`categoryId`, `accountId`, ...) is verified against the same scope
  before it is used.
- Mutations run only after every check has passed.
- Related writes, balances, inventory movements, and audit records share one transaction.
  Failures inside it **throw** so it rolls back, and the action returns a failure response.
  Every helper that writes receives `tx`.
- The module's response shape, delete strategy, and balance/payment/inventory helpers stay
  as they are; new code calls them.
- Secrets come from environment variables and never reach logs or responses. Responses
  `select` fields explicitly when the model holds hashes, tokens, or internal data.
- `after()` holds only effects the business result can survive losing (email, WhatsApp,
  notifications). It is best-effort, not a queue.
- New dependencies wait for the user's approval.

**Hard stop:** migrations (`migrate dev`, `db push`, `migrate reset`), deploys, production
data changes, destructive scripts, and dependency changes run only when the user explicitly
asks. Otherwise, write the command in the report for the user to run.

## 4. Verify and report

Run the repo's own scripts: typecheck, lint, focused tests for the module, the full suite,
and the build when it is safe. Then check the diff against every Server-side rule.

Report:

- Files changed.
- Each check as **passed**, **failed**, or **not run** (with the reason). A check that did not
  run is never reported as passing.
- Risks found: unscoped models, existing actions that break a Server-side rule (report them;
  fix them only if the user asks), commands left for the user.

**Done when** every check and every rule has a status in the report.

## References

Load a reference only when its condition matches the task:

- [references/actions.md](references/actions.md): writing or editing any Server Action.
  Lifecycle skeleton, create/update, pagination, response shapes, transactions,
  revalidation, `after()`, deletes.
- [references/data-and-validation.md](references/data-and-validation.md): editing a Prisma
  model or Zod schema, schema/migration work, payments, balances, inventory, or audit.
- [references/ui.md](references/ui.md): building a page, manager, form, table, URL filters,
  generated UI (v0), or landing/SEO pages.
- [references/integrations.md](references/integrations.md): email, WhatsApp, file uploads,
  external APIs, cron jobs, or Docker.
- [references/security.md](references/security.md): reviewing existing code, or touching
  auth, permissions, or tenant scope.
- [references/testing.md](references/testing.md): the repo has a test runner, or the task
  adds tests.
