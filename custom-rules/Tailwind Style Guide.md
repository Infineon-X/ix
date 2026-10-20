# Tailwind Styling Playbook

## Design Foundations
- **Color Tokens**: Use the CSS variables defined in `tailwind.config.ts` (`primary`, `secondary`, `muted`, etc.) instead of hard-coded hex values. Pair them with opacity utilities (`/10`, `/20`) to create layered surfaces as seen in `components/user-document-requirements.tsx`.
- **Radii & Shadows**: Respect the design system radii (`--radius`) by reusing components like `Card` and `Button`. For custom blocks, stick with `rounded-lg` for primary surfaces and only step down (`rounded-md`) for nested elements.
- **Type Stack**: `app/layout.tsx` loads Roboto weights 400/500/700. Default to `font-medium` for actionable text, reserve `font-semibold`+ for section headings.

## Layout & Spacing System
- **Containers**: Anchor pages with `container` and `px-6 md:px-8` on wrappers. Admin dashboards rely on `flex` layouts with `gap-6`; reuse that rhythm for new sections.
- **Stacking**: Prefer `space-y-*` for vertical spacing so additions inherit consistent gaps. Example: `components/pending-documents.tsx` uses `space-y-4` to align cards.
- **Elevation Bands**: Background scaffolding follows `bg-background` for base, `bg-secondary/60 backdrop-blur-sm` for translucent panels, and `bg-card` (`Card` component) for primary content blocks.
- **Grid Rules**: When presenting metrics (`app/admin/page.tsx`), start with `grid grid-cols-1 gap-6 md:grid-cols-2 xl:grid-cols-4`. Expand incrementally rather than redefining breakpoints per card.

## Surface Patterns
- **Cards**: Reach for `Card`, `CardHeader`, and `CardContent` wrappers. Augment via `className` while keeping the base tokens (`border`, `shadow-sm`).
- **Callouts**: Alerts mix `bg-primary/10` + `border-primary/20` + `rounded-lg`. Mirror this trio for other semantic notice blocks (swap to `success`, `warning`, etc.).
- **Data Rows**: Use `border` + `rounded-lg` + `transition-colors` for list items to match `UserDocumentRequirements`. State severity adjusts both border and background opacity.

## Typography & Content Formatting
- **Section Titling**: Combine `text-foreground text-lg font-semibold` with iconography sized `h-5 w-5` for dashboard and settings headers.
- **Body Copy**: Use `text-sm text-muted-foreground` for supportive text. Only promote to `text-base` when legibility demands (forms, descriptions over 2 lines).
- **Mono Highlights**: Inline tokens (`applicationId`, IDs) use `font-mono bg-primary/20 px-2 py-1 rounded`. Adopt this capsule pattern for user-facing identifiers.

## Interactive Components
- **Buttons**: Always route through `Button` or `buttonVariants`. If you need a custom affordance, compose `cn(buttonVariants({ variant: 'ghost', size: 'sm' }), 'gap-1')` instead of rewriting Tailwind strings.
- **Badges & Status Pills**: Base on `<Badge>` with semantic overrides (`bg-destructive/10 text-destructive`) so hover/focus states stay consistent.
- **Links**: Pair `underline` with `underline-offset-4` and `hover:text-primary` for external anchors (`components/footer.tsx`).
- **Loading States**: Follow the circular spinner pattern (`animate-spin` + `border-b-2 border-primary`) while centering with `flex items-center justify-center`.

## Responsive & Dark Mode
- **Breakpoint Strategy**: Mobile-first, escalate at `md` for layout shifts and `lg` for admin chrome (e.g., sidebar toggles in `admin-layout-client`). Guard interactive reveals with `lg:hidden` / `lg:flex` pairs.
- **Dark Theme**: Because colors derive from CSS variables, use `dark:` modifiers sparingly (only when explicit contrast tweaks are needed, as in status badges). Validate both themes before merging.
- **Height Utilities**: Use `min-h-screen` for full views and `h-[calc(100vh-4rem)]` for shell layouts so sticky headers and sidebars stay aligned.

## Implementation Checklist
1. Compose new surfaces from existing primitives (`Card`, `Badge`, `Button`) before introducing raw `<div>` wrappers.
2. Group classes logically: layout (`flex`, `grid`), spacing (`gap-*`, `p-*`), then visuals (`bg-*`, `border-*`, `text-*`), and finish with state/helpers (`transition`, `hover:*`).
3. Extract repeated patterns into helper components or `cva` variants when you copy a Tailwind block more than twice.
4. Run `pnpm lint` and ensure `cn()` merges custom classNames with defaults to avoid duplicate utilities.
5. Screenshot responsive breakpoints (mobile, md, lg) for PRs that touch layout or color tokens.

Adhering to these rules keeps every screen aligned with the existing Admin dashboard, document requirements UI, and onboarding flows while leaving room for future theming changes driven by CSS variables.
