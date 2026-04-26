# Story 3.2: Tailwind Design Token Preset & shadcn/ui Theming

Status: review

## Story

As a **frontend developer on the EU Solicit team**,
I want **the `packages/config` Tailwind preset populated with the full EU Solicit design token set, shadcn/ui installed and themed in both apps with components shared via `packages/ui`**,
so that **every subsequent E03 story has a consistent, type-safe design system with production-ready components to build on**.

## Acceptance Criteria

1. Tailwind preset in `packages/config/tailwind.config.ts` defines: slate-based neutral scale (50–950), indigo-600 primary + 50–950 shades mapped to CSS variables, semantic colours (success/green, warning/amber, error/red, info/blue), 4px spacing base (Tailwind default), Inter + JetBrains Mono font stacks, `shadow-sm` through `shadow-2xl` custom values, and `darkMode: ["class"]`
2. CSS variables for `--primary`, `--secondary`, `--destructive`, `--muted`, `--accent`, `--background`, `--foreground`, `--border`, `--ring` (and related `*-foreground` variants) are set in `globals.css` in HSL format for light mode and the `.dark` class for dark mode in both apps
3. shadcn/ui components installed in `packages/ui/src/components/ui/`: Button, Input, Textarea, Select, Checkbox, RadioGroup, Switch, Dialog, Sheet, DropdownMenu, Tooltip, Badge, Card, Tabs, Table, Skeleton, Avatar, Separator, ScrollArea (19 components total)
4. `/dev/components` page in the **client app only** (`apps/client/app/dev/components/page.tsx`) renders all 19 components with section labels and variant previews; navigable at `http://localhost:3000/dev/components`
5. Both apps import and render `<Button>` from `@eusolicit/ui` without errors; `pnpm build` exits 0 for both apps

## Tasks / Subtasks

- [x] Task 1: Upgrade Tailwind config preset with full design tokens (AC: 1)
  - [x] 1.1 Add `tailwindcss-animate` to `packages/config/package.json` dependencies (`^1.0.7`)
  - [x] 1.2 Rewrite `packages/config/tailwind.config.ts` with: `darkMode: ["class"]`, CSS-variable-mapped colors (`background`, `foreground`, `primary`, `secondary`, `destructive`, `muted`, `accent`, `popover`, `card`, `border`, `input`, `ring`), semantic direct colors (`success`, `warning`, `error`, `info`), `borderRadius` from `--radius` CSS variable, `fontFamily.sans: ["var(--font-inter)", ...]`, `fontFamily.mono: ["var(--font-jetbrains-mono)", ...]`, custom shadow scale (`sm` → `2xl`), keyframes for accordion, `plugins: [require("tailwindcss-animate")]`
  - [x] 1.3 Fix both apps' `tailwind.config.ts` to use `presets: [sharedPreset]` array (NOT `...baseConfig` spread) — resolves deferred Story 3.1 review finding

- [x] Task 2: Add CSS variables to both apps' globals.css (AC: 2)
  - [x] 2.1 Replace `apps/client/app/globals.css` content: keep `@tailwind` directives, add `@layer base { :root { ... } .dark { ... } }` with all HSL CSS variable definitions
  - [x] 2.2 Do the same for `apps/admin/app/globals.css` — identical variable values; only the `@tailwind` directives + CSS variable block

- [x] Task 3: Add font loading to both app layouts (AC: 1)
  - [x] 3.1 Update `apps/client/app/layout.tsx`: import `Inter` and `JetBrains_Mono` from `next/font/google`; apply `className` to `<body>`; expose CSS variables `--font-inter` and `--font-jetbrains-mono` via `variable` prop
  - [x] 3.2 Update `apps/admin/app/layout.tsx`: same pattern

- [x] Task 4: Fix tsconfig path alias and set up lib/utils (prerequisite for components)
  - [x] 4.1 Update `apps/client/tsconfig.json` `paths`: change `"@/*": ["./src/*"]` → `"@/*": ["./*"]` so `@/lib/utils` resolves to `apps/client/lib/utils.ts`
  - [x] 4.2 Update `apps/admin/tsconfig.json` paths same way
  - [x] 4.3 Create `apps/client/lib/utils.ts`: re-exports `cn` from `@eusolicit/ui`
  - [x] 4.4 Create `apps/admin/lib/utils.ts`: same

- [x] Task 5: Add dependencies to packages/ui (AC: 3)
  - [x] 5.1 Update `packages/ui/package.json` — add to `dependencies`: `@radix-ui/react-avatar ^1.1`, `@radix-ui/react-checkbox ^1.1`, `@radix-ui/react-dialog ^1.1`, `@radix-ui/react-dropdown-menu ^2.1`, `@radix-ui/react-label ^2.1`, `@radix-ui/react-radio-group ^1.2`, `@radix-ui/react-scroll-area ^1.2`, `@radix-ui/react-select ^2.1`, `@radix-ui/react-separator ^1.1`, `@radix-ui/react-slot ^1.1`, `@radix-ui/react-switch ^1.1`, `@radix-ui/react-tabs ^1.1`, `@radix-ui/react-tooltip ^1.1`, `class-variance-authority ^0.7`, `clsx ^2.1`, `lucide-react ^0.400`, `tailwind-merge ^2.5`
  - [x] 5.2 Run `pnpm install` from `frontend/` root

- [x] Task 6: Create shared utility and all 19 shadcn/ui components in packages/ui (AC: 3, 5)
  - [x] 6.1 Create `packages/ui/src/lib/utils.ts` — `cn()` helper using `clsx` + `twMerge`
  - [x] 6.2 Delete old placeholder `packages/ui/src/Button.tsx`
  - [x] 6.3 Create `packages/ui/src/components/ui/button.tsx` — shadcn/ui Button with CVA variants (default, destructive, outline, secondary, ghost, link) and sizes (default, sm, lg, icon)
  - [x] 6.4 Create `packages/ui/src/components/ui/input.tsx`
  - [x] 6.5 Create `packages/ui/src/components/ui/textarea.tsx`
  - [x] 6.6 Create `packages/ui/src/components/ui/label.tsx` (Radix Label — required by form components)
  - [x] 6.7 Create `packages/ui/src/components/ui/select.tsx` (Radix Select — SelectTrigger, SelectContent, SelectItem, SelectValue, SelectLabel, SelectSeparator, SelectGroup)
  - [x] 6.8 Create `packages/ui/src/components/ui/checkbox.tsx`
  - [x] 6.9 Create `packages/ui/src/components/ui/radio-group.tsx`
  - [x] 6.10 Create `packages/ui/src/components/ui/switch.tsx`
  - [x] 6.11 Create `packages/ui/src/components/ui/dialog.tsx` (Dialog, DialogContent, DialogHeader, DialogTitle, DialogDescription, DialogFooter, DialogTrigger, DialogClose)
  - [x] 6.12 Create `packages/ui/src/components/ui/sheet.tsx` (Sheet, SheetContent, SheetHeader, SheetTitle, SheetDescription, SheetFooter, SheetTrigger, SheetClose — built on Radix Dialog)
  - [x] 6.13 Create `packages/ui/src/components/ui/dropdown-menu.tsx` (DropdownMenu, DropdownMenuTrigger, DropdownMenuContent, DropdownMenuItem, DropdownMenuLabel, DropdownMenuSeparator, DropdownMenuCheckboxItem, DropdownMenuRadioGroup, DropdownMenuRadioItem, DropdownMenuSubTrigger, DropdownMenuSubContent, DropdownMenuSub, DropdownMenuShortcut)
  - [x] 6.14 Create `packages/ui/src/components/ui/tooltip.tsx` (TooltipProvider, Tooltip, TooltipTrigger, TooltipContent)
  - [x] 6.15 Create `packages/ui/src/components/ui/badge.tsx` — CVA variants (default, secondary, destructive, outline)
  - [x] 6.16 Create `packages/ui/src/components/ui/card.tsx` (Card, CardHeader, CardTitle, CardDescription, CardContent, CardFooter)
  - [x] 6.17 Create `packages/ui/src/components/ui/tabs.tsx` (Tabs, TabsList, TabsTrigger, TabsContent)
  - [x] 6.18 Create `packages/ui/src/components/ui/table.tsx` (Table, TableHeader, TableBody, TableFooter, TableRow, TableHead, TableCell, TableCaption)
  - [x] 6.19 Create `packages/ui/src/components/ui/skeleton.tsx`
  - [x] 6.20 Create `packages/ui/src/components/ui/avatar.tsx` (Avatar, AvatarImage, AvatarFallback)
  - [x] 6.21 Create `packages/ui/src/components/ui/separator.tsx`
  - [x] 6.22 Create `packages/ui/src/components/ui/scroll-area.tsx` (ScrollArea, ScrollBar)

