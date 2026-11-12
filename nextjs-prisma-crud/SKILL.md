---
name: nextjs-prisma-crud
description: Build, change, or review CRUD features in Next.js App Router apps that use Prisma, Zod, and Server Actions. Use when adding or editing a Prisma model, Zod schema, Server Action, admin page, form, or table; when a change touches tenant scope, permissions, transactions, balances, inventory, audit logs, or deletes; or when auditing that code for authorization gaps.
---

# Next.js + Prisma CRUD

The repository is the spec. Mirror its conventions; this skill fills the gaps and keeps new
code consistent with what is already there. Where the repo and this skill disagree, the repo
wins, except for the **Server-side rules**, which hold in every project. Where the repo has no
convention yet (a new project, a missing folder), use the house defaults in
[references/conventions.md](references/conventions.md).

## Lean code

Write the least new code that fully solves the task. Reuse before you write, in this order:
an existing action, interface, schema, component, or helper; then an installed dependency;
then the platform (Prisma, Next.js, Zod); only then new code.

- Data flows in the shape Prisma returns it. A query with `include` is typed by an
  `I<Model>Extended` interface, and the result goes straight to the component: no DTOs, no
  `.map()` reshaping, no parallel types.
- One Zod schema per model covers create, update, and the form. Child schemas saved with
  their parent live in the parent's file.
- Shared shapes grow in place: a new filter becomes one more optional field on
  `PaginationProps`, not a new props type.
- Queries use `include`. `select` appears only to drop sensitive fields (password hashes,
  tokens) or to match a summary type such as `IUserSummary`.
- Abstractions, wrappers, options, and "for later" code exist only when the task asks for them.
- A deliberate shortcut with a known limit gets a one-line comment:
  `// shortcut: <the limit>, <when to change it>`.

Lean never cuts: server-side validation, authorization, tenant scope, transaction safety, or
error handling that prevents data loss.

## 1. Discover

Read `package.json` and the lockfile, `tsconfig.json`, `next.config.*`,
`prisma/schema.prisma` (including the `generator` block), the `schema/`, `interfaces/`,
`actions/`, and `app/` folders, test and CI config, and any agent instructions already in the
repo (`AGENTS.md`, `CLAUDE.md`, `.github/copilot-instructions.md`, other skills).

**Done when** you can state each of these, or have marked it unknown and asked the user:

- Package manager (from the lockfile) and the versions of Next.js, Prisma, Zod, and React Hook Form.
- Layout: code at the root or under `src/`.
- Database provider and the Prisma Client import path (Prisma 7's `prisma-client` generator
  outputs to a custom path; models, enums, and `Prisma` come from there, not `@prisma/client`).
- Auth helper, permission mechanism, and **tenant scope** field (`clinicId`, `accountId`,
  `studioId`, or whatever this repo uses; some repos have none).
- The shared interfaces (`PaginationProps`, `ISearchParams`, page/manager/table props) and
  the existing `I<Model>Extended` interfaces.
- Audit helper, delete strategy (`isActive` or hard delete), and the `package.json` scripts
  for typecheck, lint, and tests.

## 2. Read the analog

Find the closest existing entity, the **analog**, and read its full stack: Prisma model and
related models, Zod schema, interfaces, actions, page, manager, form, and table.

**Done when** you can name the analog's file names, response shape, create/update style,
audit call, error messages, and revalidation targets. Copy those patterns.

## 3. Build

Each layer owns one job:

| Layer | Owns |
| --- | --- |
| Prisma schema | Models, relations, enums, keys, indexes, `onDelete`, tenant fields, `isActive` |
| Zod schema | Input validation and the form contract, one schema per model |
| Interfaces | Shared pagination/props types and `I<Model>Extended` for queries with `include` |
| Server Action | Auth, permissions, validation, scoping, transactions, audit, revalidation |
| Server page | Awaits `searchParams` / `params`, loads data and catalogs with `Promise.all`, passes props |
| Manager (client) | Selected entity, dialog state, search, pagination, delete confirmation, toasts |
| Form (client) | React Hook Form + Zod resolver, default values, submit, loading state |
| Table / list | Renders rows and emits callbacks (`onEdit`, `onDelete`); persistence stays in actions |

Work through the layers in order:

1. Decide the entity's tenant scope: the tenant field, a scoped parent record, or none
   (then name the risk in the report).
2. Edit the Prisma schema only when the task asks for it. Write the change and give the user
   the migration command to run.
3. Write the Zod schema in the repo's schema folder.
4. Add an `I<Model>Extended` interface if the read actions `include` relations, and any new
   filter as an optional field on `PaginationProps`.
5. Write the actions in the analog's file layout, following the lifecycle in
   [references/actions.md](references/actions.md).
6. Build the page, manager, form, and table from the analog's components
   ([references/ui.md](references/ui.md)).
7. Revalidate only the paths or tags that show the changed data.

## Server-side rules

- Every Server Action and Route Handler authenticates, checks permission, and parses input
  with the Zod schema (`safeParse`) on the server. Hidden buttons, protected layouts, and
  client-side role checks are UX, never authorization.
- Every query is **scoped**: reads and writes filter by the tenant field, and every related
  ID in the payload (`categoryId`, `patientId`, ...) is verified against the same scope
  before it is used. An update loads the existing record with the scope first.
- Mutations run only after every check has passed.
- Related writes, balances, inventory movements, and audit records share one transaction.
  Failures inside it **throw** so it rolls back, and the action returns a failure response.
  Every helper that writes receives `tx`.
- The module's response shape, delete strategy, and balance/payment/inventory helpers stay
  as they are; new code calls them.
- Secrets come from environment variables and never reach logs or responses. Password
  hashes and tokens never reach the client.
- `after()` holds only effects the business result can survive losing (email, WhatsApp,
  notifications). It is best-effort, not a queue.
- New dependencies wait for the user's approval.

**Hard stop:** migrations (`migrate dev`, `db push`, `migrate reset`), deploys, production
data changes, destructive scripts, and dependency changes run only when the user explicitly
asks. Otherwise, write the command in the report for the user to run.

## 4. Verify and report

Run the repo's own scripts: typecheck (at minimum `tsc --noEmit`), lint, focused tests for
the module, the full suite, and the build when it is safe. Then check the diff against every
Server-side rule and against **Lean code**.

Report:

- Files changed.
- Each check as **passed**, **failed**, or **not run** (with the reason). A check that did not
  run is never reported as passing.
- Risks found: unscoped models, existing actions that break a Server-side rule (report them;
  fix them only if the user asks), shortcuts taken, commands left for the user.

**Done when** every check and every rule has a status in the report.

## References

Load a reference only when its condition matches the task:

- [references/conventions.md](references/conventions.md): the repo lacks a convention you
  need (new project, missing `interfaces/` or `schema/` folder, no analog).
- [references/actions.md](references/actions.md): writing or editing any Server Action.
  Lifecycle skeleton, create/update, pagination, error messages, transactions,
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
