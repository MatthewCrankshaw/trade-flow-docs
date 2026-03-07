# Phase 2: Visit Type Management UI - Research

**Researched:** 2026-02-28
**Domain:** React frontend - CRUD management UI within business settings
**Confidence:** HIGH

## Summary

This phase builds a Visit Type management UI within the existing Business Details page, closely mirroring the established JobTypesSection and ItemsDataView patterns. The backend CRUD API already exists from Phase 1 with four endpoints (list, get-by-id, create, update). The frontend work involves: creating a new `visit-types` feature module, adding a "Scheduling" tab to BusinessDetails, building the list/card responsive views, a form dialog for create/edit, a status filter for active/inactive views, and wiring up RTK Query endpoints.

A critical finding is that the backend list endpoint currently filters to ACTIVE-only visit types, but the user decisions require a status filter to switch between active/inactive views (matching the Items page pattern where all items are returned and filtered client-side). This means the backend repository must be updated to return ALL visit types before the frontend status filter can work. This is a required backend change within this phase's scope.

The CONTEXT.md also specifies a `color` field with a preset palette -- this is a new UI element not present in the job-types pattern. The backend already supports color (hex string validated with regex), so the frontend needs a color picker/swatch component. Additionally, the backend has an `icon` field that the create request requires. The CONTEXT.md does not mention icon selection, so the frontend should provide a sensible default icon value.

**Primary recommendation:** Follow the JobTypesSection pattern exactly for structure, but adapt the ItemsDataView/ItemsFilters pattern for the status filter. Create a small backend patch to remove the active-only filter from the list endpoint. Build a simple color swatch selector for the preset palette.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Follow the existing JobTypesSection pattern: table for desktop (768px+), card list for mobile
- Default visit types get a subtle primary-tinted background/border plus a "Default" badge -- same as job types
- Each item shows name and description only (no status display in the row)
- List sorted by creation order (not alphabetical, not defaults-first)
- Dialog form (modal) matching the JobTypeFormDialog pattern
- Fields: name (required), description (optional), color (preset palette of 8-12 colors)
- Color is a new field -- the backend will need a color property added to the visit type model
- Server-side uniqueness validation only -- let the API enforce and display the error
- New "Scheduling" tab in Business Details, alongside existing Job Types and Tax Rates tabs
- Within the Scheduling tab, a "Visit Types" section header with the list below
- Section header approach leaves room for future schedule settings in the same tab
- Edit via the same dialog form in edit mode -- click a visit type to open it
- Name field is always locked (visible but disabled) -- names are immutable once created, for both default and custom types
- Description and color are editable
- Deactivate/activate from the list row action menu (dropdown) AND from the dialog action menu (top-right)
- All visit types can be deactivated (defaults included), with the API enforcing at least one must remain active
- Use the items page pattern for status handling: a status filter to switch between active/inactive views (not a simple toggle)
- Illustration + "No visit types yet" message with a prominent "Create Visit Type" button
- The "all inactive" empty state is not possible -- API enforces at least one active type
- Skeleton loading matching the layout: skeleton rows for desktop, skeleton cards for mobile (reference: JobTypesCardSkeleton)
- Toast notifications for action failures (create, edit, deactivate)
- Full-page error state if the list fails to load

### Claude's Discretion
- Exact skeleton design and count
- Toast message wording
- Specific preset color palette choices
- Color swatch UI component implementation
- Empty state illustration choice

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| VTYPE-03 | User can create custom visit types | RTK Query mutation endpoint wired to POST `/v1/business/:businessId/visit-type`. Form dialog with name, description, color fields. Backend validates name uniqueness. |
| VTYPE-04 | User can view and manage visit types for their business | RTK Query query endpoint wired to GET `/v1/business/:businessId/visit-types`. Table/card responsive view. Edit dialog. Deactivate/activate via status toggle. Status filter for active/inactive views. Backend list endpoint must be updated to return all statuses. |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| React | 19.2.0 | UI framework | Already in use |
| TypeScript | 5.9.3 | Type safety | Already in use |
| @reduxjs/toolkit (RTK Query) | 2.11.2 | API state management | Already in use for all features |
| react-hook-form | 7.71.1 | Form management | Already in use for all forms |
| valibot | 1.2.0 | Schema validation | Already in use for all form schemas |
| @hookform/resolvers | 5.2.2 | Valibot-RHF bridge | Already in use |
| Tailwind CSS | 4.1.18 | Styling | Already in use |
| shadcn/ui (Radix primitives) | Various | UI components | Already in use |
| lucide-react | 0.563.0 | Icons | Already in use |
| sonner (via toast wrapper) | 2.0.7 | Toast notifications | Already in use via `@/lib/toast` |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| class-variance-authority | 0.7.1 | Component variants | If color swatch needs variant styles |