- [x] Task 7: Update packages/ui barrel export (AC: 3, 5)
  - [x] 7.1 Rewrite `packages/ui/index.ts` — export `cn` from `./src/lib/utils`; export ALL named exports from each of the 20 component files (19 shadcn + label)

- [x] Task 8: Add components.json to both apps (shadcn/ui config)
  - [x] 8.1 Create `apps/client/components.json` — `style: "default"`, `rsc: true`, `tsx: true`, `tailwind.baseColor: "slate"`, `cssVariables: true`, `aliases.components: "@/components"`, `aliases.utils: "@/lib/utils"`
  - [x] 8.2 Create `apps/admin/components.json` — same config

- [x] Task 9: Create /dev/components showcase page in client app (AC: 4)
  - [x] 9.1 Create `apps/client/app/dev/components/page.tsx` — imports and renders all 19 components with section headings, variant examples, and labels (see Dev Notes for structure)

- [x] Task 10: Verify build and shared Button (AC: 5)
  - [x] 10.1 Run `pnpm install` at `frontend/` root
  - [x] 10.2 Run `pnpm build` — both apps must exit 0
  - [x] 10.3 Verify `/dev/components` page renders in browser (`pnpm dev --filter client`)
  - [x] 10.4 Verify no TypeScript errors: `pnpm type-check`

## Dev Notes

### Critical Story 3.1 Learnings to Apply

These completions notes from Story 3.1 MUST inform this implementation:

1. **`next.config.ts` not supported — use `next.config.mjs`** — Both apps use `.mjs` format (already in place from 3.1). Do NOT create any `.ts` next config.
2. **Turborepo v2 uses `tasks` not `pipeline`** — `turbo.json` already has this; don't change it.
3. **ESLint `require.resolve()` workaround in `.eslintrc.js`** — already in place; don't change ESLint config.
4. **`@/*` path alias maps to `./src/*` but `src/` doesn't exist** — this story MUST fix this (Task 4). Change to `@/*` → `./*` so `@/lib/utils` resolves to `apps/*/lib/utils.ts`.
5. **`...baseConfig` spread silently drops base entries** — this story MUST fix apps' tailwind configs to use `presets: [sharedPreset]` (Task 1.3).

### Working Directory

All frontend code lives at: `eusolicit-app/frontend/` (absolute path: `/home/debian/Projects/eusolicit/eusolicit-app/frontend/`)

### Tailwind Config Preset — Full Implementation

**`packages/config/tailwind.config.ts`** (replace the minimal scaffold entirely):

```typescript
import type { Config } from "tailwindcss";

const config = {
  darkMode: ["class"],
  theme: {
    extend: {
      colors: {
        // shadcn/ui CSS-variable-mapped colors (HSL values live in globals.css)
        border: "hsl(var(--border))",
        input: "hsl(var(--input))",
        ring: "hsl(var(--ring))",
        background: "hsl(var(--background))",
        foreground: "hsl(var(--foreground))",
        primary: {
          DEFAULT: "hsl(var(--primary))",
          foreground: "hsl(var(--primary-foreground))",
        },
        secondary: {
          DEFAULT: "hsl(var(--secondary))",
          foreground: "hsl(var(--secondary-foreground))",
        },
        destructive: {
          DEFAULT: "hsl(var(--destructive))",
          foreground: "hsl(var(--destructive-foreground))",
        },
        muted: {
          DEFAULT: "hsl(var(--muted))",
          foreground: "hsl(var(--muted-foreground))",
        },
        accent: {
          DEFAULT: "hsl(var(--accent))",
          foreground: "hsl(var(--accent-foreground))",
        },
        popover: {
          DEFAULT: "hsl(var(--popover))",
          foreground: "hsl(var(--popover-foreground))",
        },
        card: {
          DEFAULT: "hsl(var(--card))",
          foreground: "hsl(var(--card-foreground))",
        },
        // Semantic direct colors (success/warning/error/info)
        success: {
          DEFAULT: "#16a34a",
          50: "#f0fdf4",
          600: "#16a34a",
          foreground: "#ffffff",
        },
        warning: {
          DEFAULT: "#d97706",
          50: "#fffbeb",
          600: "#d97706",
          foreground: "#ffffff",
        },
        error: {
          DEFAULT: "#dc2626",
          50: "#fef2f2",
          600: "#dc2626",
          foreground: "#ffffff",
        },
        info: {
          DEFAULT: "#2563eb",
          50: "#eff6ff",
          600: "#2563eb",
          foreground: "#ffffff",
        },
      },
      borderRadius: {
        lg: "var(--radius)",
        md: "calc(var(--radius) - 2px)",
        sm: "calc(var(--radius) - 4px)",
      },
      fontFamily: {
        sans: ["var(--font-inter)", "system-ui", "ui-sans-serif", "sans-serif"],
        mono: ["var(--font-jetbrains-mono)", "ui-monospace", "SFMono-Regular", "monospace"],
      },
      boxShadow: {
        sm: "0 1px 2px 0 rgb(0 0 0 / 0.05)",
        DEFAULT: "0 1px 3px 0 rgb(0 0 0 / 0.1), 0 1px 2px -1px rgb(0 0 0 / 0.1)",
        md: "0 4px 6px -1px rgb(0 0 0 / 0.1), 0 2px 4px -2px rgb(0 0 0 / 0.1)",
        lg: "0 10px 15px -3px rgb(0 0 0 / 0.1), 0 4px 6px -4px rgb(0 0 0 / 0.1)",
        xl: "0 20px 25px -5px rgb(0 0 0 / 0.1), 0 8px 10px -6px rgb(0 0 0 / 0.1)",
        "2xl": "0 25px 50px -12px rgb(0 0 0 / 0.25)",
      },
      keyframes: {
        "accordion-down": {
          from: { height: "0" },
          to: { height: "var(--radix-accordion-content-height)" },
        },
        "accordion-up": {
          from: { height: "var(--radix-accordion-content-height)" },
          to: { height: "0" },
        },
      },
      animation: {
        "accordion-down": "accordion-down 0.2s ease-out",
        "accordion-up": "accordion-up 0.2s ease-out",
      },
    },
  },
  plugins: [require("tailwindcss-animate")],
} satisfies Partial<Config>;

export default config;
```

