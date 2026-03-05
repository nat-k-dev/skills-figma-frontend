---
name: rcl-create-component
description: Use this skill when the user wants to implement a new React component that is pixel-perfect to Figma AND has semantic variants. Requires a component spec (from figma-component-inspect or described by the user). Creates all files following the project's shadcn-style compound component patterns.
version: 1.0.0
last_updated: 2026-03-05
project: "@junioreinsteinbv/react-component-library"
---

# Create Component from Figma Spec

**This skill only applies to the project with `"name": "@junioreinsteinbv/react-component-library"` in `package.json`.** Verify this before proceeding.

Implements a complete React component following the project's established patterns. Always run `figma-component-inspect` first to produce the spec this skill consumes.

---

## Project Patterns (read these first)

Before implementing, read these reference files to match existing conventions:

- `src/components/Button/Button.tsx` — variant/type/color TypeScript union pattern
- `src/components/Button/buttonVariant.constants.ts` — semantic variant map format
- `src/components/Button/buttonVariants.constants.ts` — CVA format
- `src/components/Badge/Badge.tsx` — simpler compound-free component
- `src/components/InputGroup/InputGroup.tsx` + `InputGroupAddon.tsx` — compound component with separate files
- `src/components/Badge/Badge.stories.tsx` — stories format

---

## File Structure

Every component lives in `src/components/{ComponentName}/`:

```
ComponentName.tsx                    — root component
ComponentTitle.tsx                   — sub-component (if compound)
ComponentDescription.tsx             — sub-component (if compound)
ComponentAction.tsx                  — sub-component (if compound)
componentVariant.constants.ts        — semantic variant map (CSS tokens)
componentVariants.constants.ts       — CVA (base + type/color compound variants)
index.ts                             — re-exports everything
ComponentName.stories.tsx            — Storybook stories
```

One file per sub-component. No exceptions.

---

## Step 1 — Discuss semantic variants with the user BEFORE writing any code

**Semantic variants do not exist in Figma — never silently decide their values.**

Before touching any file, explicitly ask the user:

1. **Which semantic variants are needed?** (e.g. `primary`, `secondary`, `success`, `error`, `warning`, `info` — or a subset)
2. **What `type` + `color` combination should each variant map to?** Present your suggestion as a table and wait for confirmation:

```
| variant   | type  | color  |
|-----------|-------|--------|
| primary   | solid | blue   |  ← suggestion
| secondary | soft  | gray   |  ← suggestion
| success   | soft  | green  |  ← suggestion
| error     | soft  | red    |  ← suggestion
| warning   | soft  | yellow |  ← suggestion
| info      | soft  | blue   |  ← suggestion
```

Do not proceed to Step 2 until the user approves or adjusts this mapping.

---

## Step 2 — CSS tokens (tokens.css + globals.css)

### Check for existing tokens first

Read `src/styles/tokens.css` for any existing tokens for this component. If old tokens exist that will be superseded:
1. Check `src/styles/globals.css` for any references to the old tokens — fix broken references before deleting
2. Delete old tokens, add new ones

### Token naming convention

In `tokens.css` (under `:root`):
```css
/* ComponentName - Variant: Primary */
--je-componentname-primary-bg: var(--je-colors-sky-600);
--je-componentname-primary-text: var(--je-colors-white-100);
--je-componentname-primary-border: transparent;
```

Pattern: `--je-{componentname}-{variant}-{property}` where property is `bg`, `text`, or `border`.

In `globals.css` (inside `@theme`):
```css
/* ComponentName variant tokens */
/* Primary */
--color-componentname-primary-bg: var(--je-componentname-primary-bg);
--color-componentname-primary-text: var(--je-componentname-primary-text);
--color-componentname-primary-border: var(--je-componentname-primary-border);
```

Pattern: `--color-{componentname}-{variant}-{property}`.

Token values in `tokens.css` use `--je-colors-*` primitives from `primitives.css`. Always verify the primitive exists before using it.

---

## Step 3 — `componentVariant.constants.ts`

Semantic variant → Tailwind CSS token classes. No hover/focus unless the component needs them.

```ts
export const componentVariant = {
  none: '',
  primary:   'bg-componentname-primary-bg text-componentname-primary-text border-componentname-primary-border',
  secondary: 'bg-componentname-secondary-bg text-componentname-secondary-text border-componentname-secondary-border',
  success:   'bg-componentname-success-bg text-componentname-success-text border-componentname-success-border',
  error:     'bg-componentname-error-bg text-componentname-error-text border-componentname-error-border',
  warning:   'bg-componentname-warning-bg text-componentname-warning-text border-componentname-warning-border',
  info:      'bg-componentname-info-bg text-componentname-info-text border-componentname-info-border',
};
```

---

## Step 4 — `componentVariants.constants.ts`

CVA combining base styles, semantic variant, and type/color compound variants.

```ts
import { cva } from 'class-variance-authority';
import { componentVariant } from './componentVariant.constants';

export const componentVariants = cva(
  'BASE_CLASSES_FROM_SPEC',
  {
    variants: {
      variant: componentVariant,
      type: {
        solid: 'border-transparent',   // no visible border for solid
        soft:  '',                      // border color comes from compound variants
        white: 'bg-white text-gray-800 border-gray-200', // single combination — full styles here
      },
      color: {
        dark: '', gray: '', green: '', blue: '', red: '', yellow: '', light: '',
      },
    },
    compoundVariants: [
      // Solid (pixel-perfect from Figma — use sky-600 not blue-500 for blue!)
      { type: 'solid', color: 'dark',   className: 'bg-gray-800 text-white' },
      { type: 'solid', color: 'blue',   className: 'bg-sky-600 text-white' },
      // ... all combinations
      // Soft (pixel-perfect from Figma)
      { type: 'soft', color: 'green',  className: 'bg-green-100 text-green-800 border-green-200' },
      // ... all combinations
    ],
    defaultVariants: {
      variant: 'none',
    },
  },
);
```

