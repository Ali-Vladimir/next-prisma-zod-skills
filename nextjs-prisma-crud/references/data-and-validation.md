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

Soft and hard deletes both exist across projects. The model (`deletedAt`, `isActive`,
`status`) and the existing delete action decide which one an entity uses; it stays that way.

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

Check the installed major version first. Zod 3 and Zod 4 differ in the places marked below.

```typescript
// Server parse
const parsed = schema.safeParse(input);
```

```typescript
// React Hook Form
const form = useForm<z.infer<typeof entitySchema>>({
    resolver: zodResolver(entitySchema),
});
// With z.coerce on Zod 4, the input type is `unknown`. Type the form with both sides:
// useForm<z.input<typeof s>, unknown, z.output<typeof s>>
```

```typescript
// Prisma enums (import from the repo's Prisma Client path)
role: z.nativeEnum(Role),   // Zod 3
role: z.enum(Role),         // Zod 4 (nativeEnum is deprecated)
```

```typescript
// Optional, nullable, defaults
id: z.number().optional(),
notes: z.string().nullish(),
isActive: z.boolean().default(true),   // only when the domain has a real default
```

```typescript
// Coercion: only for string input (forms, FormData, query strings)
amount: z.coerce.number(),
date: z.coerce.date(),
```

```typescript
// Empty strings from forms → null
notes: z.preprocess(v => (v === "" ? null : v), z.string().nullish()),
```

```typescript
// Cross-field rule
.refine(d => d.clientId || d.newClient, {
    message: "Selecciona o crea un cliente",   // Zod 4 also accepts `error:`
    path: ["clientId"],
})
```

```typescript
// Nested payload for records saved together
const recipeSchema = z.object({
    id: z.number().optional(),
    name: z.string().min(1),
    items: z.array(z.object({
        ingredientId: z.number().int().min(1),
        quantity: z.number().positive(),
    })).min(1),
});
```

Rules:

- One schema serves create and update unless the module already splits them.
- The client and the server use the same schema; the server always parses again.
- `preprocess` and `default` appear only where the domain requires them.
- Error messages are written in the project's user-facing language.