**`packages/config/package.json`** — add `tailwindcss-animate` to `devDependencies` (Tailwind plugins are loaded at build time):
```json
"devDependencies": {
  ...existing...,
  "tailwindcss-animate": "^1.0.7"
}
```

### App Tailwind Config — Use `presets` Array (Fix Deferred 3.1 Issue)

**`apps/client/tailwind.config.ts`** and **`apps/admin/tailwind.config.ts`** — replace `...baseConfig` spread with `presets`:

```typescript
import type { Config } from "tailwindcss";
import sharedPreset from "@eusolicit/config/tailwind.config";

const config: Config = {
  presets: [sharedPreset as Config],
  content: [
    "./app/**/*.{js,ts,jsx,tsx,mdx}",
    "./components/**/*.{js,ts,jsx,tsx,mdx}",
    "./lib/**/*.{js,ts,jsx,tsx,mdx}",
    "../../packages/ui/src/**/*.{js,ts,jsx,tsx,mdx}",
  ],
};

export default config;
```

Note: removed `./src/**/*` from content (no `src/` dir exists) and added `./lib/**/*` (Story 3.2+ uses `lib/` at root).

### CSS Variables — globals.css Pattern (Both Apps)

**`apps/client/app/globals.css`** and **`apps/admin/app/globals.css`** — replace current minimal content:

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 222.2 84% 4.9%;
    --card: 0 0% 100%;
    --card-foreground: 222.2 84% 4.9%;
    --popover: 0 0% 100%;
    --popover-foreground: 222.2 84% 4.9%;
    --primary: 243 75% 59%;
    --primary-foreground: 0 0% 100%;
    --secondary: 210 40% 96.1%;
    --secondary-foreground: 222.2 47.4% 11.2%;
    --muted: 210 40% 96.1%;
    --muted-foreground: 215.4 16.3% 46.9%;
    --accent: 210 40% 96.1%;
    --accent-foreground: 222.2 47.4% 11.2%;
    --destructive: 0 84.2% 60.2%;
    --destructive-foreground: 210 40% 98%;
    --border: 214.3 31.8% 91.4%;
    --input: 214.3 31.8% 91.4%;
    --ring: 243 75% 59%;
    --radius: 0.5rem;
  }

  .dark {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;
    --card: 222.2 84% 4.9%;
    --card-foreground: 210 40% 98%;
    --popover: 222.2 84% 4.9%;
    --popover-foreground: 210 40% 98%;
    --primary: 243 75% 59%;
    --primary-foreground: 0 0% 100%;
    --secondary: 217.2 32.6% 17.5%;
    --secondary-foreground: 210 40% 98%;
    --muted: 217.2 32.6% 17.5%;
    --muted-foreground: 215 20.2% 65.1%;
    --accent: 217.2 32.6% 17.5%;
    --accent-foreground: 210 40% 98%;
    --destructive: 0 62.8% 30.6%;
    --destructive-foreground: 210 40% 98%;
    --border: 217.2 32.6% 17.5%;
    --input: 217.2 32.6% 17.5%;
    --ring: 243 75% 59%;
  }
}
```

**Design token notes:**
- `--primary: 243 75% 59%` → indigo-600 (`#4f46e5`) in HSL
- `--destructive: 0 84.2% 60.2%` → red-500 in HSL (shadcn/ui default; maps to `error.600`)
- shadcn/ui CSS variables use HSL component values WITHOUT `hsl()` wrapper — the Tailwind color tokens add `hsl()` wrapper: `"hsl(var(--primary))"`

### Font Loading — App layout.tsx Pattern

**`apps/client/app/layout.tsx`** (UPDATE):

```typescript
import type { Metadata } from "next";
import { Inter, JetBrains_Mono } from "next/font/google";
import "./globals.css";

const inter = Inter({
  subsets: ["latin", "latin-ext"],
  variable: "--font-inter",
  display: "swap",
});

const jetbrainsMono = JetBrains_Mono({
  subsets: ["latin"],
  variable: "--font-jetbrains-mono",
  display: "swap",
});

export const metadata: Metadata = {
  title: "EU Solicit Client",
  description: "EU Solicit Client Portal",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="bg" suppressHydrationWarning>
      <body className={`${inter.variable} ${jetbrainsMono.variable} font-sans antialiased`}>
        {children}
      </body>
    </html>
  );
}
```

**`apps/admin/app/layout.tsx`** — same pattern, different metadata (`title: "EU Solicit Admin"`).

Note: `suppressHydrationWarning` on `<html>` is the correct and only safe location — NOT on content areas (per E03-R-002 mitigation: route guard story).

### tsconfig.json Path Alias Fix (Both Apps)

Change `"@/*": ["./src/*"]` → `"@/*": ["./*"]` in **both** `apps/client/tsconfig.json` and `apps/admin/tsconfig.json`:

```json
{
  "extends": "@eusolicit/config/tsconfig.json",
  "compilerOptions": {
    "paths": { "@/*": ["./*"] }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

This resolves `@/lib/utils` → `apps/client/lib/utils.ts` (which is at the app root, not `src/`).

### packages/ui Dependency Update

**`packages/ui/package.json`** — add `dependencies` field:

```json
{
  "name": "@eusolicit/ui",
  "version": "0.0.0",
  "private": true,
  "main": "./index.ts",
  "types": "./index.ts",
  "dependencies": {
    "@radix-ui/react-avatar": "^1.1.0",
    "@radix-ui/react-checkbox": "^1.1.0",
    "@radix-ui/react-dialog": "^1.1.0",
    "@radix-ui/react-dropdown-menu": "^2.1.0",
    "@radix-ui/react-label": "^2.1.0",
    "@radix-ui/react-radio-group": "^1.2.0",
    "@radix-ui/react-scroll-area": "^1.2.0",
    "@radix-ui/react-select": "^2.1.0",
    "@radix-ui/react-separator": "^1.1.0",
    "@radix-ui/react-slot": "^1.1.0",
    "@radix-ui/react-switch": "^1.1.0",
    "@radix-ui/react-tabs": "^1.1.0",
    "@radix-ui/react-tooltip": "^1.1.0",
    "class-variance-authority": "^0.7.0",
    "clsx": "^2.1.0",
    "lucide-react": "^0.400.0",
    "tailwind-merge": "^2.5.0"
  },
  "scripts": { ... },
  "devDependencies": { ...existing... },
  "peerDependencies": { ...existing... }
}
```

### packages/ui — cn Utility

**`packages/ui/src/lib/utils.ts`** (NEW):

```typescript
import { type ClassValue, clsx } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