### Backend (Minor Change)
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| NestJS | 11.x | Backend framework | Modifying visit-type repository to remove active-only filter |

**Installation:** No new packages needed. All dependencies are already installed in both projects.

## Architecture Patterns

### Recommended Project Structure (Frontend)
```
trade-flow-ui/src/
├── features/
│   └── visit-types/              # NEW feature module
│       ├── components/
│       │   ├── VisitTypesSection.tsx        # Main section (mirrors JobTypesSection)
│       │   ├── VisitTypeFormDialog.tsx       # Create/edit dialog
│       │   ├── VisitTypesTable.tsx           # Desktop table view
│       │   ├── VisitTypesCardList.tsx        # Mobile card view
│       │   ├── VisitTypesCardSkeleton.tsx    # Mobile skeleton
│       │   ├── VisitTypesStatusFilter.tsx    # Active/inactive filter
│       │   ├── ColorPicker.tsx               # Preset color palette selector
│       │   └── index.ts                     # Barrel export
│       ├── hooks/
│       │   ├── useVisitTypesList.ts          # List fetching + filtering
│       │   ├── useVisitTypeActions.ts        # Mutations (create, update, toggle)
│       │   └── index.ts
│       ├── api/
│       │   ├── visitTypeApi.ts              # RTK Query endpoints
│       │   └── index.ts
│       └── index.ts                         # Feature barrel export
├── types/
│   └── api.types.ts                         # Add VisitType types
├── services/
│   ├── api.ts                               # Add "VisitType" tag type
│   └── index.ts                             # Re-export visit-type hooks
├── lib/forms/schemas/
│   └── visit-type.schema.ts                 # Valibot schema
└── features/business/components/
    └── BusinessDetails.tsx                  # Add "Scheduling" tab
```

### Backend Change
```
trade-flow-api/src/
└── visit-type/
    └── repositories/
        └── visit-type.repository.ts  # MODIFY: remove active-only filter from findVisitTypesByBusinessId
```

### Pattern 1: Feature Module (mirrors Job Types)
**What:** Self-contained feature with components, hooks, API, exported via barrel
**When to use:** Every new domain area in the frontend
**Example:**
```typescript
// src/features/visit-types/index.ts
export * from "./components";
export * from "./hooks";
export * from "./api";
```

