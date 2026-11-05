# Security review

Use this when reviewing existing actions, or when a change touches auth, permissions, or
tenant scope. The rules themselves live in `SKILL.md` under **Server-side rules**; this file is
the hunt list for where they break in practice.

## Common gaps

Check each action in scope for every item. These show up repeatedly in admin code:

1. **No auth call** in an action whose siblings have one.
2. **Permission without ownership**: the permission string is checked, but the target record
   is loaded by `id` alone, so any tenant's record can be changed.
3. **Unscoped detail or delete**: `get-by-id` or `delete` uses `where: { id }` with no tenant
   field.
4. **Parent scoped, children not**: the parent record is checked, but child IDs in the payload
   (`variantId`, `courseId`, `accountId`) are trusted.
5. **Swallowed transaction failure**: the callback returns a failure value (which commits the
   writes so far) and the action returns `ok: true` anyway.
6. **Audit outside the transaction**: the audit helper is called with `prisma` instead of `tx`.
7. **Data leak**: the response returns password hashes, tokens, or internal fields because it
   has no `select`.
8. **Client-only guard**: the only protection is a hidden button, a layout check, or a role
   check in a client component.

Scope examples worth testing against: a product variant belongs to the current company; a
class belongs to a studio the user can access; a payment's user belongs to the current
account; a voucher is changed only by the distributor who owns it; an inventory item and
everything it links to belong to the current company and branch.

## Reporting gaps

Each gap found gets: file and line, which item above it matches, and a concrete failing
scenario ("user of company A deletes product 42 of company B"). Gaps in code outside the
task are reported, and fixed only when the user asks.

Secrets, real personal data, and tokens never go into reports, tests, or fixtures.
