# UI and components

## Admin module shape

```text
app/(app)/<ruta>/page.tsx          server: reads searchParams, loads data, renders the manager
app/(app)/<ruta>/ui/
  <Entity>Manager.tsx              client: search, pagination, dialog, delete confirmation
  <Entity>Form.tsx                 client: React Hook Form + Zod, calls create-update
  <Entity>Table.tsx                client: rows and callbacks
```

Route groups and layouts stay as the repo defines them. A layout or a hidden menu item is
navigation, never access control: the actions enforce it.

## Server page

The main entity is paginated here, from `searchParams`. Catalogs for selects load in the same
`Promise.all`, reusing existing actions.

```typescript
import type { PaginatedPageProps } from "@/interfaces";

const ProductosPage = async ({ searchParams }: PaginatedPageProps) => {
    const { page, search } = await searchParams;           // a Promise since Next.js 15
    const currentPage = Number(page ?? 1);
    const currentSearch = search ?? "";

    const [{ data, totalPages }, { data: categories }] = await Promise.all([
        getPaginatedProducts({ page: currentPage, take: 10, search: currentSearch }),
        getPaginatedCategories({ page: 1, take: 100, isActive: true }),
    ]);

    return (
        <ProductManager
            data={data ?? []}
            categories={categories ?? []}
            totalPages={totalPages ?? 1}
            currentPage={currentPage}
            currentSearch={currentSearch}
        />
    );
};

export default ProductosPage;
```

Props crossing to client components are serializable. Prefer `Float`/`Int` columns the way
the repo does; if a model uses `Decimal` or `BigInt`, follow the analog's conversion.

## Manager

Props extend the shared type: `interface Props extends PaginatedManagerProps<IProductExtended> { categories: Category[] }`.

It owns the selected entity, dialog open state, delete confirmation, search input, and toasts.
URL changes go through one `updateUrl` helper (`URLSearchParams` + `router.push`); a new
search resets `page` to `1`.

## Form

- Receives the base model: `product: Product | null`. Its presence decides the heading,
  the default values, and the success message; the same action creates or updates.
- `defaultValues` read straight from the entity (`product?.name ?? ""`); dates become
  `toISOString().slice(0, 10)` for `<input type="date">`.
- Submits the parsed form data to the action as-is, shows `toast.success` / `toast.error`
  with the action's `message`, and disables the submit button while pending.
- A related entity too large to preload (patients, products) is searched from the form: call
  its paginated action in a debounced `useEffect`.

## Table or list

Props are `PaginatedTableProps<IProductExtended>`. It renders rows (cards on mobile), an
empty state with a create button, and pagination, and emits `onCreate`, `onEdit`,
`onDelete`, `onPageChange`.

Enum values display through the shared `lib/labels.ts` maps, used by both form and table.

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

- Copy and metadata come from the real business, audience, and the phrases people actually
  search ("lentes en León", not the brand name).
- `metadata`, `robots`, and `sitemap` follow the repo's existing form.
- One layout can serve several keyphrase routes (city + service) through dynamic
  `generateMetadata`, as long as each page carries content genuinely useful for that search.
- Images under 1.5 MB, with `alt` text.