### Pattern 2: RTK Query Endpoint Definition (mirrors jobTypeApi)
**What:** Inject endpoints into the shared apiSlice with tag-based invalidation
**When to use:** Every API integration
**Example:**
```typescript
// src/features/visit-types/api/visitTypeApi.ts
import { apiSlice } from "@/services/api";
import type { VisitType, CreateVisitTypeRequest, UpdateVisitTypeRequest, StandardResponse } from "@/types";

export const visitTypeApi = apiSlice.injectEndpoints({
  endpoints: (builder) => ({
    getVisitTypes: builder.query<VisitType[], string>({
      query: (businessId) => `/v1/business/${businessId}/visit-types`,
      transformResponse: (response: StandardResponse<VisitType>) => response.data,
      providesTags: (result) =>
        result
          ? [
              ...result.map(({ id }) => ({ type: "VisitType" as const, id })),
              { type: "VisitType", id: "LIST" },
            ]
          : [{ type: "VisitType", id: "LIST" }],
    }),
    createVisitType: builder.mutation<
      VisitType,
      { businessId: string; data: CreateVisitTypeRequest }
    >({
      query: ({ businessId, data }) => ({
        url: `/v1/business/${businessId}/visit-type`,
        method: "POST",
        body: data,
      }),
      transformResponse: (response: StandardResponse<VisitType>) => {
        if (response.data && response.data.length > 0) {
          return response.data[0];
        }
        throw new Error("No visit type data returned");
      },
      invalidatesTags: [{ type: "VisitType", id: "LIST" }],
    }),
    updateVisitType: builder.mutation<
      VisitType,
      { businessId: string; visitTypeId: string; data: UpdateVisitTypeRequest }
    >({
      query: ({ businessId, visitTypeId, data }) => ({
        url: `/v1/business/${businessId}/visit-type/${visitTypeId}`,
        method: "PATCH",
        body: data,
      }),
      transformResponse: (response: StandardResponse<VisitType>) => {
        if (response.data && response.data.length > 0) {
          return response.data[0];
        }
        throw new Error("No visit type data returned");
      },
      invalidatesTags: (_result, _error, { visitTypeId }) => [
        { type: "VisitType", id: visitTypeId },
        { type: "VisitType", id: "LIST" },
      ],
    }),
  }),
});

export const {
  useGetVisitTypesQuery,
  useCreateVisitTypeMutation,
  useUpdateVisitTypeMutation,
} = visitTypeApi;
```

### Pattern 3: Status Filter (mirrors Items page)
**What:** Client-side status filtering from a single API call that returns all items
**When to use:** Any list that supports active/inactive views
**Example:**
```typescript
// src/features/visit-types/hooks/useVisitTypesList.ts
const [statusFilter, setStatusFilter] = useState<"active" | "inactive">("active");

const statusFilteredTypes = useMemo(() => {
  return visitTypes.filter((vt) => vt.status === statusFilter);
}, [visitTypes, statusFilter]);
```

### Pattern 4: Responsive DataView (mirrors JobTypesSection)
**What:** Table for desktop (768px+), card list for mobile, with `useMediaQuery`
**When to use:** All list views
**Example:**
```typescript
const isDesktop = useMediaQuery("(min-width: 768px)");

return isDesktop ? (
  <VisitTypesTable ... />
) : (
  <div className="min-w-0 overflow-hidden">
    <VisitTypesCardList ... />
  </div>
);
```

### Pattern 5: Dialog Form with Create/Edit Mode (mirrors JobTypeFormDialog)
**What:** Single dialog component that handles both create and edit, with form reset on open
**When to use:** All entity create/edit dialogs
**Key aspects:**
- `mode: "create" | "edit"` prop
- `useEffect` to reset form when mode/entity changes
- Name field disabled in edit mode (immutability)
- Action menu (MoreHorizontal) in dialog header for edit mode
- `FormProvider` wrapping the form for react-hook-form

### Anti-Patterns to Avoid
- **Filtering at the API level for UI status views:** The Items pattern fetches ALL items and filters client-side. Do NOT use separate API calls for active vs inactive.
- **Custom form state management:** Do NOT use `useState` for form fields. Use `react-hook-form` with `valibot` resolver.
- **Direct sonner imports:** Do NOT import `sonner` directly. Use `toast` from `@/lib/toast`.
- **Raw Redux hooks:** Do NOT use `useDispatch`/`useSelector`. Use `useAppDispatch`/`useAppSelector`.
- **Inline API calls:** Do NOT use `fetch` or `axios`. Use RTK Query endpoint hooks.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Form state | Custom useState-based form | react-hook-form + valibot resolver | Existing pattern, handles validation, dirty tracking, reset |
| API caching | Custom fetch + state management | RTK Query with tag invalidation | Already configured, handles auth token, cache invalidation |
| Responsive detection | window.addEventListener resize | `useMediaQuery` from `@/hooks/useMediaQuery` | Already exists, SSR-safe, uses `useSyncExternalStore` |
| Toast notifications | Custom notification system | `toast` from `@/lib/toast` | Already configured with icons, types, consistent styling |
| Form validation | Manual validation logic | Valibot schema + RHF resolver | Existing pattern in all other features |
| Status toggle | Custom status management | RTK Query mutation with tag invalidation | Matches item/job-type patterns exactly |

