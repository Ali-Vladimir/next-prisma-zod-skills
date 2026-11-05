# UI and components

## Admin module shape

```text
page.tsx (server)
  └── Manager (client)
        ├── Table or list
        ├── Form (inside a dialog or its own route)
        └── Delete confirmation
```

Route groups and layouts stay as the repo defines them. A layout or a hidden menu item is
navigation, never access control: the actions enforce it.

## Server page

```typescript
export default async function ProductsPage({
    searchParams,
}: {
    searchParams: Promise<{ page?: string; search?: string }>;
}) {
    const session = await auth();
    if (!session?.user) redirect("/login");

    const { page = "1", search = "" } = await searchParams;   // a Promise since Next.js 15
    const [products, categories] = await Promise.all([
        getPaginatedProducts({ page: Number(page), search }),
        getCategories(),                                    // catalogs for selects
    ]);

    return <ProductManager initial={products} categories={categories} />;
}
```

Props crossing to client components are serializable: convert `Decimal` and `BigInt`
values the way the analog does.

## Manager

The client component that coordinates the module. It owns the selected entity, dialog open
state, delete confirmation, search input, pagination, URL sync, toasts, and the calls to
mutation actions.

## Form

Owns React Hook Form with the Zod resolver, default values for create and edit, field arrays
for nested items, payload construction, the action call, and loading state.

A shared form receives `entity: Entity | null`. The entity (or its `id`) decides the heading,
default values, success message, and whether the action creates or updates.

## Table or list

Receives data from the manager or page, renders rows, cards, or catalog entries, and emits
`onCreate`, `onEdit`, `onDelete`. Persistence lives in the actions.

## URL filters

```text
page.tsx reads searchParams → Manager reads useSearchParams()
  → Manager calls router.push() with new params → page.tsx renders again
```

Changing a search or filter resets `page` to `1`.

## Styling

Reuse what `package.json` and `components/ui` already have (commonly Tailwind CSS,
shadcn/ui or Radix, Lucide icons, Sonner). A new primitive or UI dependency comes only after
confirming nothing installed covers it.

## Generated UI (v0 and similar)

Generated UI is a starting point:

1. User-facing text in the project's language.
2. Wired to the existing actions and schemas; the backend stays as it is.
3. Authorization stays in the actions.
4. Rebuilt from the repo's own components and accessibility conventions.

## Landing and SEO pages

- Copy and metadata come from the real business, audience, and search terms.
- `metadata`, `robots`, and `sitemap` follow the repo's existing form.
- A dynamic route exists only when each page carries genuinely different content.
