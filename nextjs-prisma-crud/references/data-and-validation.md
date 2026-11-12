# Data and validation

## Prisma

### Tenant scope

The tenant field is whatever the repo uses (`companyId`, `agencyId`, `accountId`,
`studioId`, `branchId`, ...). Every sensitive read and write filters by it:

```typescript
where: { id, companyId: session.user.companyId }
```

A model with no tenant field is scoped through its parent:

```typescript
where: { id, order: { companyId } }
```

If neither exists, the report names the model as unscoped.

### Relations

- An explicit join model holds relations that carry their own data (`Enrollment`,
  `CourseTeacher`, `RecipeItem`).
- Read `onDelete` on every relation of a model before deleting from it.
- Relation sets are saved the way the analog saves them: `deleteMany` + `createMany`, or a
  diff with upserts.

### Delete strategy

The model (`isActive`, `status`) and the existing delete action decide which strategy an
entity uses; it stays that way. A new model that other records reference, and whose history
must survive, gets `isActive Boolean @default(true)` instead of a hard delete.

### Schema and migration changes

- Read the migration history and the provider before editing `schema.prisma`.
- Write the schema change, then give the user the command (`prisma migrate dev --name ...`)
  instead of running it. Production databases are never touched.
- Environment configuration is checked for presence only; its values are never printed.
- Prisma 7 needs a driver adapter and a `prisma.config.ts`; Prisma 6 projects stay on their
  setup. Match the installed major version.

### Payments and balances

In one transaction:

1. Verify the account or user ID against the tenant scope.
2. Write the payment.
3. Recalculate the balance with the repo's reconciliation helper (never a direct balance write
   when a helper exists).
4. Write the audit record.

### Inventory

In one transaction: compute the delta, update stock, write the movement records, write the
sale items and payments. A delete runs the reverse operation in its own transaction.

### Audit

Helpers vary: a transaction-aware `createAuditLogTx(tx, ...)`, a plain `createLog`, or
`prisma.auditLog.create`. Use the local one. New sensitive operations write their audit record
inside the transaction.

## Zod

One schema per model, compact: it validates create, update, and the form. Check the
installed major version first; Zod 3 and Zod 4 differ where marked.

```typescript
// schema/sale.schema.ts — the child schema lives in its parent's file
import { z } from "zod";
import { PaymentMethod } from "@/app/generated/prisma/enums";   // or "@prisma/client" on Prisma 6

export const saleItemSchema = z.object({
    id: z.number().optional(),
    productId: z.number(),
    quantity: z.number().positive(),
    pricePerUnit: z.number(),
});

// soldById comes from the session in the action, not from the payload.
export const saleSchema = z.object({
    id: z.number().optional(),                      // absent on create
    clientId: z.number().nullable().optional(),
    paymentMethod: z.enum(PaymentMethod),           // Zod 3: z.nativeEnum(PaymentMethod)
    saleDate: z.string().optional(),                // ISO string from the input, converted in the action
    notes: z.string().optional(),
    isActive: z.boolean().default(true),
    items: z.array(saleItemSchema).min(1),
});

export type SaleInput = z.infer<typeof saleSchema>;
```

Rules:

- Fields mirror the Prisma model: `?` in Prisma is `.optional()` (or `.nullable().optional()`
  when the form can clear it). Database-managed fields (`createdAt`, `updatedAt`) and
  session-owned fields (tenant, author) stay out of the schema.
- Dates travel as strings (`<input type="date">` gives one) and become `Date` in the action:
  `followUpDueOn: rest.followUpDueOn ? new Date(rest.followUpDueOn) : null`.
- Enums come from the Prisma Client, so Zod and the database share one list.
- `.default()` only where the domain has a real default.
- Empty strings from forms become `null` only where `""` breaks something: unique columns
  and optional enums. One shared helper does it:

```typescript
// schema/helpers.ts
export const emptyToNull = (value: unknown) =>
    typeof value === "string" && value.trim() === "" ? null : value;

code: z.preprocess(emptyToNull, z.string().nullable().optional()),
```

- Cross-field rules use `.refine(..., { message, path })` (Zod 4 also accepts `error:`).
- `z.coerce` is for raw strings only (`FormData`, query strings), not for React Hook Form
  inputs that already hold numbers.
- Messages are in the app's language.

React Hook Form uses the same schema. When it has `.default()`, `preprocess`, or `coerce`,
input and output types differ; type the form with both:

```typescript
const form = useForm<z.input<typeof saleSchema>, unknown, z.output<typeof saleSchema>>({
    resolver: zodResolver(saleSchema),
    defaultValues: { id: sale?.id, notes: sale?.notes ?? "", items: [] },
});
```