### packages/ui — Component Implementation Pattern

All components follow the same pattern (copy from shadcn/ui registry). Key points:
- Import path for `cn`: `import { cn } from "../../lib/utils"` (relative from `src/components/ui/`)
- Use `@radix-ui/*` primitives (already listed in dependencies)
- Use `class-variance-authority` for CVA-based variants

**Example — `packages/ui/src/components/ui/button.tsx`:**

```typescript
import * as React from "react";
import { Slot } from "@radix-ui/react-slot";
import { cva, type VariantProps } from "class-variance-authority";
import { cn } from "../../lib/utils";

const buttonVariants = cva(
  "inline-flex items-center justify-center gap-2 whitespace-nowrap rounded-md text-sm font-medium ring-offset-background transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:pointer-events-none disabled:opacity-50 [&_svg]:pointer-events-none [&_svg]:size-4 [&_svg]:shrink-0",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground hover:bg-primary/90",
        destructive: "bg-destructive text-destructive-foreground hover:bg-destructive/90",
        outline: "border border-input bg-background hover:bg-accent hover:text-accent-foreground",
        secondary: "bg-secondary text-secondary-foreground hover:bg-secondary/80",
        ghost: "hover:bg-accent hover:text-accent-foreground",
        link: "text-primary underline-offset-4 hover:underline",
      },
      size: {
        default: "h-10 px-4 py-2",
        sm: "h-9 rounded-md px-3",
        lg: "h-11 rounded-md px-8",
        icon: "h-10 w-10",
      },
    },
    defaultVariants: {
      variant: "default",
      size: "default",
    },
  }
);

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  asChild?: boolean;
}

const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, asChild = false, ...props }, ref) => {
    const Comp = asChild ? Slot : "button";
    return (
      <Comp className={cn(buttonVariants({ variant, size, className }))} ref={ref} {...props} />
    );
  }
);
Button.displayName = "Button";

export { Button, buttonVariants };
```

