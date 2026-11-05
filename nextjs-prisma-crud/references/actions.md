# Server Actions

## File layout

Mirror the analog's folder. The common layout per entity:

```text
actions/<entity>/
  create-update-<entity>.ts
  delete-<entity>.ts
  get-<entity>-by-id.ts
  get-paginated-<entity>.ts
  index.ts                  (only if the folder already uses a barrel file)
```

Domain actions (payments, sales, enrollments) sit next to these.

## Lifecycle

The order inside a mutating action:

```text
'use server' → auth → permission → safeParse → scope checks (record + related IDs)
  → transaction (mutations + audit) → revalidate → after() effects → response
```

A skeleton with every step. Names, helpers, messages, and the response shape come from the
analog; the order and the checks are the part to keep.

```typescript
"use server";

import { revalidatePath } from "next/cache";
import { auth } from "@/auth";                      // the repo's session helper
import { prisma } from "@/lib/prisma";              // the repo's client instance
import { productSchema } from "@/schema/product.schema";

export async function createUpdateProduct(input: unknown) {
    const session = await auth();
    if (!session?.user) return { ok: false, message: "No autorizado" };
    if (!can(session.user, "products:write")) return { ok: false, message: "Sin permiso" };

    const parsed = productSchema.safeParse(input);
    if (!parsed.success) return { ok: false, message: "Datos inválidos" };

    const { id, categoryId, ...data } = parsed.data;
    const companyId = session.user.companyId;

    try {
        await prisma.$transaction(async (tx) => {
            // Related ID: must belong to the same tenant.
            const category = await tx.category.findFirst({
                where: { id: categoryId, companyId },
                select: { id: true },
            });
            if (!category) throw new Error("CATEGORY_NOT_IN_SCOPE");

            const product = id
                ? await tx.product.update({
                      where: { id, companyId },     // throws P2025 if out of scope
                      data: { ...data, categoryId },
                  })
                : await tx.product.create({
                      data: { ...data, categoryId, companyId },
                  });

            await createAuditLogTx(tx, { entity: "product", entityId: product.id, userId: session.user.id });
        });
    } catch (error) {
        return { ok: false, message: "No se pudo guardar el producto" };
    }

    revalidatePath("/productos");
    return { ok: true, message: id ? "Producto actualizado" : "Producto creado" };
}
```

Since Prisma 5, `update` and `delete` accept non-unique filters (`companyId`) next to the
unique key in `where`, and throw `P2025` when the scoped record does not exist. That makes the
ownership check and the write a single call.

## Create and update

Before writing a create/update action, read in the analog:

1. `upsert` or explicit branching on `id`.
2. How relation sets are saved: replaced (`deleteMany` + `createMany`), diffed, or updated one by one.
3. Whether audit runs inside the transaction.
4. The response shape.

Every value passed to Prisma comes from `parsed.data`, never from the raw input.

## Pagination

```typescript
const where = { companyId, name: { contains: search, mode: "insensitive" as const } };
const skip = (page - 1) * take;

const [data, totalCount] = await Promise.all([
    prisma.entity.findMany({ where, skip, take, orderBy: { name: "asc" } }),
    prisma.entity.count({ where }),
]);

return { ok: true, data, page, totalPages: Math.ceil(totalCount / take), totalCount };
```

The tenant scope is part of `where` for both queries. `mode: "insensitive"` exists only on
PostgreSQL and MongoDB; match what the analog does for search.

## Response shapes

There is no universal contract. Shapes seen across projects:

```typescript
{ ok: true, message }
{ ok: true, data }
{ ok: true, data, page, totalPages, totalCount }
{ ok: false, message }
```

New actions return exactly the shape their module already returns.

## Transactions

Use one transaction when a change touches several related records, balances, inventory and
its movements, a sale and its items, relations that must stay in sync, or an audit record that
must be atomic with the change.

- Reads that the checks depend on happen inside the transaction when they must be consistent
  with the writes.
- **Returning a value from the callback commits.** A callback that returns
  `{ ok: false }` after some writes leaves those writes in the database. To roll back,
  throw; then catch outside and return the failure response.
- Every helper that writes receives `tx`. Read the helper's signature before assuming it is
  transaction-aware.

## Revalidation

Revalidate only what shows the changed data, with the API the repo already uses
(`revalidatePath`, `revalidateTag`, or `updateTag` on Next.js 16):

```typescript
revalidatePath("/productos");
revalidatePath("/pos");
```

## `after()`

`after()` runs after the response is sent: email, WhatsApp, low-stock alerts, secondary
revalidation. If the process stops, the work is lost, and it can run zero or more times.
Keep every step the business result depends on before the response.

## Deletes

Follow the entity's existing strategy, after reading the relation's `onDelete` rules:

- Soft delete: set the flag or `deletedAt`, and every read filters it out.
- Hard delete: remove the row; check cascades and restricted relations first.

Deletes are scoped like any other write: `where: { id, companyId }`.