**Important:** `type='white'` has exactly one color combination in Figma. Put all its styles in the `type` variant directly — do not add compound variants for white type.

---

## Step 5 — Root component (`ComponentName.tsx`)

Follow the Button/Badge discriminated union pattern exactly:

```tsx
import { forwardRef, type ComponentPropsWithoutRef } from 'react';
import { type VariantProps } from 'class-variance-authority';
import { cn } from '@/lib/utils';
import { componentVariants } from './componentVariants.constants';

type NamedVariant = 'primary' | 'secondary' | 'success' | 'error' | 'warning' | 'info';
export type ComponentVariant = 'none' | NamedVariant;
export type ComponentType = 'solid' | 'soft' | 'white';
export type ComponentColor = 'dark' | 'gray' | 'green' | 'blue' | 'red' | 'yellow' | 'light';

type BaseProps = Omit<
  ComponentPropsWithoutRef<'div'> & VariantProps<typeof componentVariants>,
  'type' | 'color' | 'variant'
>;

/** Variant mode: named variant; type and color are forbidden. */
type PropsVariantMode = BaseProps & {
  variant: NamedVariant;
  type?: never;
  color?: never;
};

/** Color/type mode: variant is 'none' or omitted; type and color are optional. */
type PropsColorTypeMode = BaseProps & {
  variant?: 'none';
  type?: ComponentType;
  color?: ComponentColor;
};

export type ComponentProps = PropsVariantMode | PropsColorTypeMode;

const isVariantMode = (v: ComponentProps['variant']): v is NamedVariant =>
  v !== undefined && v !== 'none';

const Component = forwardRef<HTMLDivElement, ComponentProps>(
  ({ className, variant = 'none', type: typeProp, color, ...props }, ref) => {
    const resolvedVariant = variant ?? 'none';
    const resolvedType = isVariantMode(resolvedVariant) ? undefined : (typeProp ?? 'soft');
    const resolvedColor = isVariantMode(resolvedVariant) ? undefined : (color ?? 'blue');

    return (
      <div
        ref={ref}
        data-slot="component-name"
        role="alert"  // or appropriate ARIA role
        className={cn(componentVariants({ variant: resolvedVariant, type: resolvedType, color: resolvedColor, className }))}
        {...props}
      />
    );
  },
);

Component.displayName = 'Component';
export { Component };
```

**Default fallbacks:** `typeProp ?? 'soft'` and `color ?? 'blue'` — choose sensible defaults for the component.

---

## Step 6 — Sub-components

Each sub-component in its own file. Plain `forwardRef` div with `data-slot`. No polymorphism unless the reference `src/ui/` file uses it.

```tsx
import { forwardRef, type ComponentPropsWithoutRef } from 'react';
import { cn } from '@/lib/utils';

const ComponentTitle = forwardRef<HTMLDivElement, ComponentPropsWithoutRef<'div'>>(
  ({ className, ...props }, ref) => (
    <div
      ref={ref}
      data-slot="component-title"
      className={cn('TYPOGRAPHY_FROM_SPEC', className)}
      {...props}
    />
  ),
);

ComponentTitle.displayName = 'ComponentTitle';
export type ComponentTitleProps = ComponentPropsWithoutRef<'div'>;
export { ComponentTitle };
```

### Layout strategy for icon + title + description

Do NOT create a `<Content>` wrapper sub-component. Use CSS grid on the root with `has-[>svg]` detection:

```css
/* On root */
grid
has-[>svg]:grid-cols-[auto_1fr]
gap-x-{icon-content-gap}
gap-y-{title-description-gap}
*:[svg]:row-span-2 *:[svg]:self-start *:[svg]:mt-0.5 *:[svg]:shrink-0
```

This makes icon + title + description layout work automatically without a wrapper.

### Action slot (dismiss button)

```tsx
// Action sub-component — absolute positioned
className="absolute top-2.5 right-3"
data-slot="component-action"

// On root — add right padding when action is present
'has-[[data-slot=component-action]]:pr-9'
```

---

## Step 7 — `index.ts`

Re-export everything:

```ts
export { Component, componentVariants, type ComponentProps, type ComponentVariant, type ComponentType, type ComponentColor } from './Component';
export { ComponentTitle, type ComponentTitleProps } from './ComponentTitle';
export { ComponentDescription, type ComponentDescriptionProps } from './ComponentDescription';
export { ComponentAction, type ComponentActionProps } from './ComponentAction';
```

---

## Step 8 — `ComponentName.stories.tsx`

Follow `Badge.stories.tsx` format. Required stories:

1. **Default** — minimal usage with args
2. **SemanticVariants** — all semantic variants in a column, with appropriate icons
3. **TypeColorMatrix** — all type × color combinations in a grid
4. **WithIcons** — real-world examples with lucide-react icons
5. **WithDescription** — title + description
6. **WithAction** (if applicable) — with dismiss button

Import icons from `lucide-react`. Use `AlertCircle`, `AlertTriangle`, `CheckCircle`, `Info`, `X` for alert-style components.

In stories that override variant, destructure and discard `type`/`color` from args:
```tsx
const { type: _t, color: _c, ...restArgs } = args;
```

---

## Step 9 — Verify

Run `getDiagnostics` to check for TypeScript errors. Fix any before finishing.
