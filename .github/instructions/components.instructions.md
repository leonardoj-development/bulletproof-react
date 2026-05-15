---
applyTo: "src/components/**,src/features/**/components/**"
---

# Components & Styling

## Component categories

| Location | Purpose |
|----------|---------|
| `src/components/ui/` | Generic, reusable primitives (Button, Form, Dialog, Table…) |
| `src/components/layouts/` | Page layout shells (DashboardLayout, AuthLayout, ContentLayout) |
| `src/components/errors/` | Error boundary fallback components |
| `src/components/seo/` | `<Head>` wrapper (react-helmet-async) |
| `src/features/<name>/components/` | Components used only within that feature |

## Shadcn/UI pattern

Components are **copied into the codebase**, not installed as packages. They are owned by the project and can be freely modified. The source of truth is `src/components/ui/`.

## Utility: `cn()` for classNames

Always use `cn()` from `@/utils/cn` to combine Tailwind classes:

```ts
// src/utils/cn.ts
import { type ClassValue, clsx } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

Use it for conditional classes and merging:
```tsx
<div className={cn('base-class', isActive && 'active-class', className)} />
```

## CVA — component variants

Use `cva()` for components with multiple visual variants. Define all variants in one place:

```tsx
import { cva, type VariantProps } from 'class-variance-authority';
import { cn } from '@/utils/cn';

const buttonVariants = cva(
  'inline-flex items-center justify-center rounded-md font-medium transition-colors focus-visible:outline-none',
  {
    variants: {
      variant: {
        default: 'bg-primary text-primary-foreground hover:bg-primary/90',
        destructive: 'bg-destructive text-destructive-foreground',
        outline: 'border border-input bg-background',
      },
      size: {
        default: 'h-9 px-4 py-2',
        sm: 'h-8 px-3 text-xs',
        lg: 'h-10 px-8',
      },
    },
    defaultVariants: { variant: 'default', size: 'default' },
  },
);

type ButtonProps = React.ButtonHTMLAttributes<HTMLButtonElement> &
  VariantProps<typeof buttonVariants> & {
    asChild?: boolean;
  };

export const Button = ({ className, variant, size, asChild, ...props }: ButtonProps) => {
  const Comp = asChild ? Slot : 'button';
  return <Comp className={cn(buttonVariants({ variant, size }), className)} {...props} />;
};
```

## Radix UI primitives

Use Radix for unstyled, accessible components. Wrap them with Tailwind classes:

```tsx
import * as DialogPrimitive from '@radix-ui/react-dialog';

// Style the primitives, re-export with your own API
export const Dialog = DialogPrimitive.Root;
export const DialogContent = React.forwardRef<...>(({ className, ...props }, ref) => (
  <DialogPrimitive.Content
    ref={ref}
    className={cn('fixed left-1/2 top-1/2 ...', className)}
    {...props}
  />
));
```

Never use Radix primitives directly in feature components — always through the wrapped version in `src/components/ui/`.

## Composition over props

Prefer `children` / compound component patterns over passing many props:

```tsx
// Preferred
<FormDrawer
  triggerButton={<Button>Create</Button>}
  title="Create Discussion"
>
  <DiscussionForm />
</FormDrawer>

// Avoid
<FormDrawer
  buttonLabel="Create"
  formTitle="Create Discussion"
  formFields={[...]}
  onSubmit={handleSubmit}
/>
```

## Form components

Use the shared `<Form>` wrapper from `src/components/ui/form/`. It handles `FormProvider`, `zodResolver`, and submit:

```tsx
import { Form, Input, Textarea } from '@/components/ui/form';

<Form schema={createDiscussionInputSchema} onSubmit={(values) => mutation.mutate({ data: values })}>
  {({ register, formState }) => (
    <>
      <Input label="Title" registration={register('title')} error={formState.errors.title} />
      <Textarea label="Body" registration={register('body')} error={formState.errors.body} />
      <Button type="submit" isLoading={mutation.isPending}>Submit</Button>
    </>
  )}
</Form>
```

## Tailwind conventions

- Mobile-first responsive design: `sm:`, `md:`, `lg:` prefixes
- Color palette via CSS variables defined in `src/index.css` / `src/styles/globals.css`
- Animation utilities from `tailwindcss-animate`
- No inline `style` props — use Tailwind classes exclusively
- `tailwind.config.cjs` extends the default theme for project-specific tokens

## Component single responsibility

Each component should do one thing. Extract complex JSX into sub-components rather than adding more conditions inside a single render:

```tsx
// Avoid
const DiscussionView = () => (
  <div>
    {isLoading ? <Spinner /> : error ? <ErrorMessage /> : (
      <div>
        <h1>{discussion.title}</h1>
        {comments.map(c => <div key={c.id}>{/* complex JSX */}</div>)}
      </div>
    )}
  </div>
);

// Prefer
const DiscussionView = () => <Discussion />;
const Discussion = () => (
  <div>
    <DiscussionHeader />
    <CommentsList />
  </div>
);
```