**Key insight:** This feature is almost entirely cloning existing patterns. The only genuinely new UI element is the color picker/swatch component.

## Common Pitfalls

### Pitfall 1: Backend Returns Active-Only Visit Types
**What goes wrong:** Status filter shows empty inactive list because the API never returns inactive visit types
**Why it happens:** Visit type repository has `status: VisitTypeStatus.ACTIVE` hardcoded in `findVisitTypesByBusinessId`
**How to avoid:** Modify the repository to remove the active-only filter, matching the Items repository pattern (which has no status filter on list queries)
**Warning signs:** Inactive tab always shows 0 items; reactivating a deactivated type fails to show it

### Pitfall 2: "VisitType" Tag Not Registered in apiSlice
**What goes wrong:** RTK Query cache invalidation does not work; list doesn't refresh after create/update
**Why it happens:** `apiSlice` in `src/services/api.ts` has a whitelist of `tagTypes` -- must add "VisitType"
**How to avoid:** Add "VisitType" to the `tagTypes` array in `src/services/api.ts` BEFORE defining endpoints
**Warning signs:** Creating a visit type succeeds but doesn't appear in the list until page refresh

### Pitfall 3: Backend Create Requires Icon Field
**What goes wrong:** API returns 400/422 when creating a visit type without an icon
**Why it happens:** `CreateVisitTypeRequest` has `@IsString() @IsNotEmpty() icon: string` as required
**How to avoid:** Either provide a default icon value in the create request body, or modify the backend to make icon optional with a default. Since CONTEXT.md does not mention icon selection, the frontend should send a sensible default (e.g., "calendar" or "clipboard").
**Warning signs:** Create mutation fails with validation error about icon field

### Pitfall 4: Color Hex Validation Mismatch
**What goes wrong:** API rejects color values that don't match the expected pattern
**Why it happens:** Backend validates `@Matches(/^#[0-9A-Fa-f]{6}$/)` -- must be exactly `#RRGGBB` format
**How to avoid:** Ensure the color palette preset values are all valid 6-digit hex codes with `#` prefix
**Warning signs:** Create/update fails silently or shows generic error

### Pitfall 5: Name Immutability Not Visually Clear
**What goes wrong:** Users try to edit the name field and get confused
**Why it happens:** The name field appears in edit mode but is disabled without explanation
**How to avoid:** Show the name as a read-only display field (not an input) or add helper text "Name cannot be changed after creation"
**Warning signs:** User confusion, support requests

### Pitfall 6: Empty State Shows Despite Having Types
**What goes wrong:** "No visit types yet" shows when all types are inactive and status filter is set to active
**Why it happens:** Status-filtered list is empty even though types exist
**How to avoid:** Differentiate between "no types at all" (show empty state with create button) vs "no types matching filter" (show message like "No inactive visit types" without the prominent create button)
**Warning signs:** Confusing UX when user deactivates their last visible type

### Pitfall 7: Forgetting to Re-export from Services Barrel
**What goes wrong:** Imports like `useCreateVisitTypeMutation` from `@/services` fail
**Why it happens:** New API hooks need to be added to `src/services/index.ts`
**How to avoid:** Add visit-type exports to `src/services/index.ts` matching the pattern of other features
**Warning signs:** TypeScript compilation errors on import statements

### Pitfall 8: Last Active Guard Error Not Surfaced
**What goes wrong:** User tries to deactivate the last active type and sees a generic error
**Why it happens:** Backend returns `InvalidRequestError` with code `VISIT_TYPE_LAST_ACTIVE_CANNOT_ARCHIVE` but frontend shows generic toast
**How to avoid:** Parse the error response and show a specific message: "Cannot deactivate the last active visit type"
**Warning signs:** Generic "Failed to deactivate" message without explanation

## Code Examples