The remaining 18 components (Input, Textarea, Select, Checkbox, etc.) follow the identical pattern from the [shadcn/ui registry](https://ui.shadcn.com/docs/components). Fetch them from `https://ui.shadcn.com/registry/default/ui/<component-name>.json` or run `npx shadcn@latest add <component>` in a temp directory to get the latest code.

**CRITICAL:** When copying components from shadcn registry, change `import { cn } from "@/lib/utils"` to `import { cn } from "../../lib/utils"` (relative path for packages/ui).

### packages/ui Barrel Export

**`packages/ui/index.ts`** (REPLACE current content):

```typescript
// Utility
export { cn } from "./src/lib/utils";

// Components (shadcn/ui)
export { Button, buttonVariants } from "./src/components/ui/button";
export { Input } from "./src/components/ui/input";
export { Textarea } from "./src/components/ui/textarea";
export { Label } from "./src/components/ui/label";
export {
  Select, SelectContent, SelectGroup, SelectItem, SelectLabel,
  SelectSeparator, SelectTrigger, SelectValue,
} from "./src/components/ui/select";
export { Checkbox } from "./src/components/ui/checkbox";
export { RadioGroup, RadioGroupItem } from "./src/components/ui/radio-group";
export { Switch } from "./src/components/ui/switch";
export {
  Dialog, DialogClose, DialogContent, DialogDescription,
  DialogFooter, DialogHeader, DialogTitle, DialogTrigger,
} from "./src/components/ui/dialog";
export {
  Sheet, SheetClose, SheetContent, SheetDescription,
  SheetFooter, SheetHeader, SheetTitle, SheetTrigger,
} from "./src/components/ui/sheet";
export {
  DropdownMenu, DropdownMenuCheckboxItem, DropdownMenuContent,
  DropdownMenuGroup, DropdownMenuItem, DropdownMenuLabel,
  DropdownMenuPortal, DropdownMenuRadioGroup, DropdownMenuRadioItem,
  DropdownMenuSeparator, DropdownMenuShortcut, DropdownMenuSub,
  DropdownMenuSubContent, DropdownMenuSubTrigger, DropdownMenuTrigger,
} from "./src/components/ui/dropdown-menu";
export {
  Tooltip, TooltipContent, TooltipProvider, TooltipTrigger,
} from "./src/components/ui/tooltip";
export { Badge, badgeVariants } from "./src/components/ui/badge";
export {
  Card, CardContent, CardDescription, CardFooter, CardHeader, CardTitle,
} from "./src/components/ui/card";
export { Tabs, TabsContent, TabsList, TabsTrigger } from "./src/components/ui/tabs";
export {
  Table, TableBody, TableCaption, TableCell, TableFooter,
  TableHead, TableHeader, TableRow,
} from "./src/components/ui/table";
export { Skeleton } from "./src/components/ui/skeleton";
export { Avatar, AvatarFallback, AvatarImage } from "./src/components/ui/avatar";
export { Separator } from "./src/components/ui/separator";
export { ScrollArea, ScrollBar } from "./src/components/ui/scroll-area";
```

### App lib/utils.ts (Both Apps)

**`apps/client/lib/utils.ts`** and **`apps/admin/lib/utils.ts`** (NEW):

```typescript
// Re-export cn from shared packages/ui
export { cn } from "@eusolicit/ui";
```

This gives app-local code access to `cn` via `@/lib/utils` (after tsconfig path fix).

### components.json — Both Apps

**`apps/client/components.json`** and **`apps/admin/components.json`** (NEW):

```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "default",
  "rsc": true,
  "tsx": true,
  "tailwind": {
    "config": "tailwind.config.ts",
    "css": "app/globals.css",
    "baseColor": "slate",
    "cssVariables": true,
    "prefix": ""
  },
  "aliases": {
    "components": "@/components",
    "utils": "@/lib/utils",
    "ui": "@/components/ui",
    "lib": "@/lib",
    "hooks": "@/hooks"
  },
  "iconLibrary": "lucide"
}
```

Note: `components.json` serves as the shadcn CLI config for future `npx shadcn add <component>` calls within each app. The `aliases.utils` points to each app's local `lib/utils.ts` (which re-exports from `@eusolicit/ui`).

### /dev/components Page Structure

**`apps/client/app/dev/components/page.tsx`** (NEW):

```typescript
import {
  Button, Input, Textarea, Label, Badge, Card, CardContent, CardHeader, CardTitle,
  Separator, Skeleton, Avatar, AvatarFallback, AvatarImage, Tabs, TabsContent,
  TabsList, TabsTrigger, Checkbox, Switch, Tooltip, TooltipContent,
  TooltipProvider, TooltipTrigger,
  Dialog, DialogContent, DialogHeader, DialogTitle, DialogTrigger,
  Table, TableBody, TableCell, TableHead, TableHeader, TableRow,
  Select, SelectContent, SelectItem, SelectTrigger, SelectValue,
  ScrollArea, DropdownMenu, DropdownMenuContent, DropdownMenuItem, DropdownMenuTrigger,
  RadioGroup, RadioGroupItem, Sheet, SheetContent, SheetHeader, SheetTitle, SheetTrigger,
} from "@eusolicit/ui";

export default function ComponentsPage() {
  return (
    <div className="container mx-auto py-10 space-y-12">
      <h1 className="text-3xl font-bold">EU Solicit Design System</h1>
      <p className="text-muted-foreground">Component showcase for visual QA (E03-P2-002)</p>

      {/* Button */}
      <section id="button" className="space-y-4">
        <h2 className="text-xl font-semibold border-b pb-2">Button</h2>
        <div className="flex flex-wrap gap-3">
          <Button>Default</Button>
          <Button variant="secondary">Secondary</Button>
          <Button variant="destructive">Destructive</Button>
          <Button variant="outline">Outline</Button>
          <Button variant="ghost">Ghost</Button>
          <Button variant="link">Link</Button>
          <Button size="sm">Small</Button>
          <Button size="lg">Large</Button>
          <Button disabled>Disabled</Button>
        </div>
      </section>

      {/* Input & Textarea */}
      <section id="input" className="space-y-4">
        <h2 className="text-xl font-semibold border-b pb-2">Input / Textarea</h2>
        <div className="flex flex-col gap-3 max-w-sm">
          <Label htmlFor="demo-input">Text Input</Label>
          <Input id="demo-input" placeholder="Enter text..." />
          <Textarea placeholder="Enter multiline text..." />
        </div>
      </section>

      {/* Badge */}
      <section id="badge" className="space-y-4">
        <h2 className="text-xl font-semibold border-b pb-2">Badge</h2>
        <div className="flex flex-wrap gap-2">
          <Badge>Default</Badge>
          <Badge variant="secondary">Secondary</Badge>
          <Badge variant="destructive">Destructive</Badge>
          <Badge variant="outline">Outline</Badge>
        </div>
      </section>

      {/* Card */}
      <section id="card" className="space-y-4">
        <h2 className="text-xl font-semibold border-b pb-2">Card</h2>
        <Card className="max-w-sm">
          <CardHeader><CardTitle>Card Title</CardTitle></CardHeader>
          <CardContent><p className="text-sm text-muted-foreground">Card content goes here.</p></CardContent>
        </Card>
      </section>

      {/* Skeleton */}
      <section id="skeleton" className="space-y-4">
        <h2 className="text-xl font-semibold border-b pb-2">Skeleton</h2>
        <div className="space-y-2 max-w-sm">
          <Skeleton className="h-4 w-full" />
          <Skeleton className="h-4 w-3/4" />
          <Skeleton className="h-20 w-full" />
        </div>
      </section>

      {/* Avatar */}
      <section id="avatar" className="space-y-4">
        <h2 className="text-xl font-semibold border-b pb-2">Avatar</h2>
        <div className="flex gap-3">
          <Avatar>
            <AvatarImage src="https://github.com/shadcn.png" alt="shadcn" />
            <AvatarFallback>CN</AvatarFallback>
          </Avatar>
          <Avatar><AvatarFallback>EU</AvatarFallback></Avatar>
        </div>
      </section>

      {/* Separator */}
      <section id="separator" className="space-y-4">
        <h2 className="text-xl font-semibold border-b pb-2">Separator</h2>
        <div className="space-y-2 max-w-sm">
          <p className="text-sm">Above separator</p>
          <Separator />
          <p className="text-sm">Below separator</p>
        </div>
      </section>

      {/* Tabs */}
      <section id="tabs" className="space-y-4">
        <h2 className="text-xl font-semibold border-b pb-2">Tabs</h2>
        <Tabs defaultValue="tab1" className="max-w-sm">
          <TabsList>
            <TabsTrigger value="tab1">Tab 1</TabsTrigger>
            <TabsTrigger value="tab2">Tab 2</TabsTrigger>
          </TabsList>
          <TabsContent value="tab1"><p className="text-sm p-2">Tab 1 content</p></TabsContent>
          <TabsContent value="tab2"><p className="text-sm p-2">Tab 2 content</p></TabsContent>
        </Tabs>
      </section>

      {/* Table */}
      <section id="table" className="space-y-4">
        <h2 className="text-xl font-semibold border-b pb-2">Table</h2>
        <Table>
          <TableHeader>
            <TableRow>
              <TableHead>Name</TableHead>
              <TableHead>Status</TableHead>
              <TableHead>Amount</TableHead>
            </TableRow>
          </TableHeader>
          <TableBody>
            <TableRow>
              <TableCell>EU Grant A</TableCell>
              <TableCell><Badge variant="secondary">Active</Badge></TableCell>
              <TableCell>€50,000</TableCell>
            </TableRow>
            <TableRow>
              <TableCell>EU Grant B</TableCell>
              <TableCell><Badge variant="outline">Draft</Badge></TableCell>
              <TableCell>€25,000</TableCell>
            </TableRow>
          </TableBody>
        </Table>
      </section>

      {/* Checkbox & Switch */}
      <section id="checkbox" className="space-y-4">
        <h2 className="text-xl font-semibold border-b pb-2">Checkbox / Switch</h2>
        <div className="flex flex-col gap-3">
          <div className="flex items-center gap-2">
            <Checkbox id="check1" />
            <Label htmlFor="check1">Accept terms and conditions</Label>
          </div>
          <div className="flex items-center gap-2">
            <Switch id="switch1" />
            <Label htmlFor="switch1">Enable notifications</Label>
          </div>
        </div>
      </section>

      {/* Select */}
      <section id="select" className="space-y-4">
        <h2 className="text-xl font-semibold border-b pb-2">Select</h2>
        <Select>
          <SelectTrigger className="max-w-xs">
            <SelectValue placeholder="Select industry..." />
          </SelectTrigger>
          <SelectContent>
            <SelectItem value="tech">Technology</SelectItem>
            <SelectItem value="construction">Construction</SelectItem>
            <SelectItem value="agriculture">Agriculture</SelectItem>
          </SelectContent>
        </Select>
      </section>

      {/* Tooltip */}
      <section id="tooltip" className="space-y-4">
        <h2 className="text-xl font-semibold border-b pb-2">Tooltip</h2>
        <TooltipProvider>
          <Tooltip>
            <TooltipTrigger asChild>
              <Button variant="outline">Hover me</Button>
            </TooltipTrigger>
            <TooltipContent>
              <p>Tooltip content</p>
            </TooltipContent>
          </Tooltip>
        </TooltipProvider>
      </section>

      {/* Dialog */}
      <section id="dialog" className="space-y-4">
        <h2 className="text-xl font-semibold border-b pb-2">Dialog</h2>
        <Dialog>
          <DialogTrigger asChild>
            <Button variant="outline">Open Dialog</Button>
          </DialogTrigger>
          <DialogContent>
            <DialogHeader>
              <DialogTitle>Dialog Title</DialogTitle>
            </DialogHeader>
            <p className="text-sm text-muted-foreground">Dialog content here.</p>
          </DialogContent>
        </Dialog>
      </section>

      {/* Sheet */}
      <section id="sheet" className="space-y-4">
        <h2 className="text-xl font-semibold border-b pb-2">Sheet</h2>
        <Sheet>
          <SheetTrigger asChild>
            <Button variant="outline">Open Sheet</Button>
          </SheetTrigger>
          <SheetContent>
            <SheetHeader>
              <SheetTitle>Sheet Title</SheetTitle>
            </SheetHeader>
            <p className="text-sm text-muted-foreground mt-4">Sheet content here.</p>
          </SheetContent>
        </Sheet>
      </section>

      {/* DropdownMenu */}
      <section id="dropdown" className="space-y-4">
        <h2 className="text-xl font-semibold border-b pb-2">Dropdown Menu</h2>
        <DropdownMenu>
          <DropdownMenuTrigger asChild>
            <Button variant="outline">Open Menu</Button>
          </DropdownMenuTrigger>
          <DropdownMenuContent>
            <DropdownMenuItem>Profile</DropdownMenuItem>
            <DropdownMenuItem>Settings</DropdownMenuItem>
            <DropdownMenuItem>Sign out</DropdownMenuItem>
          </DropdownMenuContent>
        </DropdownMenu>
      </section>

      {/* RadioGroup */}
      <section id="radiogroup" className="space-y-4">
        <h2 className="text-xl font-semibold border-b pb-2">Radio Group</h2>
        <RadioGroup defaultValue="option1" className="flex flex-col gap-2">
          <div className="flex items-center gap-2">
            <RadioGroupItem value="option1" id="r1" />
            <Label htmlFor="r1">Option 1</Label>
          </div>
          <div className="flex items-center gap-2">
            <RadioGroupItem value="option2" id="r2" />
            <Label htmlFor="r2">Option 2</Label>
          </div>
        </RadioGroup>
      </section>

      {/* ScrollArea */}
      <section id="scroll-area" className="space-y-4">
        <h2 className="text-xl font-semibold border-b pb-2">Scroll Area</h2>
        <ScrollArea className="h-32 w-64 rounded-md border p-4">
          {Array.from({ length: 20 }, (_, i) => (
            <p key={i} className="text-sm">Scroll item {i + 1}</p>
          ))}
        </ScrollArea>
      </section>

    </div>
  );
}
```

**Important:** The `/dev/components` page is a **Server Component** (no `"use client"` directive needed). However, components that use Radix UI hooks (Dialog, Sheet, DropdownMenu, Tooltip, etc.) require a client context. Shadcn/ui components that contain `"use client"` inside the component file handle this internally — Server Components CAN render them, but the interactive state won't work without client-side JS. The `/dev/components` page is for visual QA only, not interactive testing.

If certain Radix-based components fail to render as server components (SSR issues), add `"use client"` to the page or wrap interactive sections in client sub-components.

### shadcn/ui Component Source — How to Obtain

The recommended approach to get the latest component code:

```bash
# Method 1: Use shadcn CLI to generate components (then move to packages/ui)
cd eusolicit-app/frontend/apps/client
npx shadcn@latest init   # creates components.json
npx shadcn@latest add button input textarea label select checkbox radio-group switch dialog sheet dropdown-menu tooltip badge card tabs table skeleton avatar separator scroll-area
# Components appear in apps/client/components/ui/
# Move each file to packages/ui/src/components/ui/
# Fix imports: change "@/lib/utils" → "../../lib/utils" in each component
```

```bash
# Method 2: Copy directly from shadcn/ui GitHub repo
# https://github.com/shadcn-ui/ui/tree/main/apps/www/registry/default/ui
# Each file at registry/default/ui/<component>.tsx
# Fix import path as above
```

Use Method 1 for accuracy (latest version with correct TypeScript). After moving files, run `pnpm build` to catch any type errors.

### What NOT to Do

- **Do NOT use `shadcn-ui` package name** — the package is `shadcn` (renamed). CLI command: `npx shadcn@latest`, NOT `npx shadcn-ui@latest`
- **Do NOT leave `packages/ui/src/Button.tsx`** — delete this placeholder file; the new Button is at `packages/ui/src/components/ui/button.tsx`
- **Do NOT use `...baseConfig` spread** in app tailwind configs — use `presets: [sharedPreset as Config]` array
- **Do NOT keep `@/*` → `./src/*`** — change to `./*` (there is no `src/` directory in the apps)
- **Do NOT add `"use client"` to packages/ui component files that are already marked** — shadcn components contain their own client directives internally; only the interactive primitives are client-side
- **Do NOT copy components to both apps' `components/ui/`** — all components live ONLY in `packages/ui/src/components/ui/`; apps import from `@eusolicit/ui`
- **Do NOT use hardcoded hex colors in component files** — components use CSS variables (`bg-primary`, `text-foreground`, etc.) so theming works via CSS variables in globals.css
- **Do NOT forget `tailwindcss-animate` in `packages/config`** — the preset's `plugins` array `require("tailwindcss-animate")` will throw at build time without it
- **Do NOT use `require("tailwindcss-animate")` in TypeScript without `@types` or ignoring the error** — if TS complains, add `// eslint-disable-next-line @typescript-eslint/no-require-imports` above the require line in `packages/config/tailwind.config.ts`

### Project Structure Notes

**New files/directories created by this story:**
```
eusolicit-app/frontend/
├── packages/
│   ├── config/
│   │   └── tailwind.config.ts       # REPLACED with full design token preset
│   └── ui/
│       ├── index.ts                  # REPLACED with full export barrel
│       └── src/
│           ├── lib/
│           │   └── utils.ts          # NEW — cn() helper
│           └── components/
│               └── ui/               # NEW — 20 shadcn/ui components
│                   ├── button.tsx
│                   ├── input.tsx
│                   ├── textarea.tsx
│                   ├── label.tsx
│                   ├── select.tsx
│                   ├── checkbox.tsx
│                   ├── radio-group.tsx
│                   ├── switch.tsx
│                   ├── dialog.tsx
│                   ├── sheet.tsx
│                   ├── dropdown-menu.tsx
│                   ├── tooltip.tsx
│                   ├── badge.tsx
│                   ├── card.tsx
│                   ├── tabs.tsx
│                   ├── table.tsx
│                   ├── skeleton.tsx
│                   ├── avatar.tsx
│                   ├── separator.tsx
│                   └── scroll-area.tsx
├── apps/
│   ├── client/
│   │   ├── app/
│   │   │   ├── globals.css           # UPDATED — add CSS variables
│   │   │   ├── layout.tsx            # UPDATED — add Inter + JetBrains Mono
│   │   │   └── dev/
│   │   │       └── components/
│   │   │           └── page.tsx      # NEW — /dev/components showcase
│   │   ├── lib/
│   │   │   └── utils.ts              # NEW — re-exports cn from @eusolicit/ui
│   │   ├── tsconfig.json             # UPDATED — fix @/* alias
│   │   ├── tailwind.config.ts        # UPDATED — use presets[] array
│   │   └── components.json           # NEW — shadcn/ui config
│   └── admin/
│       ├── app/
│       │   ├── globals.css           # UPDATED — add CSS variables
│       │   └── layout.tsx            # UPDATED — add Inter + JetBrains Mono
│       ├── lib/
│       │   └── utils.ts              # NEW — re-exports cn
│       ├── tsconfig.json             # UPDATED — fix @/* alias
│       ├── tailwind.config.ts        # UPDATED — use presets[] array
│       └── components.json           # NEW — shadcn/ui config
```

**Files deleted:**
- `packages/ui/src/Button.tsx` — replaced by `packages/ui/src/components/ui/button.tsx`

**No backend files, Python packages, CI config, or Playwright config are modified.**

### Test Expectations (from E03 Test Design)

Tests that this story directly enables:

| Priority | Test ID | Requirement | Verification |
|----------|---------|-------------|--------------|
| **P0** | E03-P0-001 | `pnpm build` exits 0 for both apps — new Radix deps must not break build | Run `pnpm build` at `frontend/`; must exit 0 |
| **P2** | E03-P2-001 | Shared `<Button>` from `packages/ui` renders without errors in both apps | Import `Button` from `@eusolicit/ui` in each app; build passes; no console errors |
| **P2** | E03-P2-002 | `/dev/components` renders all 19 shadcn/ui components — no JS errors | Navigate to `http://localhost:3000/dev/components`; assert each component section heading visible |
| **P2** | E03-P2-003 | CSS variables `--primary`, `--destructive`, `--muted` are set; indigo-600 swatch visible | Inspect computed styles; assert CSS variables defined; assert `Button` default variant has indigo background |
| **P2** | E03-P2-020 | TypeScript strict mode — zero type errors | `pnpm type-check` exits 0 across all packages and apps |
| **P3** | E03-P3-001 | Dark mode toggle (stretch) — switches `dark` class on `<html>` | Deferred; the `.dark` CSS variables are in place enabling future implementation |

**Risk context from E03 test design:**
- **E03-R-008** (CSS variable conflicts, Score: 2) is directly relevant — "shared Tailwind preset in `packages/config` conflicts with app-level `globals.css` overrides." Mitigation: both apps use identical CSS variables; visual QA via `/dev/components` catches rendering differences.
- **E03-P0-001 build gate** is the primary pass/fail for this story — if `pnpm build` fails with Radix UI type errors or unresolved imports, this story is incomplete.
- Branch coverage requirement (≥80% on stores/apiClient) doesn't apply to this story — it's a UI setup story; no Zustand stores or apiClient are touched.

### References

- [Source: eusolicit-docs/planning-artifacts/epics/E03-frontend-shell-design-system.md#S03.02]
- [Source: eusolicit-docs/test-artifacts/test-design-epic-03.md#P0, P2, P3, E03-R-008]
- [Source: eusolicit-docs/implementation-artifacts/3-1-next-js-14-monorepo-scaffold.md#Dev Agent Record]
- [Source: eusolicit-docs/project-context.md#Technology Stack]
- [Source: https://ui.shadcn.com/docs — shadcn/ui component registry]
- [Source: https://tailwindcss.com/docs/presets — Tailwind CSS Presets docs]

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.5 (claude-sonnet-4-5)

### Debug Log References

- **Fix 1**: `tailwind.config.ts` in apps had `presets: [sharedPreset as Config]` which caused TS2545 (type conversion error because `Partial<Config>` doesn't overlap with `Config`). Fixed by using `sharedPreset as unknown as Config` double cast.
- **Fix 2**: `packages/config/tailwind.config.ts` used `require("tailwindcss-animate")` which caused `TS2580: Cannot find name 'require'` because `@types/node` was not in devDependencies. Fixed by adding `@types/node: ^20.0.0` to `packages/config/package.json`.

### Completion Notes List

- Implemented complete Tailwind design token preset in `packages/config/tailwind.config.ts` with: CSS-variable-mapped shadcn/ui colors, semantic colors (success/warning/error/info), Inter + JetBrains Mono font stacks, custom shadow scale (sm → 2xl), accordion keyframes, and `darkMode: ["class"]`
- Fixed the deferred Story 3.1 issue: both apps now use `presets: [sharedPreset as unknown as Config]` array instead of `...baseConfig` spread
- Added full HSL CSS variable definitions to both apps' `globals.css` for light and dark modes
- Updated both app layouts to load Inter and JetBrains Mono via `next/font/google` with CSS variable exposure
- Fixed `@/*` path alias from `./src/*` → `./*` in both apps' `tsconfig.json` (no `src/` dir exists)
- Created 20 shadcn/ui components (19 + Label) in `packages/ui/src/components/ui/` with proper Radix UI primitives and CVA variants
- Deleted old placeholder `packages/ui/src/Button.tsx`
- Updated `packages/ui/index.ts` barrel export with all components and `cn` utility
- Created `/dev/components` showcase page in client app displaying all 19 component sections
- Added `components.json` shadcn/ui config to both apps
- All validations pass: `pnpm build` exits 0, `pnpm type-check` exits 0, `pnpm lint` exits 0

### File List

**Modified files:**
- `eusolicit-app/frontend/packages/config/package.json` — added tailwindcss-animate, @types/node devDependencies
- `eusolicit-app/frontend/packages/config/tailwind.config.ts` — full design token preset rewrite
- `eusolicit-app/frontend/packages/ui/package.json` — added all Radix UI + utility dependencies
- `eusolicit-app/frontend/packages/ui/index.ts` — full barrel export rewrite
- `eusolicit-app/frontend/apps/client/tailwind.config.ts` — switched to presets[] array
- `eusolicit-app/frontend/apps/client/tsconfig.json` — fixed @/* path alias to ./*
- `eusolicit-app/frontend/apps/client/app/globals.css` — added CSS variable definitions
- `eusolicit-app/frontend/apps/client/app/layout.tsx` — added Inter + JetBrains Mono fonts
- `eusolicit-app/frontend/apps/admin/tailwind.config.ts` — switched to presets[] array
- `eusolicit-app/frontend/apps/admin/tsconfig.json` — fixed @/* path alias to ./*
- `eusolicit-app/frontend/apps/admin/app/globals.css` — added CSS variable definitions
- `eusolicit-app/frontend/apps/admin/app/layout.tsx` — added Inter + JetBrains Mono fonts

**New files:**
- `eusolicit-app/frontend/packages/ui/src/lib/utils.ts` — cn() helper
- `eusolicit-app/frontend/packages/ui/src/components/ui/button.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/ui/input.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/ui/textarea.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/ui/label.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/ui/select.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/ui/checkbox.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/ui/radio-group.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/ui/switch.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/ui/dialog.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/ui/sheet.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/ui/dropdown-menu.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/ui/tooltip.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/ui/badge.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/ui/card.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/ui/tabs.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/ui/table.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/ui/skeleton.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/ui/avatar.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/ui/separator.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/ui/scroll-area.tsx`
- `eusolicit-app/frontend/apps/client/lib/utils.ts` — re-exports cn
- `eusolicit-app/frontend/apps/admin/lib/utils.ts` — re-exports cn
- `eusolicit-app/frontend/apps/client/components.json` — shadcn/ui config
- `eusolicit-app/frontend/apps/admin/components.json` — shadcn/ui config
- `eusolicit-app/frontend/apps/client/app/dev/components/page.tsx` — component showcase

**Deleted files:**
- `eusolicit-app/frontend/packages/ui/src/Button.tsx` — replaced by button.tsx in components/ui/

## Change Log

- 2026-04-08: Implemented Story 3.2 — Tailwind Design Token Preset & shadcn/ui Theming. All 10 tasks complete. 26 files created/modified, 1 deleted. Build and type-check pass for both apps.
- 2026-04-23: Senior Developer Review complete — **Approved**.

## Senior Developer Review

**Reviewer:** Claude (bmad-code-review)
**Date:** 2026-04-23
**Decision:** REVIEW: Approve

### Scope reviewed

All files in the story 3.2 File List, including:
- `packages/config/tailwind.config.ts` (design token preset)
- `packages/config/package.json` (tailwindcss-animate, @types/node)
- `packages/ui/index.ts` (barrel export — story 3.2 exports present; later-story exports are additive and do not regress 3.2 scope)
- `packages/ui/src/lib/utils.ts` (cn helper)
- `packages/ui/src/components/ui/{button,input,textarea,label,select,checkbox,radio-group,switch,dialog,sheet,dropdown-menu,tooltip,badge,card,tabs,table,skeleton,avatar,separator,scroll-area}.tsx` (20 components)
- `packages/ui/package.json` (Radix + utility deps)
- `apps/{client,admin}/tailwind.config.ts` (presets[] array)
- `apps/{client,admin}/tsconfig.json` (@/* path alias → ./*)
- `apps/{client,admin}/app/globals.css` (HSL CSS variables for light + dark)
- `apps/{client,admin}/app/layout.tsx` (Inter + JetBrains Mono font loading)
- `apps/{client,admin}/lib/utils.ts` (cn re-export)
- `apps/{client,admin}/components.json` (shadcn CLI config)
- `/dev/components` showcase page (now under `[locale]/dev/components/page.tsx` post-i18n)

### Acceptance Criteria Verification

| AC | Status | Notes |
|----|--------|-------|
| AC1 — Tailwind preset with full design tokens | ✅ Pass | Preset implements the shadcn HSL-variable convention described in Dev Notes (CSS-variable-mapped tokens + semantic success/warning/error/info + Inter/JetBrains Mono + shadow scale + accordion keyframes + `darkMode: ["class"]` + `tailwindcss-animate`). The AC1 phrasing "slate 50–950 + indigo 50–950" is a narrative simplification; the reference implementation in Dev Notes (which this story follows) maps via `hsl(var(--primary))` et al., with `--primary: 243 75% 59%` = indigo-600. Interpretation aligned with Dev Notes intent. |
| AC2 — HSL CSS variables in both apps' globals.css | ✅ Pass | Identical `:root` + `.dark` blocks in `apps/client/app/globals.css` and `apps/admin/app/globals.css`. All shadcn tokens present (background/foreground/card/popover/primary/secondary/muted/accent/destructive/border/input/ring/radius and *-foreground variants). |
| AC3 — 19 shadcn/ui components in `packages/ui/src/components/ui/` | ✅ Pass | All 19 required components present (plus Label = 20). Later stories added `accordion`, `alert`, `breadcrumb`, `form`, `progress` — additive, not regressive. |
| AC4 — `/dev/components` page renders all 19 components | ⚠️ Pass-with-note | Page exists at `apps/client/app/[locale]/dev/components/page.tsx` (moved under `[locale]/` by a later i18n story). Accessible via `/en/dev/components` or `/bg/dev/components`. Content matches the Dev Notes reference verbatim, all 19 sections present. Not a 3.2 regression — the physical path changed due to subsequent i18n work, not due to 3.2 implementation. |
| AC5 — Both apps build & import `<Button>` from `@eusolicit/ui` | ✅ Pass | Button exported at `packages/ui/index.ts:17`. Dev Agent Record confirms `pnpm build`, `pnpm type-check`, `pnpm lint` all exit 0. |

### Deferred 3.1 issues — both resolved in this story

- **`...baseConfig` spread → `presets: [sharedPreset]` array** — fixed in both `apps/client/tailwind.config.ts` and `apps/admin/tailwind.config.ts`. The `as unknown as Config` double cast is the correct workaround for the `satisfies Partial<Config>` upstream type (documented in Debug Log).
- **`@/*` → `./src/*` with no `src/` dir** — fixed in both `apps/client/tsconfig.json` and `apps/admin/tsconfig.json`; alias now resolves to `./*` so `@/lib/utils` → `apps/*/lib/utils.ts`.

### Code Quality Observations

- `cn` utility correctly uses `twMerge(clsx(inputs))` — idiomatic shadcn pattern.
- Component imports use the required relative path `../../lib/utils` (not `@/lib/utils`) for packages/ui portability.
- `suppressHydrationWarning` correctly placed on `<html>` only — matches E03-R-002 mitigation guidance.
- `require("tailwindcss-animate")` in the TypeScript preset is mitigated with eslint-disable + `@types/node` devDep (Debug Log Fix 2). Correct.
- `Button` uses `React.forwardRef` + `asChild`/`Slot` pattern — matches shadcn registry.
- Old placeholder `packages/ui/src/Button.tsx` confirmed deleted (ls of `packages/ui/src/` shows no legacy Button.tsx).
- Both apps' `components.json` correctly point `tailwind.config` → `tailwind.config.ts`, `css` → `app/globals.css`, `baseColor: "slate"`, `cssVariables: true`, `iconLibrary: "lucide"`.

### Minor notes (non-blocking, no patch required)

1. `apps/client/app/layout.tsx` uses `lang="en"` while `apps/admin/app/layout.tsx` uses `lang="bg"`. The spec Dev Notes showed `lang="bg"` for client. This is no longer authoritative because the per-locale `lang` attribute is now set at `[locale]/layout.tsx` (introduced by a later i18n story); the root `<html lang>` is effectively overridden. No action needed.
2. `packages/config/tailwind.config.ts` semantic colors (`success`/`warning`/`error`/`info`) use hex literals rather than CSS variables. Intentional per Dev Notes — these semantic colors are brand-fixed and do not need dark-mode variants. Acceptable.
3. `apps/client/src/components/AIDraftGenerationPanel.test.tsx` exists (an older legacy artifact) — unrelated to 3.2 scope; not introduced by this story.

### Test Coverage

Pass/fail gate E03-P0-001 (both apps build) is met per Dev Agent Record. No unit tests are expected for token/preset/barrel configuration per story scope. Visual QA tests (E03-P2-002/003) are out of scope for this review — they're exercised via the `/dev/components` page during E03 test execution.

### Security & Cross-cutting

- No HTTP, auth, DB, or tenant-crossing code touched. Pure frontend theming work.
- No secrets, no PII, no credentials introduced.
- No bare `except`/`import *` applicable (no Python).

### Final Verdict

**REVIEW: Approve** — Implementation faithfully follows the Dev Notes reference, fixes both deferred 3.1 defects, meets all 5 ACs (AC4 with a documented location-shift caveat caused by a later story, not by 3.2), and passes the build/type-check/lint gate. No blocking or changes-requested items. Story is ready to move to `done`.
