# House conventions

Defaults for a repo that has no convention yet. An existing repo's own layout always wins.

## Layout

```text
prisma/schema.prisma
lib/prisma.ts                      single Prisma Client instance (globalThis cache in dev)
schema/<entity>.schema.ts          Zod schemas  + schema/index.ts barrel
interfaces/pagination.interface.ts shared pagination and props types
interfaces/<entity>.interface.ts   I<Model>Extended types + interfaces/index.ts barrel
actions/<entity>/                  one folder per model, singular, kebab-case
  create-update-<entity>.ts
  delete-<entity>.ts
  get-paginated-<entity>.ts
  get-<entity>-by-id.ts            only when a detail page needs it
  index.ts                         barrel: export * from each file
app/(app)/<ruta>/page.tsx          route segment in the app's language (/clientes)
app/(app)/<ruta>/ui/               <Entity>Manager.tsx, <Entity>Form.tsx, <Entity>Table.tsx
lib/labels.ts                      enum → display label maps, shared by forms and tables
```

All of it may sit under `src/`. Code identifiers are in English; user-facing text, messages,
and route segments are in the app's language.

## Shared interfaces

One file serves every module. A new filter is one more optional field here.

```typescript
// interfaces/pagination.interface.ts
export interface PaginationProps {      // Server Action params
    page: number;
    take?: number;
    search?: string;
    isActive?: boolean;
}

export interface ISearchParams {        // URL params, always strings
    page?: string;
    search?: string;
}

export interface PaginatedPageProps {
    searchParams: Promise<ISearchParams>;
}

export interface PaginatedManagerProps<T> {
    data: T[];
    totalPages?: number;
    currentPage?: number;
    currentSearch?: string;
}

export interface PaginatedTableProps<T> {
    data: T[];
    currentPage?: number;
    totalPages?: number;
    onPageChange?: (page: number) => void;
    onCreate?: () => void;
    onEdit?: (item: T) => void;
    onDelete?: (item: T) => void;
}
```

## Extended interfaces

A read action that uses `include` returns more than the base model. Type it by extending
the generated model, never by redeclaring its fields:

```typescript
// interfaces/project.interface.ts
import type { Client, Payment, Project } from "@/app/generated/prisma/client"; // or "@prisma/client" on Prisma 6

export interface IProjectExtended extends Project {
    client: Client;
    payments: Payment[];
}
```

Summary and safe types are derived with `Pick` and `Omit`, once, in the model's interface file:

```typescript
export type IUserSafe = Omit<User, "passwordHash">;      // what may reach the client
export type IUserSummary = Pick<User, "id" | "fullName">; // for owners, responsibles, selects
```

The form receives the base model (`Project | null`); the table and manager receive
`IProjectExtended[]`.

## Models

- `id Int @id @default(autoincrement())`, `createdAt DateTime @default(now())`,
  `updatedAt DateTime @updatedAt`.
- Every relation states `onDelete` (`Cascade` or `SetNull`).
- Models that other records reference and whose history must survive get
  `isActive Boolean @default(true)`; "deleting" them sets it to `false`.
- `@@index` on foreign keys and on fields used in filters (`status`, `role`, `date`, the
  tenant field).

## Server Action responses

```typescript
{ ok: true, message }                              // mutations
{ ok: true, data, page, totalPages, totalCount }  // paginated reads
{ ok: true, data }                                 // single reads
{ ok: false, message }                             // every failure
```

Messages are complete sentences the user can act on ("Ya existe un cliente con ese correo").