### Visit Type TypeScript Interface (for api.types.ts)
```typescript
// Source: derived from IVisitTypeResponse in trade-flow-api
export type VisitTypeStatus = "active" | "inactive";

export interface VisitType {
  id: string;
  name: string;
  description: string | null;
  color: string;
  icon: string;
  status: VisitTypeStatus;
}

export interface CreateVisitTypeRequest {
  name: string;
  description?: string | null;
  color: string;
  icon: string;
}

export interface UpdateVisitTypeRequest {
  description?: string | null;
  color?: string;
  icon?: string;
  status?: VisitTypeStatus;
}
```

### Valibot Form Schema
```typescript
// Source: derived from job-type.schema.ts pattern + CONTEXT.md color field
import * as v from "valibot";

export const visitTypeFormSchema = v.object({
  name: v.pipe(
    v.string(),
    v.minLength(1, "Name is required"),
    v.maxLength(100, "Name must be less than 100 characters"),
  ),
  description: v.optional(v.string()),
  color: v.pipe(
    v.string(),
    v.regex(/^#[0-9A-Fa-f]{6}$/, "Invalid color format"),
  ),
});

export type VisitTypeFormValues = v.InferOutput<typeof visitTypeFormSchema>;
```

### BusinessDetails Tab Addition
```typescript
// Source: existing BusinessDetails.tsx structure
// Add to imports:
import { Calendar } from "lucide-react";
import { VisitTypesSection } from "@/features/visit-types";

// Add to TabsList (after job-types trigger):
<TabsTrigger value="scheduling" className="gap-2">
  <Calendar className="h-4 w-4" />
  Scheduling
</TabsTrigger>

// Add TabsContent:
<TabsContent value="scheduling">
  <VisitTypesSection businessId={businessId} />
</TabsContent>
```

### Color Swatch Preset Palette
```typescript
// Claude's discretion -- recommended palette of 8 colors
// All are valid #RRGGBB hex codes per backend validation
export const VISIT_TYPE_COLORS = [
  { value: "#3B82F6", label: "Blue" },
  { value: "#10B981", label: "Green" },
  { value: "#F59E0B", label: "Amber" },
  { value: "#EF4444", label: "Red" },
  { value: "#8B5CF6", label: "Purple" },
  { value: "#EC4899", label: "Pink" },
  { value: "#06B6D4", label: "Cyan" },
  { value: "#F97316", label: "Orange" },
] as const;
```

### Backend Repository Fix (remove active-only filter)
```typescript
// Source: trade-flow-api/src/visit-type/repositories/visit-type.repository.ts
// BEFORE (current):
public async findVisitTypesByBusinessId(
  businessId: string,
  queryOptions: IBaseQueryOptionsDto,
): Promise<DtoCollection<IVisitTypeDto>> {
  const filter: Filter<IVisitTypeEntity> = {
    businessId: new ObjectId(businessId),
    status: VisitTypeStatus.ACTIVE,  // <-- REMOVE THIS LINE
  };
  // ...
}

// AFTER (matching items pattern):
public async findVisitTypesByBusinessId(
  businessId: string,
  queryOptions: IBaseQueryOptionsDto,
): Promise<DtoCollection<IVisitTypeDto>> {
  const filter: Filter<IVisitTypeEntity> = {
    businessId: new ObjectId(businessId),
  };
  // ...
}
```

