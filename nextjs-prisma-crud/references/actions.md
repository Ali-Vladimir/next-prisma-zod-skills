# Server Actions

## File layout

Mirror the analog's folder (defaults in `conventions.md`): `create-update-<entity>.ts`,
`delete-<entity>.ts`, `get-paginated-<entity>.ts`, `get-<entity>-by-id.ts` when a detail page
needs it, and the `index.ts` barrel. Domain actions (payments, sales, enrollments) sit next to
these.

Before adding a read action, check whether an existing one already answers it. Catalogs for
selects reuse the entity's paginated action:

```typescript
// shortcut: first 100 active categories, switch to a dedicated get-* if a tenant exceeds it.
getPaginatedCategories({ page: 1, take: 100, isActive: true })
```

## Lifecycle

```text
'use server' → auth → permission → safeParse → scope checks (record + related IDs)
  → transaction (mutation + audit) → revalidate → after() effects → response
```

A skeleton in the house style. Names, helpers, and messages come from the analog; the order
and the checks are the part to keep.

```typescript
"use server";

import { revalidatePath } from "next/cache";
import { Prisma } from "@/app/generated/prisma/client";  // "@prisma/client" on Prisma 6
import { auth } from "@/auth";
import prisma from "@/lib/prisma";                         // the repo's client instance
import { productSchema } from "@/schema";
import { createAuditLogTx } from "@/actions/audit-log";

export async function createUpdateProduct(data: unknown) {
    const session = await auth();
    if (!session?.user) return { ok: false, message: "Tu sesión expiró. Inicia sesión de nuevo." };

    const parsed = productSchema.safeParse(data);
    if (!parsed.success) return { ok: false, message: "Los datos del producto son inválidos." };

    const { id, ...rest } = parsed.data;
    const companyId = Number(session.user.companyId);

    try {
        await prisma.$transaction(async (tx) => {
            // Scoped lookups: the existing record and every related ID belong to this tenant.
            const before = id ? await tx.product.findFirst({ where: { id, companyId } }) : null;
            if (id && !before) throw new Error("NOT_IN_SCOPE");

            const category = await tx.category.findFirst({ where: { id: rest.categoryId, companyId } });
            if (!category) throw new Error("NOT_IN_SCOPE");

            const saved = await tx.product.upsert({
                where: { id: id ?? 0 },
                update: rest,
                create: { ...rest, companyId },
            });

            await createAuditLogTx(tx, { userId: Number(session.user.id), entity: "Product", before, after: saved });
        });
    } catch (error) {
        if (error instanceof Prisma.PrismaClientKnownRequestError && error.code === "P2002") {
            return { ok: false, message: "Ya existe un producto con ese código." };
        }
        console.error(error);
        return { ok: false, message: "No se pudo guardar el producto. Intenta de nuevo." };
    }

    revalidatePath("/productos");
    return { ok: true, message: `Producto ${id ? "actualizado" : "creado"} correctamente.` };
}
```

The `before` lookup does double duty: it is the audit's old value and the ownership check.
Without the scope, `upsert({ where: { id } })` lets a user of one tenant overwrite another
tenant's row.

Every value passed to Prisma comes from `parsed.data`. Fields the server owns (`companyId`,
`createdById`, `reportedById`) come from the session, never from the payload.

## Error messages

Translate known Prisma errors into a sentence that tells the user what to do:

| Code | Meaning | Typical message |
| --- | --- | --- |
| `P2002` | Unique constraint | "Ya existe un registro con ese código." |
| `P2025` | Record not found (or filtered out by a scoped `where`) | "El registro no existe o no tienes acceso." |
| `P2003` | Foreign key blocks the delete | "No se puede eliminar porque tiene registros relacionados; desactívalo." |

Unknown errors are logged with `console.error` and get a generic retry message.

## Pagination

```typescript
export async function getPaginatedProducts({ page, take = 10, search = "", isActive }: PaginationProps) {
    // auth + tenant from the session, as above
    const where = {
        companyId,
        ...(isActive !== undefined && { isActive }),
        ...(search && { name: { contains: search, mode: "insensitive" as const } }),
    };

    const [data, totalCount] = await Promise.all([
        prisma.product.findMany({ where, skip: (page - 1) * take, take, include: { category: true }, orderBy: { createdAt: "desc" } }),
        prisma.product.count({ where }),
    ]);

    return { ok: true, data, page, totalPages: Math.ceil(totalCount / take), totalCount };
}
```

- The tenant scope is part of `where`, shared by both queries.
- `data` is returned as Prisma gives it and typed by `I<Model>Extended` at the consumer.
- `mode: "insensitive"` exists only on PostgreSQL and MongoDB; on MySQL/MariaDB the
  collation already ignores case.

## Transactions

Use one transaction when a change touches several related records, balances, inventory and
its movements, a sale and its items, relations that must stay in sync, or an audit record that
must be atomic with the change.

- Reads that the checks depend on happen inside the transaction.
- **Returning a value from the callback commits.** A callback that returns
  `{ ok: false }` after some writes leaves those writes in the database. To roll back,
  throw; then catch outside and return the failure response.
- Every helper that writes receives `tx`. Read the helper's signature before assuming it is
  transaction-aware.

## Revalidation

Revalidate only what shows the changed data, with the API the repo already uses
(`revalidatePath`, `revalidateTag`, or `updateTag` on Next.js 16).

## `after()`

`after()` (from `next/server`) runs after the response is sent: email, WhatsApp, low-stock
alerts. If the process stops, the work is lost, and it can run zero or more times. Every step
the business result depends on happens before the response.

## Deletes

Follow the entity's strategy, after reading the relation's `onDelete` rules:

- `isActive` models: the delete action sets `isActive: false`, and lists filter
  `isActive: true` by default.
- Hard delete: remove the row. Business conditions go in the scoped `where`, so a row that
  does not qualify throws `P2025`:

```typescript
await tx.finding.delete({ where: { id, companyId, status: "OPEN" } });
```