### Error Handling for Last-Active Guard
```typescript
// Source: pattern from existing error handling + visit-type updater service
const handleDeactivate = async (visitType: VisitType) => {
  try {
    await updateVisitType({
      businessId,
      visitTypeId: visitType.id,
      data: { status: "inactive" },
    }).unwrap();
    toast.success("Visit type deactivated", {
      description: `"${visitType.name}" has been deactivated.`,
    });
  } catch (error) {
    const apiError = error as { data?: { errors?: ApiError[] } };
    const errorMessage = apiError?.data?.errors?.[0]?.message;

    toast.error("Failed to deactivate visit type", {
      description: errorMessage || "Please try again.",
    });
  }
};
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Job types hides inactive (filters in component) | Items page uses status filter dropdown | Already in codebase | Visit types should follow Items pattern per CONTEXT.md |
| Job types shows status column | Visit types omit status from row | CONTEXT.md decision | Simpler row, status managed via filter view |

**Note on backend response field -- `isDefault`:** The backend visit-type response does NOT include an `isDefault` field. The JobType response includes `isDefault: boolean` which is used for the "Default" badge. For visit types, we need to determine how to identify defaults. Options:
1. The backend could add an `isDefault` field to the response (backend change)
2. We can infer "default" from some other property

Looking at the default visit types creator from Phase 1, let me verify this.

**Important finding:** The visit-type DTO does NOT have an `isDefault` field. The `IVisitTypeResponse` returns: `id, name, description, color, icon, status`. The job-type DTO DOES have `isDefault`. Since the CONTEXT.md says "Default visit types get a subtle primary-tinted background/border plus a Default badge", the backend needs to be extended to include an `isDefault` field, OR the frontend needs another way to identify defaults. This is a backend gap that must be addressed.

## Open Questions

1. **isDefault field missing from visit-type backend**
   - What we know: `IVisitTypeResponse` has `{id, name, description, color, icon, status}` -- no `isDefault` field. The `IVisitTypeDto` and `IVisitTypeEntity` also lack this field.
   - What's unclear: Whether to add `isDefault` to the backend data model or find another approach
   - Recommendation: Add `isDefault: boolean` to the visit-type entity, DTO, and response -- matching the JobType pattern exactly. Set `isDefault: true` for types created by `DefaultVisitTypesCreator` and `isDefault: false` for custom types. This requires a backend migration for existing data and changes to the creator service. Alternatively, if migration is too heavy, the default visit types could be identified by a naming convention, but this is fragile.

2. **Icon field is required on create but not mentioned in CONTEXT.md**
   - What we know: `CreateVisitTypeRequest` requires `icon: string` (`@IsString() @IsNotEmpty()`). CONTEXT.md only mentions name, description, and color fields.
   - What's unclear: Whether to add icon selection UI or use a default
   - Recommendation: Send a default icon value (e.g., `"calendar"`) from the frontend for all created visit types. Optionally make the icon field optional on the backend with a default value. This avoids adding UI complexity not requested.

3. **Sorting by creation order**
   - What we know: CONTEXT.md says "List sorted by creation order (not alphabetical, not defaults-first)". The backend returns items in MongoDB's natural insertion order by default, which is creation order.
   - What's unclear: Whether the backend guarantees creation order in the response
   - Recommendation: Rely on backend ordering (MongoDB natural order IS insertion order for non-sharded collections). No explicit sort needed on the frontend. If order is inconsistent, add `sort: { _id: 1 }` to the backend query (ObjectIds are time-ordered).

## Sources

### Primary (HIGH confidence)
- **Existing codebase** -- all patterns directly verified by reading source files
  - `trade-flow-ui/src/features/job-types/` -- reference implementation for section, form dialog, card list, skeleton
  - `trade-flow-ui/src/features/items/` -- reference implementation for status filter, data view, hooks
  - `trade-flow-ui/src/features/business/components/BusinessDetails.tsx` -- tab structure to modify
  - `trade-flow-api/src/visit-type/` -- complete backend module from Phase 1
  - `trade-flow-ui/src/services/api.ts` -- RTK Query base config with tag types
  - `trade-flow-ui/src/types/api.types.ts` -- all existing type definitions
  - `trade-flow-ui/src/lib/forms/schemas/job-type.schema.ts` -- valibot schema pattern
  - `trade-flow-ui/src/lib/toast.tsx` -- toast notification utility

### Secondary (MEDIUM confidence)
- **Phase 1 Summary** (`.planning/phases/01-visit-type-backend/01-01-SUMMARY.md`) -- confirms API endpoints, data model, and patterns established

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all libraries already in use, no new dependencies
- Architecture: HIGH -- direct pattern cloning from job-types and items features
- Pitfalls: HIGH -- identified by reading actual source code and comparing patterns
- Backend gap (active-only filter): HIGH -- confirmed by reading repository source code
- Backend gap (isDefault field): HIGH -- confirmed by reading DTO, entity, and response interfaces
- Color picker: MEDIUM -- new UI component, but simple implementation (just styled buttons)

**Research date:** 2026-02-28
**Valid until:** 2026-03-28 (stable -- all patterns are established in codebase)
