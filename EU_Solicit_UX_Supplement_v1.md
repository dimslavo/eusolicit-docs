# EU Solicit — UX Specification Supplement v1

**Platform**: EU Solicit | **Domain**: EUSolicit.com | **Version**: 1.0 | **Date**: 2026-04-05 | **Status**: Draft

**Companion document**: User Journeys, UX Workflows & E2E Dataflows v1

**Purpose**: This document extends the base UX specification by addressing all identified gaps from the Implementation Readiness Assessment. It covers: design system foundations, new screen specifications for features without UX coverage, component additions to existing screens, analytics dashboard breakdown, and cross-cutting UX patterns.

---

## Table of Contents

1. [Design System & UI Framework](#section-1-design-system--ui-framework) — Principles, responsive strategy, app shell, design tokens, components, accessibility, extensibility
2. [New Screen Specifications (Critical Gaps 1-5)](#screen-1-task-board) — Task Board, Approval Pipeline, Pricing Assistant, Clause Risk Panel, Requirements Checklist
3. [New Screen Specifications (Critical Gaps 6-10)](#screen-6-reusable-content-blocks-library) — Content Blocks Library, ESPD Builder, Consortium Finder, Reporting Templates, Regulation Tracker
4. [Component Additions to Existing Screens](#section-3-component-additions-to-existing-screens) — Relevance Score, Template Selector, Language Selector, Document Manager, Permissions, Usage Meters, Submission Guide, Add-On Purchase, Parsed Results
5. [Analytics Dashboard Breakdown](#section-4-analytics-dashboard-breakdown) — Market Intelligence, ROI Tracker, Team Performance, Competitor Intelligence, Pipeline Forecast, Usage, Reports
6. [Cross-Cutting UX Patterns](#section-5-cross-cutting-ux-patterns) — Empty States, AI Operation States, Notifications, Search/Filter, Mobile, Trial/Downgrade, Usage Warnings
7. [Updated Screen Inventory](#updated-screen-inventory)
8. [Updated UX Requirements Registry](#updated-ux-requirements-registry)

---

# Section 1: Design System & UI Framework

## 1.1 Design Principles

The EU Solicit interface follows four foundational principles that inform every design decision across the platform.

**Clean, Data-Dense, Professional SaaS Aesthetic**

The visual language draws from the design systems of Linear, Vercel, and Notion: generous whitespace balanced against information density, restrained color usage with purposeful accents, and typography-driven hierarchy. EU procurement data is complex — the interface must present it clearly without oversimplifying. Every pixel serves a purpose: either conveying information or creating the breathing room that makes that information legible.

- Flat design with subtle elevation for layering (cards over background, dropdowns over cards)
- Monochromatic base palette (slate neutrals) with a single primary accent (indigo) and semantic colors reserved strictly for status communication
- No decorative elements, gradients, or ornamental borders — visual interest comes from content, typography, and spacing
- Dense but not cramped: data tables show 15–25 rows without scrolling on desktop; cards carry 4–6 key data points without feeling overloaded

**Content-First**

Data is the product. The interface chrome (sidebars, toolbars, navigation) exists to serve the content, never to compete with it. Structural UI elements use muted colors and thin borders; content areas receive maximum width and visual prominence.

- Sidebar collapses to icon-only mode to reclaim horizontal space
- Detail panels use the full viewport height with minimal header overhead
- Charts, tables, and proposal editors extend to container edges — no unnecessary inset padding
- Loading states (skeletons) mirror the exact layout of incoming content so the page does not reflow

**Progressive Disclosure**

EU procurement workflows involve deep, multi-layered data: a single opportunity can have dozens of requirements, multiple evaluation criteria, compliance frameworks, and associated documents. Showing everything at once overwhelms; hiding it creates friction. Progressive disclosure resolves this tension.

- Primary information is always visible (opportunity title, deadline, budget, status, source)
- Secondary information is one click away (full requirements list, evaluation criteria weights, parsed document details)
- Tertiary information is accessed through explicit drill-down (audit history, raw source data, advanced analytics)
- Patterns: tabs for peer-level sections, accordions for expandable sub-sections, inspector panel for contextual detail, modals for focused tasks

**Consistent Patterns**

The same interaction should produce the same visual and behavioral result regardless of context. A user who learns to filter opportunities can immediately filter proposals, grants, or audit logs because the filter mechanism is identical.

- One data table component with a single API; variations come from column configuration, not component forks
- One filter panel pattern (bar + chips + sheet) used everywhere filterable lists appear
- One AI operation state machine (idle → requesting → streaming/processing → complete/error) applied to all eleven AI features
- One empty state template (icon + headline + description + CTA) used across all list and dashboard views

---

## 1.2 Responsive Strategy

**Breakpoint System**

The application uses Tailwind CSS's mobile-first breakpoint system. All styles are authored for the smallest screen first, then layered upward.

| Token | Min Width | Target Devices | Layout Tier |
|-------|-----------|----------------|-------------|
| (default) | 0px | Small phones (iPhone SE, Galaxy S series) | Mobile |
| `sm` | 640px | Large phones in landscape, small tablets | Mobile |
| `md` | 768px | Tablets in portrait (iPad Mini, iPad) | Tablet |
| `lg` | 1024px | Tablets in landscape, small laptops | Desktop |
| `xl` | 1280px | Standard laptops, external monitors | Desktop |
| `2xl` | 1536px | Large monitors, ultrawide displays | Desktop |

**Layout Tier Specifications**

**Mobile (< 768px)**

| Aspect | Specification |
|--------|---------------|
| Navigation | Bottom tab bar: 5 icons (Dashboard, Opportunities, Proposals, Calendar, More). "More" opens a full-screen Sheet with remaining items. |
| Layout | Single column, full-width. No sidebar, no inspector panel. |
| Content | Cards stack vertically. Data tables transform to card lists. |
| Drawers | All secondary panels (filters, detail views, navigation overflow) use Sheet (bottom drawer, 90vh max height). |
| Touch targets | Minimum 44x44px for all interactive elements. Minimum 8px gap between adjacent targets. |
| Typography | Body text minimum 16px to prevent iOS zoom on input focus. |
| Spacing | Horizontal page padding: 16px. Card padding: 12px–16px. |

**Tablet (768px–1023px)**

| Aspect | Specification |
|--------|---------------|
| Navigation | Collapsible left sidebar. Defaults to collapsed (icon-only, 64px). Toggle via hamburger icon in top bar. Overlays content when expanded (does not push). |
| Layout | 1–2 columns depending on content. Opportunity list: single column card list or compact table. Dashboard: 2-column metric grid. |
| Inspector | Not available as a pinned side panel. Opens as a Sheet from right (60vw width). |
| Touch targets | Minimum 44x44px maintained. |
| Spacing | Horizontal page padding: 24px. |

**Desktop (1024px+)**

| Aspect | Specification |
|--------|---------------|
| Navigation | Persistent left sidebar. Defaults to expanded (256px) on first visit. Collapse state persisted in localStorage. |
| Layout | Sidebar + main content + optional right inspector. Main content area: fluid width, max-width 1440px centered on 2xl. |
| Inspector | Right panel slides in from right edge, 380px wide on lg/xl, 420px on 2xl. Can be pinned open or dismissed. When pinned, main content area compresses. |
| Data tables | Full tabular display with sortable column headers, row hover states, and bulk selection checkboxes. |
| Spacing | Horizontal page padding: 32px. Gap between sidebar and content: 0 (sidebar is a distinct region with its own background). |

**Responsive Conversion Rules**

| Desktop Component | Mobile Replacement | Trigger |
|-------------------|--------------------|---------|
| Data table | Card list with key fields | < 768px |
| Right inspector panel | Full-screen Sheet | < 1024px |
| Filter sidebar section | Bottom Sheet | < 768px |
| Multi-column grid (3-col) | Single column stack | < 768px |
| Multi-column grid (3-col) | 2-column grid | 768px–1023px |
| Side-by-side comparison | Stacked or tabbed | < 1024px |
| Horizontal tabs | Horizontally scrollable pill tabs | < 640px |
| Modal dialog (wide) | Full-screen dialog | < 768px |
| Context menu (right-click) | Long-press menu | Touch devices |
| Hover tooltip | Tap to toggle tooltip | Touch devices |
| Drag-and-drop | Long-press to initiate, then drag | Touch devices |

---

## 1.3 Application Shell

### Client Application Shell (`eusolicit.com`)

The shell consists of three regions: a left sidebar, a top bar, and a main content area. An optional right inspector panel overlays or compresses the main content area when activated.

```
┌──────────────────────────────────────────────────────────────────────┐
│ Top Bar: Breadcrumbs | Cmd+K Search | Notifications | Lang | User   │
├────────────┬─────────────────────────────────────┬───────────────────┤
│            │                                     │                   │
│  Sidebar   │         Main Content Area           │  Inspector Panel  │
│  (256px /  │         (fluid width)               │  (380px, opt.)    │
│   64px)    │                                     │                   │
│            │                                     │                   │
│  - Dash    │                                     │                   │
│  - Opps    │                                     │                   │
│  - Props   │                                     │                   │
│  - Grants  │                                     │                   │
│  - Analyt  │                                     │                   │
│  - Calndr  │                                     │                   │
│  - Settngs │                                     │                   │
│            │                                     │                   │
│  [Avatar]  │                                     │                   │
│  [Tier]    │                                     │                   │
└────────────┴─────────────────────────────────────┴───────────────────┘
```

**Left Sidebar**

| Property | Expanded | Collapsed |
|----------|----------|-----------|
| Width | 256px | 64px |
| Toggle | Chevron button at sidebar bottom edge, or Cmd+B / Ctrl+B | Same |
| State persistence | localStorage key `sidebar-collapsed` | — |
| Transition | 200ms ease-out width animation | — |
| Background | `--color-surface-sidebar` (slate-50 light / slate-900 dark) | Same |
| Border | 1px right border, `--color-border-subtle` | Same |

Sidebar navigation items, top to bottom:

| Section | Icon | Label | Badge |
|---------|------|-------|-------|
| Logo / App name | EU Solicit mark | "EU Solicit" (hidden when collapsed) | — |
| Dashboard | `LayoutDashboard` | Dashboard | — |
| Opportunities | `Search` | Opportunities | Count of new since last visit |
| Proposals | `FileText` | Proposals | Count of in-progress |
| Grants | `Award` | Grants | — |
| Analytics | `BarChart3` | Analytics | — |
| Calendar | `CalendarDays` | Calendar | Count of deadlines this week |
| Content Library | `Library` | Content Blocks | — |
| Lessons Learned | `Lightbulb` | Lessons Learned | — |
| — (divider) | | | |
| Settings | `Settings` | Settings | — |
| User section | Avatar circle | User name + tier badge (e.g., "Professional") | — |

In collapsed mode, hovering over an icon shows a tooltip with the label. Badges render as small colored dots on the icon.

**Top Bar**

Height: 56px. Sticky at top of viewport.

| Element | Position | Behavior |
|---------|----------|----------|
| Breadcrumbs | Left | Auto-generated from route segments. E.g., `Opportunities > OPP-2024-0847 > Analysis`. Clickable segments. Truncated with ellipsis on mobile. |
| Global Search | Center | Input field with `Cmd+K` hint. On click or shortcut, opens CommandMenu overlay. On desktop, the input is 320px wide. On mobile, replaced by a search icon that opens CommandMenu. |
| Notification Bell | Right | Bell icon with unread count badge (red dot with number, max "99+"). Opens notification drawer on click. |
| Language Switcher | Right | Dropdown: EN, BG (phase 1). Displays current language as 2-letter code. |
| User Menu | Right | Avatar circle. Dropdown: Profile, Organization Settings, Billing, Help & Support, Sign Out. |

**Right Inspector Panel**

- Triggered by: clicking a row in a data table, clicking "View Details" on a card, or explicit "Open Inspector" action
- Width: 380px (lg/xl), 420px (2xl)
- On screens < 1024px: renders as a Sheet (drawer from right, 85vw width, max 420px)
- Header: entity title + close button + "Open Full Page" link
- Content: context-dependent. For opportunities: key metadata, deadline countdown, quick actions (Summarize, Check Eligibility, Start Proposal). For proposals: status, completion percentage, quick-edit fields.
- Pinning: a pin icon in the header toggles between pinned (compresses main content) and overlay (floats over main content with backdrop scrim) modes. Default: overlay on first use, state persisted in localStorage.

### Admin Application Shell (`admin.eusolicit.com`)

Identical shell structure. Sidebar navigation items differ:

| Section | Icon | Label |
|---------|------|-------|
| Dashboard | `LayoutDashboard` | Admin Dashboard |
| Organizations | `Building2` | Organizations |
| Users | `Users` | Users |
| Subscriptions | `CreditCard` | Subscriptions |
| Data Pipelines | `Database` | Data Pipelines |
| White-Label | `Palette` | White-Label |
| Compliance | `Shield` | Compliance Frameworks |
| Audit Logs | `ScrollText` | Audit Logs |
| System Config | `Wrench` | System Config |

---

## 1.4 Design Tokens

All design tokens are defined as CSS custom properties on `:root` and overridden under a `.dark` class (or `[data-theme="dark"]` attribute) for dark mode. This enables runtime theme switching and white-label overrides without recompiling Tailwind.

### Color Tokens

**Primary Palette**

| Token | Light Value | Dark Value | Usage |
|-------|-------------|------------|-------|
| `--color-primary-50` | `#eef2ff` | `#1e1b4b` | Hover backgrounds |
| `--color-primary-100` | `#e0e7ff` | `#312e81` | Selected backgrounds |
| `--color-primary-200` | `#c7d2fe` | `#3730a3` | Borders on active elements |
| `--color-primary-500` | `#6366f1` | `#818cf8` | Secondary text, icons |
| `--color-primary-600` | `#4f46e5` | `#a5b4fc` | Primary buttons, links |
| `--color-primary-700` | `#4338ca` | `#c7d2fe` | Button hover |

**Semantic Colors**

| Token | Light Value | Dark Value | Usage |
|-------|-------------|------------|-------|
| `--color-success` | `#16a34a` (green-600) | `#4ade80` (green-400) | Awarded status, completion, verification |
| `--color-success-bg` | `#f0fdf4` (green-50) | `#052e16` (green-950) | Success alert backgrounds |
| `--color-warning` | `#d97706` (amber-600) | `#fbbf24` (amber-400) | Closing soon, approaching limits, medium risk |
| `--color-warning-bg` | `#fffbeb` (amber-50) | `#451a03` (amber-950) | Warning alert backgrounds |
| `--color-error` | `#dc2626` (red-600) | `#f87171` (red-400) | Overdue, errors, high/critical risk, destructive actions |
| `--color-error-bg` | `#fef2f2` (red-50) | `#450a0a` (red-950) | Error alert backgrounds |
| `--color-info` | `#2563eb` (blue-600) | `#60a5fa` (blue-400) | Informational notices, tips |
| `--color-info-bg` | `#eff6ff` (blue-50) | `#172554` (blue-950) | Info alert backgrounds |

**Neutral Scale**

Based on Tailwind's `slate` palette. Used for text, borders, backgrounds, and structural UI elements.

| Token | Light Value | Dark Value | Usage |
|-------|-------------|------------|-------|
| `--color-bg-primary` | `#ffffff` | `#0f172a` (slate-900) | Page background |
| `--color-bg-secondary` | `#f8fafc` (slate-50) | `#1e293b` (slate-800) | Card backgrounds, sidebar |
| `--color-bg-tertiary` | `#f1f5f9` (slate-100) | `#334155` (slate-700) | Table row hover, input backgrounds |
| `--color-text-primary` | `#0f172a` (slate-900) | `#f8fafc` (slate-50) | Headings, primary body text |
| `--color-text-secondary` | `#475569` (slate-600) | `#94a3b8` (slate-400) | Descriptions, secondary text |
| `--color-text-tertiary` | `#94a3b8` (slate-400) | `#64748b` (slate-500) | Placeholders, captions, disabled |
| `--color-border-default` | `#e2e8f0` (slate-200) | `#334155` (slate-700) | Card borders, dividers |
| `--color-border-subtle` | `#f1f5f9` (slate-100) | `#1e293b` (slate-800) | Separator lines, sidebar border |

**Data Source Colors**

| Source | Color | Token |
|--------|-------|-------|
| AOP | Blue (`#2563eb`) | `--color-source-aop` |
| TED | Green (`#16a34a`) | `--color-source-ted` |
| EU Grants | Purple (`#9333ea`) | `--color-source-eu-grants` |

### Typography Tokens

Font family: `Inter, -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif`. Loaded via `next/font/google` for zero-CLS font loading.

| Token | Size | Line Height | Weight | Usage |
|-------|------|-------------|--------|-------|
| `--text-2xl` | 30px / 1.875rem | 36px | 700 (bold) | Page titles |
| `--text-xl` | 24px / 1.5rem | 32px | 600 (semibold) | Section headings |
| `--text-lg` | 20px / 1.25rem | 28px | 600 (semibold) | Card titles, dialog titles |
| `--text-base` | 16px / 1rem | 24px | 400 (regular) | Body text, table cells |
| `--text-sm` | 14px / 0.875rem | 20px | 400 (regular) | Secondary text, form labels |
| `--text-xs` | 12px / 0.75rem | 16px | 500 (medium) | Badges, captions, timestamps |

### Spacing Scale

Base unit: 4px. All spacing values are multiples of this base.

| Token | Value | Common Usage |
|-------|-------|--------------|
| `--space-0` | 0px | — |
| `--space-1` | 4px | Tight gaps (badge padding, icon-label gap) |
| `--space-2` | 8px | Inline element spacing, compact form gaps |
| `--space-3` | 12px | Card internal padding (mobile), small sections |
| `--space-4` | 16px | Standard card padding, form field gaps, page padding (mobile) |
| `--space-5` | 20px | Section gaps within a card |
| `--space-6` | 24px | Page padding (tablet), gaps between cards |
| `--space-8` | 32px | Page padding (desktop), major section dividers |
| `--space-10` | 40px | Page-level vertical spacing between sections |
| `--space-12` | 48px | Large section padding |
| `--space-16` | 64px | Sidebar collapsed width, major layout spacing |

### Elevation

| Token | Value (Light) | Value (Dark) | Usage |
|-------|---------------|--------------|-------|
| `--shadow-sm` | `0 1px 2px 0 rgba(0,0,0,0.05)` | `0 1px 2px 0 rgba(0,0,0,0.3)` | Cards, buttons on hover |
| `--shadow-md` | `0 4px 6px -1px rgba(0,0,0,0.1), 0 2px 4px -2px rgba(0,0,0,0.1)` | `0 4px 6px -1px rgba(0,0,0,0.4), 0 2px 4px -2px rgba(0,0,0,0.3)` | Dropdowns, popovers, floating panels |
| `--shadow-lg` | `0 10px 15px -3px rgba(0,0,0,0.1), 0 4px 6px -4px rgba(0,0,0,0.1)` | `0 10px 15px -3px rgba(0,0,0,0.5), 0 4px 6px -4px rgba(0,0,0,0.4)` | Modals, command menu, sheets |

### Border Radius

| Token | Value | Usage |
|-------|-------|-------|
| `--radius-sm` | 4px | Badges, small buttons, input elements |
| `--radius-md` | 8px | Cards, dropdowns, dialogs |
| `--radius-lg` | 12px | Large cards, modal containers, panels |
| `--radius-full` | 9999px | Avatars, circular badges, pills |

### Transition Tokens

| Token | Duration | Easing | Usage |
|-------|----------|--------|-------|
| `--transition-fast` | 150ms | `ease-out` | Button hover/active, badge appear, toggle switch |
| `--transition-normal` | 200ms | `ease-in-out` | Panel slide, sidebar collapse, accordion expand |
| `--transition-slow` | 300ms | `ease-in-out` | Page transitions, sheet open/close, modal appear |
| `--transition-spring` | 300ms | `cubic-bezier(0.34, 1.56, 0.64, 1)` | Notification pop-in, toast entrance |

All transitions respect `prefers-reduced-motion: reduce` — when this media query matches, all durations drop to 0ms.

---

## 1.5 Core Component Patterns

All components are based on shadcn/ui, installed locally via the CLI into `src/components/ui/`. They are project-owned, not dependency-owned, and are extended as needed.

### DataTable

The primary component for listing opportunities, proposals, grants, audit logs, users, and organizations.

| Property | Specification |
|----------|---------------|
| Base | `@tanstack/react-table` headless table, rendered with shadcn/ui Table primitives |
| Sorting | Click column header to sort asc/desc/none. Sort state synced to URL params (`?sort=deadline&order=asc`). |
| Filtering | Integrated with the global filter system (see Section 5.4). Column-level quick filters via dropdown in column header. |
| Pagination | Bottom bar: rows-per-page selector (10/25/50/100), page navigation (prev/next + page numbers), total count. Server-side pagination via TanStack Query `keepPreviousData`. |
| Row selection | Checkbox column on left. "Select all on page" + "Select all matching filter" (for bulk actions). |
| Bulk actions | When rows selected, a sticky toolbar appears above the table: selected count + action buttons (e.g., "Export", "Add to Proposal", "Archive"). |
| Column visibility | Column toggle dropdown accessible from a settings icon in the table toolbar. Persisted in localStorage per-table. |
| Row actions | Last column: `MoreHorizontal` icon opening DropdownMenu with context actions. |
| Loading | Full skeleton rows matching column structure. Preserved pagination controls (disabled). |
| Empty | Empty state component rendered in place of table body (see Section 5.1). |
| Responsive | Below 768px, table transforms to a card list. Each card shows the 4–5 most important fields. Swipe right reveals action buttons. |

### CommandMenu (Cmd+K)

| Property | Specification |
|----------|---------------|
| Base | shadcn/ui Command (wraps cmdk library) |
| Trigger | `Cmd+K` (Mac) / `Ctrl+K` (Windows), or click the search input in the top bar |
| Appearance | Centered modal overlay with backdrop scrim. 640px max width, 480px max height. |
| Input | Auto-focused text input at top. Debounced query (200ms). |
| Categories | Results grouped: "Recent" (last 5 searches), "Opportunities" (title, CPV, authority), "Proposals" (title, linked opportunity), "Organizations" (name), "Actions" (navigate to page, e.g., "Go to Settings"). |
| Keyboard nav | Arrow keys to navigate results, Enter to select, Escape to close. Category headers are skipped. |
| Loading | Inline spinner below input while fetching. |
| No results | "No results found for '[query]'" with suggestion to try different terms. |

### Sheet (Drawer)

| Property | Specification |
|----------|---------------|
| Base | shadcn/ui Sheet (wraps Radix Dialog with directional slide) |
| Side variants | `left` (mobile nav), `right` (inspector, notification drawer), `bottom` (mobile filters, mobile detail views) |
| Sizing | Right: 380px (lg), 420px (2xl), 85vw (mobile, max 420px). Bottom: 90vh max height, drag-to-resize handle. Left: 280px. |
| Backdrop | Semi-transparent scrim (`rgba(0,0,0,0.4)`). Click to dismiss. |
| Animation | Slide in from edge, 200ms ease-in-out. |
| Close | X button in header, Escape key, backdrop click, swipe-to-dismiss on touch (bottom sheets). |

### Dialog / AlertDialog

| Property | Specification |
|----------|---------------|
| Usage | Dialog: forms, confirmations, multi-step workflows. AlertDialog: destructive confirmations (delete, downgrade, archive). |
| Sizing | `sm` (400px) for simple confirmations, `md` (560px) for forms, `lg` (720px) for complex content. Full-screen on mobile (< 768px). |
| AlertDialog | Requires explicit confirmation. Destructive action button is red. "Cancel" is always available and focused by default. Cannot be dismissed by clicking backdrop. |
| Stacking | Maximum 2 levels of dialog stacking (e.g., Dialog opens AlertDialog for "Are you sure?"). |

### Tabs

| Property | Specification |
|----------|---------------|
| Base | shadcn/ui Tabs (wraps Radix Tabs) |
| Variant: Underline | Used for page-level sub-navigation (Opportunity detail tabs). Tab list has a bottom border; active tab has a 2px primary-colored bottom border. |
| Variant: Pills | Used for filter group toggles and inline option switching. Rounded-full background on active tab. |
| Responsive | Below 640px, tab list becomes horizontally scrollable with no wrapping. Active tab auto-scrolls into view. Fade edges indicate overflow. |
| URL sync | Page-level tabs sync to URL hash (`#overview`, `#documents`) for deep linking. Inline tabs use local state. |

### Card

| Property | Specification |
|----------|---------------|
| Base | shadcn/ui Card with custom variants |
| Variant: OpportunityCard | Source badge (top-left), title (linked), contracting authority, budget range, deadline with countdown, relevance score bar, status badge. Footer: quick actions (Summarize, View, Track). |
| Variant: MetricCard | Label, large numeric value, trend indicator (up/down arrow + percentage), optional sparkline (Recharts). Used on Dashboard. |
| Variant: StatusCard | Icon, title, description, status badge. Used for proposal status, grant status. |
| Hover | Subtle shadow lift (`--shadow-sm` → `--shadow-md`), 150ms transition. |
| Click | Entire card is clickable (wrapped in a Link or onClick handler). Inline action buttons use `stopPropagation` to prevent card navigation. |

### Badge

| Variant | Colors | Usage |
|---------|--------|-------|
| Status: Open | Green background, green text | Opportunity is accepting submissions |
| Status: Closing Soon | Amber background, amber text | Deadline within 7 days |
| Status: Closed | Slate background, slate text | Submission deadline passed |
| Status: Awarded | Blue background, blue text | Contract/grant has been awarded |
| Risk: Low | Green outline | Risk assessment result |
| Risk: Medium | Amber outline | Risk assessment result |
| Risk: High | Red outline | Risk assessment result |
| Risk: Critical | Red solid background, white text | Risk assessment result |
| Tier: Free | Slate outline | User/org subscription tier |
| Tier: Professional | Indigo solid | User/org subscription tier |
| Tier: Enterprise | Gradient indigo-to-purple | User/org subscription tier |
| Source: AOP | Blue solid | Data source indicator |
| Source: TED | Green solid | Data source indicator |
| Source: EU Grants | Purple solid | Data source indicator |

### Toast

| Property | Specification |
|----------|---------------|
| Position | Bottom-right on desktop. Bottom-center on mobile (full-width with 16px horizontal margins). |
| Duration | Success: 4 seconds. Error: persistent until dismissed. Info: 5 seconds. |
| Stacking | Max 3 visible, newer on top. Older toasts slide down. |
| Content | Icon (success/error/info) + title + optional description + optional action button ("Undo", "View"). |
| Dismiss | X button, swipe right (touch), auto-dismiss after duration. |

### Skeleton

Every data-fetching component has a corresponding skeleton state. Skeletons exactly match the layout of loaded content: same heights, widths, and spacing. Skeleton blocks use `--color-bg-tertiary` with a shimmer animation (left-to-right gradient sweep, 1.5s cycle). Skeletons are pre-built per component, not generated dynamically.

### DropdownMenu

| Property | Specification |
|----------|---------------|
| Base | shadcn/ui DropdownMenu (wraps Radix) |
| Trigger | Icon button (`MoreHorizontal`, `ChevronDown`), or right-click context menu on desktop |
| Content | Menu items with optional icons (left) and keyboard shortcuts (right). Separators between groups. Destructive items in red. |
| Sub-menus | One level of nesting supported (e.g., "Move to..." → list of folders). |

---

## 1.6 Accessibility (WCAG 2.1 AA)

The application targets WCAG 2.1 AA compliance. shadcn/ui's foundation on Radix primitives provides a strong accessible baseline; the specifications below codify additional requirements.

**Color and Contrast**

| Requirement | Target | Verification |
|-------------|--------|--------------|
| Normal text contrast (< 18pt) | 4.5:1 minimum | Automated check via axe-core in CI |
| Large text contrast (>= 18pt or >= 14pt bold) | 3:1 minimum | Automated check |
| UI component contrast (borders, icons, focus rings) | 3:1 minimum against adjacent colors | Manual review for custom components |
| Non-text contrast (charts, data visualizations) | 3:1 minimum | Chart colors validated per palette |
| Color is not the sole indicator | All color-coded statuses also have text labels or icons | Design review checklist |

**Keyboard Navigation**

| Pattern | Behavior |
|---------|----------|
| Focus ring | 2px solid `--color-primary-500`, 2px offset. Visible only on `:focus-visible` (keyboard navigation), not on `:focus` (mouse click). |
| Tab order | Logical reading order: sidebar (when focused), top bar, main content (top to bottom, left to right), inspector panel. |
| Skip link | Hidden link "Skip to main content" appears on first Tab press, jumps focus to `<main>` landmark. |
| Roving tabindex | Within tab lists, toolbars, and menu bars: Tab enters the group, arrow keys navigate within, Tab exits. Per Radix defaults. |
| Escape key | Closes the topmost overlay (modal, dropdown, sheet, command menu). |
| Data table | Arrow keys navigate cells. Enter activates row action or enters edit mode. Space toggles row selection. |

**Screen Readers**

| Element | ARIA Implementation |
|---------|---------------------|
| Sidebar navigation | `<nav aria-label="Main navigation">`. Current page marked with `aria-current="page"`. |
| Badges | `aria-label` includes full text, e.g., `aria-label="Status: Closing Soon"`. |
| Icon-only buttons | `aria-label` describes the action, e.g., `aria-label="Open notifications"`. |
| Charts | `role="img"` with `aria-label` summarizing the data. Detailed data available in an associated table (visually hidden, accessible to screen readers). |
| Loading states | Skeleton regions have `aria-busy="true"`. Completion announced via `aria-live="polite"` region. |
| Form validation | Error messages linked via `aria-describedby`. Errors announced via `aria-live="assertive"` on submit. Inline errors announced via `aria-live="polite"` on field blur. |
| Toasts | Toast container is `aria-live="polite"` with `role="status"`. Error toasts use `aria-live="assertive"`. |
| Modals/Dialogs | Focus trapped within modal. Title linked via `aria-labelledby`. Description linked via `aria-describedby`. |

**Reduced Motion**

When `prefers-reduced-motion: reduce` is detected:
- All CSS transitions and animations are set to `duration: 0ms`
- Skeleton shimmer animation is replaced with a static gray background
- Sheet/modal open/close is instant (no slide or fade)
- Page transitions are instant
- Toast entrance is instant (no spring animation)
- Chart animations (Recharts) are disabled

---

## 1.7 Extensibility Architecture

**Component Ownership Model**

shadcn/ui components are installed into `src/components/ui/` and become project-owned source code. This allows:
- Direct modification of component internals without fighting library abstractions
- Custom variants added alongside existing ones (e.g., adding a `variant="metric"` to Card)
- No dependency version lock-in; components evolve with the project

Feature-specific composite components are built in `src/components/features/` and compose UI primitives. For example, `OpportunityCard` composes `Card`, `Badge`, `Button`, and feature-specific logic.

**Feature Flag System**

Feature flags control the visibility of features across the UI. They are evaluated at render time and influence both navigation items and in-page elements.

| Flag Source | Behavior |
|-------------|----------|
| Subscription tier | Determines which features are available. Free tier users see gated features with upgrade prompts. |
| Server-side feature flags | Controls progressive rollout of new features (e.g., "beta" features shown to 10% of users). Flags fetched at session start and cached. |
| Environment flags | `NEXT_PUBLIC_FF_*` environment variables for build-time flags (e.g., disabling features in staging). |

Gated features display a consistent "locked" treatment: the UI element is visible but visually muted, with a lock icon and "Available on Professional" tooltip. Clicking opens the upgrade modal.

**Slot-Based Layouts**

The application shell uses named slots that allow new features to register themselves without modifying the shell component:

| Slot | Location | Registration |
|------|----------|--------------|
| `sidebar-nav-items` | Left sidebar, below core nav | New nav items registered via route-based convention (`/app/(dashboard)/[feature]/layout.tsx` exports sidebar config) |
| `inspector-tabs` | Right inspector panel tab list | Feature modules export inspector tab definitions |
| `top-bar-actions` | Top bar, before notification bell | Feature modules export action button definitions |
| `dashboard-widgets` | Dashboard grid | Widget definitions registered in a widget registry; users can configure visible widgets |

**Theme Token Override (White-Label)**

White-label theming operates by overriding CSS custom property values at the `:root` level. A white-label configuration object is loaded at app initialization and injected as a `<style>` tag:

```
Primary color → --color-primary-{50..900}
Accent color → --color-accent-{50..900}
Logo URL → rendered in sidebar via config
App name → rendered in sidebar, page titles, emails
```

No component code changes are required for white-label deployments. The same codebase renders different visual identities purely through token substitution.

**Route-Based Code Splitting**

Next.js App Router provides automatic code splitting per route segment. Each feature is a route group:

```
src/app/(dashboard)/
  opportunities/     → Opportunity list + detail pages
  proposals/         → Proposal list + editor + detail
  grants/            → Grant tracking
  analytics/         → Charts and reports
  calendar/          → Deadline calendar
  settings/          → User and org settings
```

New features are added as new route segments. They automatically receive code splitting, can define their own layouts, and register themselves in the sidebar via the slot system described above.

---

---


---


I now have all the context needed. Here are the complete screen specifications:

---

# EU Solicit -- Screen Specifications for Missing UX Features

## Table of Contents

1. [Screen 1: Task Board](#screen-1-task-board)
2. [Screen 2: Approval Pipeline](#screen-2-approval-pipeline)
3. [Screen 3: Pricing Assistant Panel](#screen-3-pricing-assistant-panel)
4. [Screen 4: Clause Risk Flagging Panel](#screen-4-clause-risk-flagging-panel)
5. [Screen 5: Requirements Checklist Panel](#screen-5-requirements-checklist-panel)

---

## Screen 1: Task Board

### Route

`/app/opportunities/[opportunityId]/tasks`

Also accessible as a tab ("Tasks") within the Opportunity Detail page shell at `/app/opportunities/[opportunityId]`.

### Access

| Tier | Access Level |
|------|-------------|
| Free | No access (gated) |
| Starter | View-only (read tasks generated by AI summary, no create/assign) |
| Professional | Full access (create, assign, drag, manage dependencies) |
| Enterprise | Full access + cross-opportunity task rollup on dashboard |
| **Roles** | Bid Manager: full CRUD. Contributor/Technical Writer: create & edit own tasks, move own tasks. Reviewer: view-only. Read-only: view-only. Admin: full CRUD + delete. |

### Purpose

Per-opportunity Kanban-style task management board with list view toggle, enabling teams to orchestrate the bid preparation workflow from opportunity identification through final submission.

### Layout

- **App shell**: Collapsible sidebar with "Opportunities" selected. Opportunity Detail page header (opportunity title, status badge, deadline countdown) persists at top.
- **Tab bar**: Analysis | Requirements | Risk Analysis | Proposals | **Tasks** | Timeline
- **Content area**: Full width below tab bar. View toggle (Board / List) in top-right of content area. Filter bar immediately below.
- **Inspector panel** (optional right panel): Opens when a task card is clicked, showing full task detail form. On desktop, pushes content left (inspector is 400px). On tablet/mobile, opens as a slide-over sheet.

### Components

| Component | Type | Data | Behavior |
|-----------|------|------|----------|
| **View Toggle** | `shadcn/ui ToggleGroup` | Board / List | Persists preference in Zustand store. Default: Board. |
| **Filter Bar** | Horizontal bar with `Select` dropdowns | Assignee (avatar + name), Priority (Critical/High/Medium/Low), Due Date (Overdue/Today/This Week/Later/No Date) | Filters are composable (AND logic). Active filter count shown as badge. "Clear all" link appears when any filter active. |
| **"+ Add Task" Button** | `Button` (primary) | — | Opens inline task creation card at top of "To Do" column (Board) or top of list (List). |
| **"Apply Template" Button** | `Button` (outline, secondary) | — | Opens template selection dialog. Available templates: "Tender Preparation" (12 tasks), "Grant Application" (10 tasks), "Framework Agreement" (8 tasks). Auto-populates To Do column. Only shown when task count = 0. |
| **Board View** | 4-column Kanban (`dnd-kit` or `@hello-pangea/dnd`) | Columns: To Do, In Progress, In Review, Done | Drag cards between columns and within columns to reorder. Column headers show task count. Scroll within columns when overflow. |
| **Task Card** (Board) | `Card` | Title (truncated 2 lines), assignee avatar (24px circle, tooltip with name), due date (relative: "in 3d", "overdue" in red), priority badge (colored dot: red/orange/yellow/gray), dependency indicator (chain-link icon if blocked), proposal link icon (if linked to proposal section) | Click opens Inspector. Drag handle on hover. Blocked tasks show a red left border and "Blocked by [task name]" chip. |
| **List View** | `Table` with sortable columns | Columns: Checkbox, Title, Status (dropdown), Assignee (avatar), Priority (badge), Due Date, Dependencies, Linked Section | Sort by any column. Inline status change via dropdown. Bulk actions on checked rows (assign, change priority, move to column, delete). |
| **Task Inspector Panel** | Right slide-over panel (400px desktop, full-screen sheet mobile) | Title (editable inline), Description (Tiptap mini-editor, 200px), Status dropdown, Assignee dropdown (team members), Priority select, Due Date picker, Dependencies multi-select (other tasks from this opportunity), Linked Proposal Section dropdown, Subtask checklist (add/check/delete), Activity log (timestamped entries) | Save on blur/change (optimistic updates via TanStack Query mutation). "Delete task" in overflow menu with confirmation dialog. |
| **Dependency Visualization** (Board) | Visual indicator on cards | Blocked tasks, blocking tasks | Blocked tasks have a red lock icon + tooltip "Blocked by: [task title]". When hovering a task with dependencies, related task cards pulse with a blue outline. No connecting lines (too complex for Kanban; reserved for Gantt view in Timeline tab). |
| **Task Template Dialog** | `Dialog` | Template name, description, preview list of tasks with default assignments | "Apply Template" inserts tasks into To Do. If tasks already exist, confirm dialog: "This will add X tasks. Existing tasks are unchanged." |
| **Inline Create Card** | Inline form at top of "To Do" column | Title input (auto-focus), Enter to save, Escape to cancel | After save, card appears in To Do. Click the card to open Inspector for full details. |
| **Column Drop Zone** | Visual feedback area | — | When dragging, columns show a dashed blue border and background tint. Drop position indicated by a horizontal blue line between cards. |
| **Empty State** | Centered illustration + CTA | "No tasks yet" | Shows: "Start organizing your bid preparation. Apply a template to get started, or create your first task." Two buttons: "Apply Template" (primary), "Create Task" (secondary). |
| **Progress Summary Bar** | Horizontal bar above columns | Task counts per status, completion % | "4 of 12 tasks complete (33%)" with a segmented progress bar (To Do: gray, In Progress: blue, In Review: amber, Done: green). |

### Interactions

| User Action | System Response |
|-------------|----------------|
| Drags task card from "To Do" to "In Progress" | Optimistic UI update. PATCH task status. If task is blocked by an incomplete dependency, show toast: "This task is blocked by [dep name]. Move anyway?" with Confirm/Cancel. Audit log entry created. |
| Clicks "+ Add Task" | Inline card appears at top of To Do column. Cursor in title field. Enter saves with default priority (Medium), no assignee, no due date. |
| Clicks "Apply Template" | Template dialog opens. User selects template. On confirm, POST creates all template tasks. Board populates with animation (cards fade in sequentially, 50ms stagger). |
| Clicks a task card | Inspector panel slides in from right. Task details load. Other content area narrows (desktop) or panel overlays (mobile). |
| Changes assignee in Inspector | Dropdown shows team members with avatars. Selection triggers PATCH. Assignee receives in-app notification + email (if notification preferences allow). Audit log entry. |
| Sets a dependency | Multi-select dropdown of other tasks in this opportunity. Selecting a task creates a "blocked-by" relationship. If the blocking task is not Done, the current task shows "Blocked" indicator. Circular dependency check: if selecting task X would create a cycle, show error toast "Circular dependency detected" and prevent selection. |
| Clicks linked proposal section | Navigates to Proposal Editor at the specific section anchor. Browser URL changes to `/app/opportunities/[id]/proposals/[proposalId]#section-[sectionId]`. |
| Bulk select in List View | Checkboxes enable. Toolbar appears: "X selected" + action buttons (Assign to, Set Priority, Move to, Delete). Bulk PATCH on confirm. |
| Filters by "Overdue" | Board/List shows only tasks with due date < now AND status != Done. Filter badge shows count. URL query params updated for shareability. |

### Auto-Generated Task Templates

When a user first navigates to the Tasks tab for an opportunity with zero tasks, the system checks the opportunity type and suggests the appropriate template:

**Tender Preparation Template (12 tasks)**:

| # | Task | Default Priority | Suggested Relative Due Date |
|---|------|-----------------|----------------------------|
| 1 | Review tender documents | High | Deadline - 21d |
| 2 | Complete requirements checklist | High | Deadline - 18d |
| 3 | Run bid/no-bid analysis | Critical | Deadline - 17d |
| 4 | Assign team roles | Medium | Deadline - 16d |
| 5 | Draft technical approach | High | Deadline - 12d |
| 6 | Draft financial proposal | High | Deadline - 12d |
| 7 | Compile supporting documents | Medium | Deadline - 10d |
| 8 | Internal technical review | High | Deadline - 7d |
| 9 | Legal review of contract terms | High | Deadline - 6d |
| 10 | Management sign-off | Critical | Deadline - 4d |
| 11 | Run compliance check | High | Deadline - 3d |
| 12 | Export and submit | Critical | Deadline - 1d |

Dependencies: 2 blocks 5; 3 blocks 4; 5,6 block 8; 8 blocks 9; 9 blocks 10; 10 blocks 11; 11 blocks 12.

**Grant Application Template (10 tasks)**: Similar structure adapted for grant workflows (eligibility check, budget build, logframe, consortium, etc.).

### Responsive Behavior

| Breakpoint | Behavior |
|------------|----------|
| **Desktop (>1280px)** | Full 4-column Kanban with Inspector as side panel (400px). All columns visible simultaneously. Drag-and-drop fully functional. |
| **Tablet (768-1279px)** | 4-column Kanban with horizontal scroll if needed. Columns narrower (min-width: 240px). Inspector opens as slide-over sheet (80% width). Drag-and-drop functional. |
| **Mobile (<768px)** | Default to List View (toggle still available). Board view shows one column at a time with horizontal swipe between columns. Column selector tabs appear above the board. Inspector opens as full-screen sheet. Drag-and-drop disabled; status change via dropdown in Inspector or long-press context menu. |

### Data Sources

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `GET /api/v1/opportunities/{id}/tasks` | GET | Fetch all tasks for opportunity. Supports `?status=`, `?assignee=`, `?priority=` query params. |
| `POST /api/v1/opportunities/{id}/tasks` | POST | Create task. Body: `{title, description, status, assignee_id, priority, due_date, dependencies[], linked_section_id}` |
| `PATCH /api/v1/opportunities/{id}/tasks/{taskId}` | PATCH | Update task fields (partial update). |
| `DELETE /api/v1/opportunities/{id}/tasks/{taskId}` | DELETE | Soft-delete task. |
| `POST /api/v1/opportunities/{id}/tasks/bulk` | POST | Bulk create from template. Body: `{template_id}` or `{tasks: [...]}` |
| `PATCH /api/v1/opportunities/{id}/tasks/bulk` | PATCH | Bulk update (assign, priority, status). Body: `{task_ids[], updates: {}}` |
| `PUT /api/v1/opportunities/{id}/tasks/reorder` | PUT | Update sort order within/across columns. Body: `{task_id, target_status, position}` |
| `GET /api/v1/task-templates` | GET | Fetch available task templates. |
| `GET /api/v1/companies/{id}/team-members` | GET | Fetch team members for assignee dropdowns. |

**DB tables** (new, in `client` schema): `client.tasks`, `client.task_dependencies`, `client.task_templates`, `client.task_template_items`.

### UX Requirements

- **UX-P09**: Task Board must support real-time optimistic updates -- drag-and-drop status changes reflect instantly in the UI before server confirmation. On server error, revert the card position and show an error toast. TanStack Query `useMutation` with `onMutate` optimistic update and `onError` rollback.

- **UX-P10**: Auto-generated task templates must calculate due dates relative to the opportunity submission deadline. If the deadline is less than 14 days away, compress the template schedule proportionally and show a warning banner: "Tight timeline -- some tasks overlap. Review and adjust due dates." If no deadline is set on the opportunity, leave all task due dates blank.

- **UX-P11**: Blocked tasks must be visually distinct (red left border, lock icon, "Blocked by [name]" chip) and must not be draggable to "Done" until all blocking dependencies are marked Done. Attempting to drag a blocked task to Done shows a confirmation: "This task has incomplete dependencies. Complete them first or remove the dependency."

- **UX-P12**: The Task Board must display a persistent progress summary bar above the columns showing completion percentage and per-status counts. This bar updates in real-time as tasks are moved.

---

## Screen 2: Approval Pipeline

This feature manifests in three distinct UI locations: (A) a pipeline status bar on the Proposal Detail page, (B) an approval configuration screen per opportunity, and (C) a reviewer action interface.

### Screen 2A: Pipeline Status Bar (Proposal Detail Header)

#### Route

Embedded within `/app/opportunities/[opportunityId]/proposals/[proposalId]` -- not a standalone route.

#### Access

| Tier | Access Level |
|------|-------------|
| Free / Starter | No access (proposal features gated) |
| Professional | View pipeline status. Bid Manager can configure stages. Reviewers can approve/reject their assigned stages. |
| Enterprise | Same as Professional + admin can set org-wide default templates. |
| **Roles** | Bid Manager: configure stages, assign reviewers, override/skip stages. Contributor: view status only. Reviewer: approve/reject assigned stage. Admin: all permissions. |

#### Purpose

Visual horizontal pipeline showing the current approval status of a proposal, from draft through final sign-off.

#### Layout

Positioned directly below the Proposal Detail page header (proposal title, version number, last edited), above the Tiptap editor area. Full width. Height: 80px collapsed, expandable to show stage details.

#### Components

| Component | Type | Data | Behavior |
|-----------|------|------|----------|
| **Pipeline Stepper** | Custom horizontal stepper (shadcn/ui `Steps`-like) | Ordered stages, each with: name, reviewer avatar(s), status icon, timestamp | Stages flow left-to-right. Current active stage is highlighted (blue ring). Completed stages show green check. Rejected stages show red X. Pending stages show gray circle. Skipped stages show gray dash. |
| **Stage Node** | Clickable circle + label | Stage name (e.g., "Technical Review"), reviewer avatar (24px), status badge | Click expands the stage detail card below the stepper bar. Hover shows tooltip: "[Reviewer Name] -- [Status] -- [Timestamp]". |
| **Connecting Lines** | Styled `<hr>` or SVG | — | Green (completed), blue (active/in-progress), gray (pending). Animated pulse on the active segment. |
| **Stage Detail Card** (expanded) | Expandable card below clicked stage | Reviewer name + role, status with timestamp, comments thread, attachments, approve/reject buttons (if current user is the assigned reviewer) | Slides down with animation. Only one stage detail open at a time (accordion behavior). |
| **"Configure Pipeline" Button** | `Button` (ghost, gear icon) | — | Visible only to Bid Manager / Admin. Opens Pipeline Configuration dialog. Positioned at the far right of the stepper bar. |
| **Overall Status Badge** | `Badge` | "Draft" / "In Review" / "Approved" / "Rejected" / "Ready to Submit" | Positioned at the far right of the stepper bar, summarizing the overall pipeline state. Color-coded: gray/blue/green/red/green. |

#### Interactions

| User Action | System Response |
|-------------|----------------|
| Clicks a stage node | Stage detail card expands below with reviewer info, comments, and action buttons. |
| Reviewer clicks "Approve" | Confirmation dialog: "Approve this proposal for [Stage Name]?" with optional comment field. On confirm: stage status -> Approved, next stage activates, next reviewer notified (in-app + email). Pipeline stepper updates. Audit log entry. |
| Reviewer clicks "Request Changes" | Dialog with required comment field + optional section-specific highlights. On submit: stage status -> Changes Requested, proposal author notified with specific feedback. Pipeline stepper shows amber indicator on this stage. |
| Reviewer clicks "Reject" | Dialog with required comment and severity (Minor/Major/Critical). On submit: stage status -> Rejected, pipeline halts, overall status -> Rejected. Author notified. Bid Manager can override to restart from this stage. |
| Bid Manager clicks "Skip Stage" | Confirmation dialog: "Skip [Stage Name]? This will be logged in the audit trail." On confirm: stage -> Skipped, next stage activates. Audit log records skip with user and reason. |
| Bid Manager clicks "Configure Pipeline" | Opens configuration dialog (Screen 2B). |

### Screen 2B: Approval Pipeline Configuration

#### Route

Dialog/modal accessed from the Pipeline Status Bar. No standalone route. Can also be accessed from `/app/opportunities/[opportunityId]/settings` under an "Approval Pipeline" section.

#### Purpose

Configure the approval stages, assign reviewers, and set requirements for each stage of the proposal approval workflow.

#### Components

| Component | Type | Data | Behavior |
|-----------|------|------|----------|
| **Template Selector** | `Select` dropdown | "Tender Default" (Technical -> Legal -> Management), "Grant Default" (Technical -> Financial -> Management), "Custom", "No Approval Required" | Selecting a template pre-populates stages. "Custom" starts with an empty stage list. |
| **Stage List** | Sortable list (`dnd-kit`) | Ordered list of stages, each a card | Drag handle to reorder. "+" button to add stage. "X" button to remove (with confirmation). Minimum 1 stage (unless "No Approval Required"). |
| **Stage Card** (config) | Card with form fields | Stage name (text input), Reviewer(s) (multi-select of team members), Required (toggle -- if off, stage can be skipped), Checklist items (optional, text list of things reviewer should verify), Auto-advance (toggle -- if all checklist items checked, auto-approve) | Each card is a self-contained form section. Changes are staged locally until "Save Pipeline" is clicked. |
| **"Save Pipeline" Button** | `Button` (primary) | — | Validates: at least 1 stage (unless "No Approval"), each stage has at least 1 reviewer assigned, no duplicate reviewers across consecutive stages. On save, POST/PUT pipeline config. |
| **"Reset to Default" Button** | `Button` (outline) | — | Reverts to the template default for this opportunity type. Confirmation dialog if custom changes exist. |
| **Pipeline Preview** | Mini stepper visualization | Shows the configured pipeline as it will appear | Updates in real-time as stages are added/removed/reordered. |

#### Default Templates

**Tender Default (3 stages)**:
1. Technical Review -- Reviewer role: Contributor/Technical Writer -- Checklist: Technical approach addressed, Methodology appropriate, Team composition complete, Timeline realistic
2. Legal Review -- Reviewer role: Legal Reviewer -- Checklist: Contract terms reviewed, Risk clauses flagged, IP obligations acceptable, Liability terms acceptable
3. Management Sign-Off -- Reviewer role: Bid Manager/Admin -- Checklist: Budget approved, Strategic alignment confirmed, Resource commitment confirmed

**Grant Default (3 stages)**:
1. Technical Review -- Same as tender
2. Financial Review -- Reviewer role: Financial Analyst -- Checklist: Budget compliant with EU cost categories, Co-financing calculated, Overhead rates correct, Cost justifications complete
3. Management Sign-Off -- Same as tender

### Screen 2C: Reviewer Action Interface

#### Route

Accessed via notification deep link: `/app/opportunities/[opportunityId]/proposals/[proposalId]?review_stage=[stageId]`

Also accessible by clicking the active stage in the Pipeline Status Bar.

#### Purpose

Present the proposal content alongside a review checklist and approve/reject actions for the assigned reviewer.

#### Layout

Split view: Proposal content (left, 60%) + Review panel (right, 40%). On mobile, Review panel is a bottom sheet that overlays the proposal.

#### Components

| Component | Type | Data | Behavior |
|-----------|------|------|----------|
| **Proposal Content Viewer** | Read-only Tiptap renderer | Full proposal content with section headers | Scrollable. Section headers are anchor-linked for comment references. Highlighted sections (if previous reviewer added section-specific comments) shown with amber left border + comment indicator icon. |
| **Review Panel Header** | Panel header | Stage name, "Reviewing as [Your Name]", deadline (if set) | Fixed at top of review panel. |
| **Review Checklist** | Interactive checklist (`Checkbox` list) | Checklist items from stage configuration | Each item: checkbox + label. Optional "Add note" link per item. All items must be checked before "Approve" is enabled (if Auto-advance is off). |
| **Comments Thread** | Threaded comment list | Previous stage comments + current stage comments | Each comment shows author avatar, timestamp, text, and optional section reference (clickable to scroll proposal viewer). "Add Comment" textarea at bottom. Markdown supported. |
| **Section-Specific Comment** | Inline action | — | Reviewer can select text in the proposal viewer, right-click or use toolbar button "Comment on selection", which creates a comment pinned to that section. These appear as margin annotations in the proposal viewer. |
| **Action Buttons** | Button group, fixed at bottom of review panel | "Approve" (green), "Request Changes" (amber), "Reject" (red) | Approve: enabled when all required checklist items are checked. Request Changes: always enabled, requires at least one comment. Reject: always enabled, requires comment and severity. |
| **Previous Stage Summary** | Collapsible section | Prior stages' outcomes | Shows: "[Stage name]: Approved by [Name] on [Date]" or "[Stage name]: Skipped". Collapsed by default. |

### Responsive Behavior

| Breakpoint | Behavior |
|------------|----------|
| **Desktop (>1280px)** | Pipeline stepper: full horizontal bar with all stages visible. Reviewer interface: side-by-side split (60/40). Configuration: centered dialog (640px). |
| **Tablet (768-1279px)** | Pipeline stepper: horizontal with stage labels truncated (tooltip on hover). Reviewer interface: stacked -- proposal on top (scrollable), review panel below (scrollable independently). Configuration: full-width dialog. |
| **Mobile (<768px)** | Pipeline stepper: compact -- show current stage name + overall status badge + "N of M stages complete". Tap to expand full stepper vertically (stages stack). Reviewer interface: proposal full-screen with floating "Review" FAB that opens review panel as bottom sheet (70% height). Configuration: full-screen sheet. |

### Data Sources

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `GET /api/v1/proposals/{id}/approval-pipeline` | GET | Fetch pipeline config + stage statuses for a proposal. |
| `PUT /api/v1/proposals/{id}/approval-pipeline` | PUT | Create or update pipeline config. Body: `{template_id?, stages: [{name, reviewers[], required, checklist[], auto_advance}]}` |
| `POST /api/v1/proposals/{id}/approval-pipeline/stages/{stageId}/approve` | POST | Approve a stage. Body: `{comment?, checklist_status: {}}` |
| `POST /api/v1/proposals/{id}/approval-pipeline/stages/{stageId}/request-changes` | POST | Request changes. Body: `{comment, section_references?[]}` |
| `POST /api/v1/proposals/{id}/approval-pipeline/stages/{stageId}/reject` | POST | Reject a stage. Body: `{comment, severity}` |
| `POST /api/v1/proposals/{id}/approval-pipeline/stages/{stageId}/skip` | POST | Skip a stage (Bid Manager only). Body: `{reason}` |
| `GET /api/v1/proposals/{id}/approval-pipeline/stages/{stageId}/comments` | GET | Fetch comments for a stage. |
| `POST /api/v1/proposals/{id}/approval-pipeline/stages/{stageId}/comments` | POST | Add comment. Body: `{text, section_ref?}` |
| `GET /api/v1/approval-templates` | GET | Fetch available approval pipeline templates. |

**DB tables** (new, in `client` schema): `client.approval_pipelines`, `client.approval_stages`, `client.approval_actions` (approve/reject/request-changes log), `client.approval_comments`.

### UX Requirements

- **UX-P13**: The Pipeline Status Bar must be visible at all times while editing a proposal. It must not scroll away with page content. When a stage is pending the current user's review, the stage node must pulse with a blue animation and show a "Your review needed" badge to create urgency.

- **UX-P14**: Rejection must always require a comment explaining the reason. When a proposal is rejected or changes are requested, the author must see the reviewer's comments inline in the proposal editor as margin annotations (similar to Google Docs comment UX). Clicking an annotation scrolls to and highlights the relevant section.

- **UX-P15**: Pipeline configuration must prevent invalid states: no empty reviewer assignments, no circular stage ordering, no removing a stage that is already completed. If a pipeline is reconfigured after review has started, show a warning: "Changing the pipeline will reset all pending stages. Completed stages are preserved."

- **UX-P16**: The reviewer notification (email + in-app) must include a one-click deep link that opens the Reviewer Action Interface with the review panel pre-focused. The email must include a summary: proposal title, opportunity name, deadline, and what stage they are reviewing.

---

## Screen 3: Pricing Assistant Panel

### Route

No standalone route. Side panel triggered from within the Proposal Editor at `/app/opportunities/[opportunityId]/proposals/[proposalId]`.

Panel state is reflected in URL query param: `?panel=pricing-assistant` (for deep linking and state restoration).

### Access

| Tier | Access Level |
|------|-------------|
| Free / Starter | No access (proposal features gated) |
| Professional | Full access. Usage-metered (counts as an AI analysis under tier limits). |
| Enterprise | Full access, unlimited usage. |
| **Roles** | Bid Manager, Financial Analyst: full access + insert pricing. Contributor/Technical Writer: view-only (cannot insert). Reviewer: view-only. |

### Purpose

AI-powered side panel providing competitive pricing intelligence based on historical award data, market benchmarks, and tender-specific budget ceilings, with interactive parameter adjustment and direct insertion into the proposal.

### Layout

- **Trigger**: "Pricing Assistant" button in the Proposal Editor toolbar (icon: currency/chart icon) or a floating action button in the bottom-right when the editor cursor is in a pricing-related section.
- **Panel**: Slides in from the right. Width: 480px on desktop. The Tiptap editor area narrows to accommodate. Panel has its own scroll area.
- **Panel sections**: Stacked vertically -- Market Context, Price Positioning Chart, Parameter Adjustments, Suggested Pricing Table, Insert Action.

### Components

| Component | Type | Data | Behavior |
|-----------|------|------|----------|
| **Panel Header** | Sticky header | "Pricing Assistant" title, close button (X), "Refresh Data" button (reload icon) | Close button collapses panel and restores editor to full width. Refresh re-runs the Pricing Assistant Agent with current parameters. |
| **Loading State** | Skeleton + progress text | "Analyzing market data...", "Reviewing historical awards...", "Calculating price range..." | Sequential status messages during AI agent execution (~5-10s). Skeleton placeholders for chart and table areas. |
| **Market Context Card** | `Card` | Budget ceiling (from tender docs, if available), number of historical awards analyzed, average award value, award value range (min-max), tender type/sector | Read-only informational card. Budget ceiling highlighted in amber if present. "Source: N historical awards in [CPV sector] from [date range]" footnote. |
| **Price Positioning Chart** | `Recharts` scatter/bar chart | X-axis: historical award values (sorted). Y-axis: frequency or individual awards. Overlay: colored zones (Low / Competitive / Aggressive). Draggable marker showing user's current/suggested price. Budget ceiling line (if available). | Interactive: user can drag the marker along the price axis to see where their price falls. Zones update dynamically. Tooltip on each data point: "[Contracting Authority], [Date], [Amount]". Chart height: 240px. |
| **AI Suggestion Card** | Highlighted card (blue border) | Three price points: Conservative (high win probability, lower margin), Competitive (balanced), Aggressive (lower win probability, higher margin). Each with: amount, estimated win probability %, rationale sentence. | Default selection: Competitive. User clicks a price point to select it. Selected point moves the chart marker. |
| **Parameter Adjustment Section** | Form section | **Margin target**: Slider (5%-40%, default 15%). **Risk appetite**: Select (Conservative / Balanced / Aggressive). **Cost base**: Number input (company's cost, pre-filled if available from company profile). **Competitors expected**: Number input (1-20, AI-suggested default). **Contract duration adjustment**: Toggle for multi-year normalization. | Any parameter change triggers a re-calculation (debounced 500ms). While re-calculating, chart and suggestions show a subtle loading overlay. New AI agent call with updated params. |
| **Suggested Pricing Table** | `Table` | Rows: line items from tender requirements (if structured pricing is required). Columns: Item, Quantity, Unit Price, Total, Notes. Footer: Grand Total, VAT line (if applicable), Budget utilization %. | Editable cells: user can override individual unit prices. Overrides highlighted in blue. "Reset to suggested" link per row. Grand total updates in real-time. |
| **Budget Utilization Indicator** | Progress bar + percentage | Suggested total vs. budget ceiling | Green (<80%), Amber (80-95%), Red (>95%). If no budget ceiling available, shows "Budget ceiling not specified in tender documents" with info icon. |
| **"Insert Pricing" Button** | `Button` (primary, full width, sticky at bottom of panel) | — | Inserts the pricing table as a formatted table into the Tiptap editor at the current cursor position, or replaces an existing pricing section if one is detected. Confirmation if replacing: "Replace existing pricing section?" |
| **"Export as Annex" Button** | `Button` (secondary) | — | Downloads the pricing table as a standalone XLSX file formatted for tender financial annexes. |
| **Confidence Indicator** | `Badge` + tooltip | Low / Medium / High confidence | Based on: number of historical data points (>20 = High, 5-20 = Medium, <5 = Low), data recency, sector match quality. Tooltip explains the confidence factors. |
| **Data Source Footer** | Small text | "Based on N awards from [sources] (2022-2026). Market Data Store last updated: [date]." | Transparency about data provenance. Clickable "View source awards" link opens a dialog with the full list of reference awards used. |
| **Empty State** (no data) | Illustration + message | "No historical pricing data available for this sector/region combination." | Shows: "The Pricing Assistant works best when historical award data is available. You can still set your pricing manually." CTA: "Set Pricing Manually" inserts an empty pricing table template. |

### Interactions

| User Action | System Response |
|-------------|----------------|
| Clicks "Pricing Assistant" button in toolbar | Panel slides in from right (300ms ease-out). Editor narrows. API call to Pricing Assistant Agent. Loading state shown during ~5-10s agent execution. |
| Drags price marker on chart | Real-time update: marker position, zone color, estimated win probability recalculates (client-side interpolation from AI response data). No new API call. |
| Clicks a price point (Conservative/Competitive/Aggressive) | Chart marker moves to that price. Pricing table updates with corresponding values. Selected card gets a blue ring. |
| Adjusts margin slider | Debounced 500ms, then new API call with updated parameters. Loading overlay on chart and table. Updated suggestions render. |
| Edits a cell in pricing table | Cell highlights blue (override). Grand total recalculates client-side. Budget utilization updates. Win probability estimate adjusts. |
| Clicks "Insert Pricing" | Detects if Tiptap editor has an existing `pricing-table` node. If yes, confirmation dialog to replace. On confirm, inserts/replaces formatted pricing table. Editor cursor moves to after the table. Toast: "Pricing table inserted." Panel remains open for further adjustments. |
| Clicks close (X) | Panel slides out (200ms). Editor expands to full width. Panel state preserved in memory (reopening shows previous state without re-fetching). URL query param `?panel=pricing-assistant` removed. |
| Clicks "Export as Annex" | Browser downloads `pricing_annex_[opportunity_ref].xlsx` with formatted pricing table, cover page with opportunity details, and summary row. |

### Responsive Behavior

| Breakpoint | Behavior |
|------------|----------|
| **Desktop (>1280px)** | Side panel (480px) alongside narrowed editor. Chart is interactive with hover tooltips and draggable marker. Full pricing table visible. |
| **Tablet (768-1279px)** | Side panel (400px) alongside narrowed editor. Chart slightly smaller (200px height). Pricing table scrolls horizontally if many columns. |
| **Mobile (<768px)** | Panel opens as full-screen bottom sheet (swipe down to close). Editor is hidden behind the sheet. Chart is full-width. Pricing table scrolls horizontally. "Insert Pricing" becomes a sticky footer button. User must close panel to return to editor. |

### Data Sources

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `POST /api/v1/opportunities/{id}/pricing-analysis` | POST | Trigger Pricing Assistant Agent. Body: `{cost_base?, margin_target?, risk_appetite?, competitors_expected?}`. Returns market context, price points, historical data, suggested table. |
| `GET /api/v1/opportunities/{id}/pricing-analysis/{analysisId}` | GET | Fetch cached pricing analysis results (avoid re-running agent on panel reopen). |
| `GET /api/v1/opportunities/{id}/budget-ceiling` | GET | Fetch extracted budget ceiling from parsed tender documents. |
| `GET /api/v1/market-data/awards?cpv={code}&region={code}&date_from={}&date_to={}` | GET | Fetch historical award data for the chart. Paginated. |
| `POST /api/v1/proposals/{id}/insert-content` | POST | Insert generated pricing table into proposal. Body: `{content_type: "pricing_table", position: "cursor"|"replace", data: {}}`. |

**KraftData agent**: Pricing Assistant Agent (#6 in inventory). Reads from Market Data Store (#5 in vector store inventory).

**Usage metering**: Each `POST /pricing-analysis` call increments `usage:ai_analyses:{company}:{period}` in Redis.

### UX Requirements

- **UX-P17**: The Pricing Assistant panel must show a confidence indicator reflecting the quality and quantity of underlying market data. If confidence is Low (<5 historical data points), show an amber warning: "Limited market data available. Pricing suggestions may be less reliable. Consider supplementing with your own market research."

- **UX-P18**: Parameter adjustments must use debounced re-calculation (500ms) to avoid excessive API calls. During re-calculation, the chart and suggestions must show a subtle loading overlay (not a full skeleton -- preserve the previous data as context). The user must be able to continue reading previous results while the update loads.

- **UX-P19**: The "Insert Pricing" action must detect an existing pricing table in the proposal (by Tiptap node type) and offer to replace it rather than inserting a duplicate. If the user has manually edited the pricing table in the editor after insertion, warn: "The pricing table in your proposal has been manually edited. Replacing will overwrite your changes."

---

## Screen 4: Clause Risk Flagging Panel

### Route

Tab within the Opportunity Detail page: `/app/opportunities/[opportunityId]?tab=risk-analysis`

Also accessible as a standalone route: `/app/opportunities/[opportunityId]/risk-analysis`

### Access

| Tier | Access Level |
|------|-------------|
| Free | No access (gated behind paywall with upgrade prompt showing sample risk flags) |
| Starter | View AI-generated risk summary (limited: top 3 risks only, no full clause text). Upgrade prompt for full analysis. Usage-metered. |
| Professional | Full access: all flagged clauses, full text, AI explanations, actions. Usage-metered. |
| Enterprise | Full access, unlimited. |
| **Roles** | Bid Manager: full access + assign to reviewer. Legal Reviewer: full access + resolve/acknowledge. Contributor: view-only. Read-only: view-only. |

### Purpose

AI-powered contract risk analysis that identifies, categorizes, and explains high-risk clauses extracted from tender documents, with workflow actions for legal review and resolution tracking.

### Layout

- **App shell**: Sidebar with "Opportunities" selected. Opportunity Detail page with tab bar.
- **Tab bar**: Analysis | Requirements | **Risk Analysis** | Proposals | Tasks | Timeline
- **Content area**: Risk Summary header card at top, followed by filterable clause list below.

### Components

| Component | Type | Data | Behavior |
|-----------|------|------|----------|
| **Risk Summary Card** | Full-width card at top | Overall risk score (0-100, with label: Low/Medium/High/Critical), risk distribution bar (count per severity), comparison text ("This tender's risk level is [higher/lower/typical] compared to similar [type] tenders"), last analyzed timestamp, "Re-analyze" button | Risk score uses a circular gauge (Recharts `RadialBarChart`). Distribution bar is segmented (Critical: red, High: orange, Medium: yellow, Low: green). |
| **"Analyze Contract Risks" Button** | `Button` (primary, prominent) | — | Shown only when no analysis exists yet. Triggers Clause Risk Analyzer Agent. Replaced by "Re-analyze" (secondary, in Summary Card) after first analysis. |
| **Analysis Status Banner** | `Banner` / `Alert` | "Analysis in progress... Scanning [N] documents." or "Analysis complete. [N] clauses flagged." or "No tender documents uploaded. Upload documents to enable risk analysis." | Shown during and after analysis. Auto-dismisses after 10s on completion. Persistent if no documents available. |
| **Filter Bar** | Horizontal bar with pills/dropdowns | Severity (Critical/High/Medium/Low), Category (Liability, IP, Penalty, Payment Terms, Termination, Confidentiality, Insurance, Indemnity), Status (New/Acknowledged/Under Review/Resolved), Assigned To | Active filters shown as removable pills. "Clear all" link. Result count updates: "Showing N of M flagged clauses." |
| **Sort Control** | `Select` dropdown | Sort by: Severity (default, desc), Category, Document Order, Status, Date Flagged | Persistent preference. |
| **Clause Card** | `Card` (list item) | Severity badge (color-coded), Category tag, Clause title/summary (AI-generated 1-line summary), Source reference ("Document: [filename], Page [N], Section [ref]"), Status badge | Card has expandable detail section. Left border color matches severity (red/orange/yellow/green). New/unreviewed clauses have a subtle blue background tint. |
| **Clause Detail** (expanded) | Expandable section within card | **Extracted Text**: blockquote with the exact clause text from the document (monospace, gray background, with quotation marks). **AI Explanation**: paragraph explaining why this clause is risky, in plain language. **Recommended Action**: badge (Negotiate / Accept with Reservation / Reject / Seek Legal Review) + explanation. **Comparable Clauses**: "In similar tenders, this clause typically reads: [typical text]" (if available). **Notes**: thread of user-added notes with timestamps. **History**: status change log. | Expand/collapse on card click or chevron. Only one card expanded at a time (accordion) or allow multiple (toggle in settings). |
| **Clause Action Bar** (within expanded card) | Button group | "Acknowledge" (gray->blue), "Assign to Legal" (opens reviewer picker), "Mark Resolved" (green, with required resolution note), "Add Note" (opens note input) | Status transitions: New -> Acknowledged -> Under Review (when assigned) -> Resolved. Cannot skip to Resolved from New (must acknowledge first). |
| **Assignee Picker** (inline) | `Popover` with team member list | Team members with Legal Reviewer role prioritized at top | On assign, sends in-app notification to assignee. Assignee avatar appears on the clause card. |
| **Export Risk Report Button** | `Button` (outline, in Summary Card) | — | Generates a PDF report: cover page (opportunity name, date, overall risk score), executive summary, full clause listing with AI explanations and recommendations, appendix with raw clause text. Download as `risk_report_[opportunity_ref].pdf`. |
| **Bulk Actions Bar** | Toolbar (appears when clauses selected) | Checkboxes on clause cards | Select all / select none. Actions: Bulk Acknowledge, Bulk Assign, Bulk Mark Resolved. |
| **Empty State** (no documents) | Illustration + CTA | "Upload tender documents to analyze contract risks." | Button: "Upload Documents" (navigates to Documents tab). |
| **Empty State** (analysis complete, no risks) | Illustration + message | "No significant risk clauses identified." | Shows: "The AI analyzed [N] pages across [M] documents and found no unusual or high-risk clauses. This is a positive signal, but always have legal counsel review contract terms before signing." Green checkmark illustration. |
| **Document Source Link** | Inline link in clause card | Document filename, page number | Clicking opens the document viewer (PDF viewer or document detail page) scrolled to the referenced page. Opens in a new tab or split view. |

### Interactions

| User Action | System Response |
|-------------|----------------|
| Clicks "Analyze Contract Risks" | API call to Clause Risk Analyzer Agent. Banner shows progress. Agent analyzes all uploaded tender documents for this opportunity. On completion (~10-30s depending on document volume), clause cards populate with animation (fade-in, stagger). Summary Card renders risk score. Audit log entry. Usage meter incremented. |
| Clicks "Re-analyze" | Confirmation: "Re-analyze will refresh all risk flags. Existing status (Acknowledged/Resolved) will be preserved for matching clauses. New clauses may appear." On confirm, re-runs agent. Diff against previous: new clauses marked "NEW", removed clauses marked "No longer flagged", existing clauses preserve status. |
| Expands a clause card | Detail section slides down (200ms). AI explanation, extracted text, recommended action, notes, and action bar render. Scroll position adjusts to keep expanded card visible. |
| Clicks "Acknowledge" | Status badge changes New -> Acknowledged. Blue tint removed from card. Timestamp logged. Toast: "Clause acknowledged." |
| Clicks "Assign to Legal" | Popover with team members. Legal Reviewer role members shown first with a "Legal" badge. On select, status -> Under Review, assignee avatar appears on card. Assignee receives notification with deep link to this clause. |
| Clicks "Mark Resolved" | Dialog with required "Resolution note" textarea and resolution type dropdown (Accepted / Negotiated / Removed / Not Applicable). On submit, status -> Resolved. Card collapses and moves to bottom of list (or filtered out if Status filter excludes Resolved). |
| Clicks document source link | Opens document viewer in new tab, scrolled to the referenced page. If page extraction is approximate, highlights the approximate region. |
| Clicks "Export Risk Report" | Loading toast: "Generating risk report..." Download triggers in 2-5s. PDF includes all flagged clauses with their current status, AI explanations, and any resolution notes. |

### Responsive Behavior

| Breakpoint | Behavior |
|------------|----------|
| **Desktop (>1280px)** | Summary Card full-width. Clause cards in a single column (full-width) for readability of clause text. Filter bar horizontal. |
| **Tablet (768-1279px)** | Same layout, slightly narrower clause text. Filter bar wraps to two rows if needed. |
| **Mobile (<768px)** | Summary Card stacks gauge and stats vertically. Clause cards are full-width with truncated summaries (expand to see full text). Filter bar becomes a "Filter" button opening a bottom sheet with all filter options. Export button moves to a "..." overflow menu. Bulk selection via long-press. |

### Data Sources

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `POST /api/v1/opportunities/{id}/risk-analysis` | POST | Trigger Clause Risk Analyzer Agent. Returns analysis ID for polling. |
| `GET /api/v1/opportunities/{id}/risk-analysis` | GET | Fetch latest risk analysis results: summary + flagged clauses. Supports `?severity=`, `?category=`, `?status=` filters. |
| `GET /api/v1/opportunities/{id}/risk-analysis/clauses/{clauseId}` | GET | Fetch full detail for a single clause (expanded view data). |
| `PATCH /api/v1/opportunities/{id}/risk-analysis/clauses/{clauseId}` | PATCH | Update clause status, assignee, or notes. Body: `{status?, assignee_id?, resolution_note?, resolution_type?}`. |
| `POST /api/v1/opportunities/{id}/risk-analysis/clauses/{clauseId}/notes` | POST | Add a note to a clause. Body: `{text}`. |
| `PATCH /api/v1/opportunities/{id}/risk-analysis/clauses/bulk` | PATCH | Bulk update clauses. Body: `{clause_ids[], updates: {status?, assignee_id?}}`. |
| `GET /api/v1/opportunities/{id}/risk-analysis/export` | GET | Generate and download PDF risk report. Returns file stream. |
| `GET /api/v1/opportunities/{id}/documents` | GET | Fetch list of uploaded tender documents (for source links). |

**KraftData agent**: Clause Risk Analyzer Agent (#4 in inventory).

**DB tables** (new, in `client` schema): `client.risk_analyses`, `client.risk_clauses`, `client.risk_clause_notes`.

### UX Requirements

- **UX-P20**: Risk severity badges must use consistent, accessible color coding across the entire platform: Critical (red-600, white text), High (orange-500, white text), Medium (yellow-500, black text), Low (green-600, white text). Colors must meet WCAG 2.1 AA contrast requirements. Additionally, each severity level must have a distinct icon (not just color) for color-blind accessibility: Critical (octagon-stop), High (triangle-warning), Medium (circle-info), Low (shield-check).

- **UX-P21**: When tender documents are uploaded or updated for an opportunity, the Risk Analysis tab must show a notification badge: "Documents changed since last analysis. Re-analyze recommended." This prompt appears as a yellow info banner at the top of the tab content, with a one-click "Re-analyze now" button.

- **UX-P22**: The AI explanation for each flagged clause must be written in plain business language, not legal jargon. It must explain: (1) what the clause means in practice, (2) why it poses a risk to the bidder, and (3) what is typical in similar contracts. This requirement applies to the Clause Risk Analyzer Agent prompt design, but the UI must allocate sufficient space (minimum 3 lines visible before truncation with "Read more" expand).

- **UX-P23**: Clause status workflow must enforce sequencing (New -> Acknowledged -> Under Review -> Resolved) and prevent skipping steps without Bid Manager/Admin override. Status changes must be timestamped and attributed to the acting user in the clause history log. The history log is visible in the expanded clause detail.

---

## Screen 5: Requirements Checklist Panel

### Route

Tab within the Opportunity Detail page: `/app/opportunities/[opportunityId]?tab=requirements`

Also accessible as a standalone route: `/app/opportunities/[opportunityId]/requirements`

### Access

| Tier | Access Level |
|------|-------------|
| Free | No access (gated with upgrade prompt showing sample requirement count) |
| Starter | View auto-generated checklist (read-only, no status updates or assignments). Usage-metered (counts as an AI analysis). |
| Professional | Full access: edit status, assign, link to proposal sections, add notes. Usage-metered. |
| Enterprise | Full access, unlimited. Bulk import/export. |
| **Roles** | Bid Manager: full CRUD + bulk operations + export. Contributor: edit status and notes on assigned items. Reviewer: view-only. Read-only: view-only. Admin: full CRUD + delete. |

### Purpose

Auto-generated interactive compliance checklist mapping every mandatory requirement extracted from tender documents to a trackable item with status, assignment, proposal section linkage, and completion tracking.

### Layout

- **App shell**: Sidebar with "Opportunities" selected. Opportunity Detail page with tab bar.
- **Tab bar**: Analysis | **Requirements** | Risk Analysis | Proposals | Tasks | Timeline
- **Content area**: Progress header at top, filter bar below, then the checklist table/list.

### Components

| Component | Type | Data | Behavior |
|-----------|------|------|----------|
| **Progress Header** | Full-width card | Completion percentage (circular gauge or horizontal bar), counts by status (Not Started / In Progress / Complete / N/A), estimated completeness ("23 of 47 requirements addressed"), last updated timestamp | Progress bar is segmented by status: Not Started (gray), In Progress (blue), Complete (green), N/A (light gray). Updates in real-time as items change status. |
| **"Generate Checklist" Button** | `Button` (primary, prominent) | — | Shown only when no checklist exists. Triggers Requirement Checklist Agent. Replaced by "Refresh Checklist" (secondary) after first generation. |
| **Refresh Banner** | `Alert` (info) | "Tender documents have been updated since this checklist was generated. Refresh to capture new requirements." | Appears when document upload timestamp > checklist generation timestamp. One-click "Refresh now" button. |
| **Filter Bar** | Horizontal bar | Category (Administrative/Technical/Financial/Legal/Qualification), Status (Not Started/In Progress/Complete/N/A), Assignee, Search (free text search across requirement text) | Composable AND filters. Active filter count as badge. "Clear all" link. Result count: "Showing N of M requirements." |
| **Category Tabs** (alternative filter) | `Tabs` component | All | Administrative | Technical | Financial | Legal | Qualification | Quick filter alternative to dropdown. Each tab shows count badge. "All" tab shows aggregate. Can be combined with other filters. |
| **Requirements Table** | `Table` with expandable rows | Columns: Checkbox (for bulk), #, Requirement (text, truncated), Category (badge), Status (dropdown), Linked Section (link), Assignee (avatar), Actions (overflow menu) | Sortable columns. Rows alternate subtle background. Category badges color-coded: Administrative (blue), Technical (purple), Financial (green), Legal (orange), Qualification (gray). |
| **Requirement Row** (collapsed) | Table row | Requirement number (auto-generated), requirement text (first 120 chars + "..."), category badge, status dropdown (inline edit), linked proposal section name (clickable), assignee avatar (clickable to change) | Click row to expand. Inline status change via dropdown (no dialog needed). |
| **Requirement Row** (expanded) | Expanded section below row | **Full requirement text** (complete extracted text with source reference: "From: [document], Page [N], Section [ref]"), **Notes** (editable textarea, auto-saves on blur), **Linked proposal section** (dropdown of proposal sections, or "Not linked" with "Auto-link" suggestion button), **Assignee** (team member picker), **History** (status change log with timestamps) | Expand/collapse on row click. Only one expanded at a time (or multiple, configurable). |
| **Proposal Section Link** | Clickable link + dropdown | Linked section name (e.g., "3.1 Technical Approach") | Click link: navigates to proposal editor scrolled to that section. Dropdown to change link. "Auto-link" button: AI suggests the best matching proposal section. If no proposal exists yet, shows "Create proposal first" tooltip. |
| **Auto-Link Suggestions** | AI-powered suggestion UI | For each requirement, AI-suggested proposal section mapping | When a proposal is generated, a banner appears: "AI has suggested section mappings for [N] requirements. Review suggestions?" On click, shows a dialog with each requirement + suggested section + Accept/Reject per row + "Accept All" button. |
| **Bulk Actions Bar** | Toolbar (appears when checkboxes selected) | — | Actions: Bulk Set Status, Bulk Assign, Bulk Set Category, Bulk Mark N/A. Appears as a floating bar at bottom of viewport. "[N] selected" + action buttons. |
| **Add Requirement Button** | `Button` (outline, "+") | — | Adds a manual requirement row at the bottom of the list. For requirements missed by AI extraction. Fields: text (required), category (required), source reference (optional). |
| **Export Button** | `Button` (outline, in Progress Header) | — | Dropdown: "Export as PDF" (formatted checklist with status, assignees, completion %), "Export as XLSX" (spreadsheet with all fields, suitable for import/editing). |
| **Import Button** | `Button` (outline, Enterprise only) | — | Upload XLSX with requirements to merge/replace. Confirmation dialog showing diff (new, modified, removed items). |
| **Empty State** (no documents) | Illustration + CTA | "Upload tender documents to auto-generate the requirements checklist." | Button: "Upload Documents". Secondary text: "Or add requirements manually." |
| **Empty State** (no requirements found) | Illustration + message | "No mandatory requirements identified in the uploaded documents." | Shows: "The AI analyzed [N] pages but couldn't extract specific requirements. This may happen with loosely structured tender documents. You can add requirements manually." CTA: "Add Requirement Manually". |
| **Requirement Source Link** | Inline link in expanded row | Document filename + page + section | Clicking opens document viewer at the referenced page (same as Risk Analysis source links). |

### Interactions

| User Action | System Response |
|-------------|----------------|
| Clicks "Generate Checklist" | Triggers Requirement Checklist Agent. Loading state: "Extracting requirements from [N] documents..." with progress stages ("Scanning administrative requirements...", "Scanning technical requirements...", etc.). On completion (~10-30s), checklist populates. Progress Header renders. Usage meter incremented. Audit log entry. |
| Changes status via inline dropdown | Optimistic update. PATCH request. Status badge color changes immediately. Progress Header updates (count and percentage recalculate). If changing to "Complete", a subtle green flash on the row. If changing from "Complete" to something else, progress decreases (no confirmation needed). |
| Clicks linked proposal section | Navigates to `/app/opportunities/[id]/proposals/[proposalId]#section-[sectionId]`. Editor scrolls to and briefly highlights (yellow flash) the linked section. Back button returns to requirements tab. |
| Clicks "Auto-link" on a requirement | Small loading spinner. AI suggests the best matching section from the current proposal. On completion, suggestion appears as a chip: "[Section name] -- Accept / Reject". Accept sets the link. Reject clears the suggestion. |
| Reviews auto-link suggestions (batch) | Dialog: table with columns: Requirement (text), Suggested Section, Confidence (High/Medium/Low), Accept (checkbox, default checked for High confidence). "Accept Selected" button at bottom. On confirm, all accepted links are saved in one PATCH call. |
| Clicks "Export as PDF" | Loading toast. PDF generates server-side: cover page (opportunity details, generation date, overall completion %), table of requirements grouped by category, each with status/assignee/notes. Download triggers. |
| Clicks "Export as XLSX" | Immediate download. Columns: #, Requirement Text, Category, Status, Assignee, Linked Section, Notes, Source Reference. One sheet per category + summary sheet. |
| Selects multiple rows + "Bulk Set Status" | Dropdown appears: Not Started / In Progress / Complete / N/A. On select, all checked rows update. Progress Header recalculates. Toast: "[N] requirements updated to [status]." |
| Clicks "Add Requirement" | New row appears at bottom of list in edit mode. Title field auto-focused. Category dropdown required. Enter saves. Row gets a "Manual" source badge (vs "AI-extracted" for auto-generated). |
| Clicks "Refresh Checklist" | Confirmation: "Refreshing will re-analyze documents. Existing requirements will be preserved -- new ones added, removed ones flagged. Statuses and notes are not affected." On confirm, agent re-runs. Diff applied: new requirements appear with "NEW" badge, requirements no longer found show "Removed from source" badge (user can keep or delete). |
| Text search in filter bar | Debounced 300ms. Filters requirements whose text contains the search term (case-insensitive). Matching text is highlighted in yellow in the requirement text column. |

### Responsive Behavior

| Breakpoint | Behavior |
|------------|----------|
| **Desktop (>1280px)** | Full table layout with all columns visible. Expanded rows show all fields inline. Bulk action bar floats at bottom. Category tabs horizontal. |
| **Tablet (768-1279px)** | Table collapses "Assignee" and "Actions" columns into the expanded row view. Category tabs scroll horizontally. Bulk action bar floats at bottom. |
| **Mobile (<768px)** | Table transforms to card list. Each requirement is a card showing: number, text (truncated), category badge, status badge. Tap card to expand (full text, notes, assignee, linked section). Filter bar becomes "Filter" button opening a bottom sheet. Bulk selection via long-press. Export and Import in overflow menu. Category tabs become a horizontal scroll strip. Progress Header shows percentage and bar only (no per-status counts; tap to expand). |

### Data Sources

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `POST /api/v1/opportunities/{id}/requirements` | POST | Trigger Requirement Checklist Agent. Returns analysis ID for polling or immediate result. |
| `GET /api/v1/opportunities/{id}/requirements` | GET | Fetch all requirements. Supports `?category=`, `?status=`, `?assignee=`, `?search=` query params. Returns items + summary stats. |
| `GET /api/v1/opportunities/{id}/requirements/{reqId}` | GET | Fetch single requirement with full detail (expanded view data). |
| `PATCH /api/v1/opportunities/{id}/requirements/{reqId}` | PATCH | Update requirement fields. Body: `{status?, assignee_id?, linked_section_id?, notes?, category?}`. |
| `POST /api/v1/opportunities/{id}/requirements/manual` | POST | Add a manual requirement. Body: `{text, category, source_ref?}`. |
| `DELETE /api/v1/opportunities/{id}/requirements/{reqId}` | DELETE | Soft-delete a requirement (Bid Manager/Admin only). |
| `PATCH /api/v1/opportunities/{id}/requirements/bulk` | PATCH | Bulk update. Body: `{requirement_ids[], updates: {status?, assignee_id?, category?}}`. |
| `POST /api/v1/opportunities/{id}/requirements/auto-link` | POST | Trigger AI auto-linking for all unlinked requirements against proposal sections. Returns `{suggestions: [{req_id, section_id, confidence}]}`. |
| `POST /api/v1/opportunities/{id}/requirements/{reqId}/auto-link` | POST | Trigger AI auto-linking for a single requirement. |
| `GET /api/v1/opportunities/{id}/requirements/export?format=pdf|xlsx` | GET | Generate and download export file. |
| `POST /api/v1/opportunities/{id}/requirements/import` | POST | Import from XLSX. Multipart upload. Returns diff preview before applying. |
| `GET /api/v1/proposals/{proposalId}/sections` | GET | Fetch proposal sections for the "Linked Section" dropdown. |

**KraftData agent**: Requirement Checklist Agent (#18 in inventory).

**DB tables** (new, in `client` schema): `client.requirement_checklists`, `client.requirement_items`, `client.requirement_section_links`.

### UX Requirements

- **UX-P24**: The requirements checklist must show a persistent progress indicator at the top of the tab that updates in real-time. The progress bar must be segmented by status (Not Started: gray, In Progress: blue, Complete: green, N/A: light gray) and show both the percentage and absolute count ("23 of 47 complete -- 49%"). This provides at-a-glance bid readiness assessment.

- **UX-P25**: Auto-linking between requirements and proposal sections must use a confidence indicator (High/Medium/Low) for each suggestion. High-confidence matches (>90%) should be auto-accepted by default in the batch dialog. Medium and Low confidence suggestions require manual review. The user must always have the option to override or reject any AI-suggested link.

- **UX-P26**: When refreshing the checklist after document changes, the system must preserve all user-entered data (statuses, assignments, notes, manual requirements) and present a diff view before applying changes. New requirements appear with a "NEW" badge. Requirements no longer found in source documents appear with a "Removed from source" badge but are NOT automatically deleted -- the user must explicitly choose to remove them.

- **UX-P27**: The checklist must support manual additions to account for requirements the AI may have missed. Manual requirements are visually distinguished with a "Manual" source badge (vs "AI-extracted") but are otherwise treated identically in progress tracking, export, and assignment workflows.

- **UX-P28**: Clicking a linked proposal section from the requirements checklist must navigate to the proposal editor and scroll to the exact section, with a brief highlight animation (yellow flash, 1.5s fade). A floating "Back to Requirements" button must appear in the editor to enable quick return navigation without losing context.

---

## UX Requirements Summary

All new requirements surfaced by these screen specifications, continuing from the existing UX-P08:

| ID | Requirement | Screen | Feature |
|---|---|---|---|
| UX-P09 | Optimistic drag-and-drop updates with error rollback | Task Board | FR31 |
| UX-P10 | Template due dates relative to deadline with compression warning | Task Board | FR31 |
| UX-P11 | Blocked task visual distinction and Done-column guard | Task Board | FR31 |
| UX-P12 | Persistent progress summary bar above Kanban columns | Task Board | FR31 |
| UX-P13 | Sticky pipeline bar with pulse animation for pending reviews | Approval Pipeline | FR35 |
| UX-P14 | Rejection requires comment; inline margin annotations in editor | Approval Pipeline | FR35 |
| UX-P15 | Pipeline config validation preventing invalid states | Approval Pipeline | FR35 |
| UX-P16 | One-click reviewer notification deep link with summary context | Approval Pipeline | FR35 |
| UX-P17 | Pricing confidence indicator with low-data warning | Pricing Assistant | FR17 |
| UX-P18 | Debounced parameter re-calculation with subtle loading overlay | Pricing Assistant | FR17 |
| UX-P19 | Detect existing pricing table; warn before overwrite on insert | Pricing Assistant | FR17 |
| UX-P20 | Accessible severity badges with distinct icons for color-blind users | Clause Risk Flagging | FR10 |
| UX-P21 | Document-change notification with re-analyze prompt | Clause Risk Flagging | FR10 |
| UX-P22 | Plain-language AI explanations with minimum 3-line visible space | Clause Risk Flagging | FR10 |
| UX-P23 | Enforced status workflow sequencing with timestamped history | Clause Risk Flagging | FR10 |
| UX-P24 | Persistent segmented progress bar with real-time updates | Requirements Checklist | FR9 |
| UX-P25 | Confidence-based auto-link suggestions with manual override | Requirements Checklist | FR9 |
| UX-P26 | Diff-based checklist refresh preserving user data | Requirements Checklist | FR9 |
| UX-P27 | Manual requirement additions with "Manual" source badge | Requirements Checklist | FR9 |
| UX-P28 | Cross-navigation to proposal section with highlight and back button | Requirements Checklist | FR9 |

---

## New Database Tables Summary

All tables belong to the `client` schema and are owned by the Client API service.

| Table | Screen | Purpose |
|-------|--------|---------|
| `client.tasks` | Task Board | Task records (title, description, status, priority, assignee, due date, sort order, opportunity FK, proposal section FK) |
| `client.task_dependencies` | Task Board | Dependency edges (blocked_task_id, blocking_task_id) |
| `client.task_templates` | Task Board | Template definitions (name, type: tender/grant/framework) |
| `client.task_template_items` | Task Board | Template line items (template FK, title, default priority, relative due offset, dependencies) |
| `client.approval_pipelines` | Approval Pipeline | Pipeline config per proposal (proposal FK, template, status) |
| `client.approval_stages` | Approval Pipeline | Stage definitions (pipeline FK, name, order, reviewers, required, checklist, auto_advance) |
| `client.approval_actions` | Approval Pipeline | Action log (stage FK, actor, action type, comment, timestamp) |
| `client.approval_comments` | Approval Pipeline | Review comments (stage FK, author, text, section_ref, timestamp) |
| `client.risk_analyses` | Clause Risk Flagging | Analysis run records (opportunity FK, risk score, timestamp, status) |
| `client.risk_clauses` | Clause Risk Flagging | Flagged clauses (analysis FK, severity, category, text, explanation, recommendation, status, assignee) |
| `client.risk_clause_notes` | Clause Risk Flagging | Notes on clauses (clause FK, author, text, timestamp) |
| `client.requirement_checklists` | Requirements Checklist | Checklist header (opportunity FK, generation timestamp, document hash) |
| `client.requirement_items` | Requirements Checklist | Individual requirements (checklist FK, text, category, status, assignee, source ref, sort order) |
| `client.requirement_section_links` | Requirements Checklist | Linkage table (requirement FK, proposal section FK, confidence, auto_linked boolean) |agentId: a792c88b628699dd2 (use SendMessage with to: 'a792c88b628699dd2' to continue this agent)
<usage>total_tokens: 62095
tool_uses: 21
duration_ms: 822171</usage>


---


# EU Solicit — Screen Specifications (Screens 6–10)

---

## SCREEN 6: Reusable Content Blocks Library

**Route:** `/content-library`

**Access:** All authenticated users. Free tier limited to 10 blocks. Professional tier unlimited. Enterprise tier adds organization-wide shared library. Approval workflow available on Professional+.

**Purpose:** Manage a searchable library of approved boilerplate content blocks that can be inserted into proposals and grant applications, ensuring consistency and reducing repetitive writing.

**Layout:** Standard app shell — collapsible sidebar (with "Content Library" nav item under Tools group) + main content area. Optional inspector panel slides in from right when a block is selected for preview. When invoked from the proposal editor via "Insert Block," the library renders as a side panel (480px) docked to the right of the editor.

---

### Components

#### 6.1 Page Header

| Element | Details |
|---|---|
| Title | "Content Library" — `h1`, `text-2xl font-semibold` |
| Subtitle | Block count: "47 blocks · 12 approved" — `text-sm text-muted-foreground` |
| Primary action | `Button` "New Block" — opens creation modal/page. Icon: `Plus` |
| Secondary action | `Button` variant `outline` "Import" — dropdown with "Paste from Proposal" and "Upload DOCX" |
| View toggle | `ToggleGroup` with `LayoutGrid` (grid) and `List` (list) icons. Persisted in user preferences via Zustand |

#### 6.2 Toolbar — Search & Filters

| Element | Details |
|---|---|
| Search input | `Input` with `Search` icon prefix. Placeholder: "Search blocks by title, content, or keyword…". Debounced 300ms. Searches title, body text, and tags via API full-text search |
| Category filter | `Select` multi-select dropdown. Options: Company Overview, Quality Management, Sustainability, Team CVs, Methodology, Past Projects, Financial Capability, Legal Compliance, Custom. Chip display for active filters |
| Status filter | `Select` single-select. Options: All, Draft, Approved, Archived |
| Author filter | `Select` single-select. Shows team members. Only visible when organization has >1 user |
| Sort | `Select` single-select. Options: Last Updated (default), Title A–Z, Title Z–A, Word Count, Date Created |
| Clear filters | `Button` variant `ghost` "Clear all" — visible only when filters are active |

#### 6.3 Content Blocks — Grid View (Default)

Each block rendered as a `Card` in a responsive grid.

| Element | Details |
|---|---|
| Grid layout | `grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-4` |
| Card dimensions | Min-height 200px. Max content preview 3 lines |
| Status indicator | Top-right corner: `Badge` — "Approved" (`variant="default"`, green) or "Draft" (`variant="secondary"`, gray) or "Archived" (`variant="outline"`, muted) |
| Category tag | Top-left: `Badge` variant `outline` with category color. Colors: Company Overview = blue, Quality Management = emerald, Sustainability = green, Team CVs = violet, Methodology = amber, Past Projects = slate, Financial Capability = cyan, Legal Compliance = red, Custom = gray |
| Title | `text-sm font-medium truncate` — single line, ellipsis overflow |
| Preview excerpt | `text-xs text-muted-foreground line-clamp-3` — first 150 chars of plain-text content, stripped of formatting |
| Metadata row | Bottom of card: `text-xs text-muted-foreground`. Shows: author avatar (16px) + name, word count, last updated (relative time via `formatDistanceToNow`) |
| Hover state | `ring-2 ring-ring` border. Slight `shadow-md` elevation. Cursor pointer |
| Context menu | Right-click or `...` `DropdownMenu` button (top-right, appears on hover): Edit, Duplicate, Archive/Unarchive, Delete, View History |
| Selection | Click opens the block in the inspector panel (preview mode). Double-click opens in full editor |

#### 6.4 Content Blocks — List View

| Column | Width | Details |
|---|---|---|
| Status icon | 32px | Colored dot: green (Approved), gray (Draft), muted (Archived) |
| Title | flex-1 | `text-sm font-medium`. Clickable — opens inspector |
| Category | 140px | `Badge` with category tag |
| Author | 120px | Avatar + name, `text-sm` |
| Word Count | 80px | `text-sm text-muted-foreground`, right-aligned |
| Last Updated | 120px | Relative time, `text-sm text-muted-foreground` |
| Actions | 40px | `DropdownMenu` with same options as grid context menu |

Rendered using shadcn `Table` with sticky header. Sortable columns (click header to sort). Row hover: `bg-muted/50`.

#### 6.5 Inspector Panel (Preview)

Slides in from right when a block is selected. Width: 400px desktop, full-screen overlay on mobile.

| Element | Details |
|---|---|
| Header | Block title — `text-lg font-semibold`. Close button (`X` icon) |
| Status badge | `Badge` showing Draft/Approved |
| Metadata | Category, author, created date, last updated, word count, version number |
| Content preview | Rendered rich text (read-only Tiptap instance) with full formatting |
| Action bar | Bottom-fixed: `Button` "Edit" (primary), `Button` "Duplicate" (outline), `Button` "Insert" (only when opened from proposal editor, primary with `Plus` icon) |
| Version history | `Accordion` section "Version History" — list of versions with date, author, change summary. Click to preview a prior version. `Button` "Restore" to revert |

#### 6.6 Block Editor (Create / Edit)

Opens as a full-page view (`/content-library/new` or `/content-library/{blockId}/edit`) or as a modal depending on context.

| Element | Details |
|---|---|
| Title input | `Input` large, placeholder "Block title…". Auto-focus on create. `text-xl font-semibold` styling |
| Category selector | `Select` single-select, required. Same category options as filter |
| Status toggle | `Switch` labeled "Approved" — toggling on sets status to Approved. Only users with `content:approve` permission can toggle to Approved. Toggling off reverts to Draft |
| Rich text editor | Tiptap editor instance. Toolbar: Bold, Italic, Underline, Strikethrough | Heading 1/2/3 | Bullet list, Numbered list | Blockquote | Link | Table | Undo/Redo. Min-height 400px. Supports paste from Word/Google Docs with formatting preservation |
| Word count | Bottom-left of editor: live word count + character count. `text-xs text-muted-foreground` |
| Tags input | `Input` with autocomplete for free-form tags (for search enhancement). Comma-separated |
| Save actions | `Button` "Save" (primary) — saves and returns to library. `Button` "Save & Continue Editing" (outline). Auto-save every 30 seconds with `toast` notification "Draft saved" |
| Cancel | `Button` variant `ghost` "Cancel" — prompts unsaved changes dialog if dirty |
| Delete | `Button` variant `destructive` "Delete Block" — bottom of form, with confirmation `AlertDialog` |

#### 6.7 Import Flows

**Paste from Proposal:**

| Step | Details |
|---|---|
| Trigger | "Import > Paste from Proposal" from page header |
| Dialog | `Dialog` with `Select` to choose an existing proposal, then a list of detected sections (parsed by heading structure). User checks sections to import |
| Result | Each checked section creates a new block in Draft status with title derived from section heading |

**Upload DOCX:**

| Step | Details |
|---|---|
| Trigger | "Import > Upload DOCX" from page header |
| Dialog | `Dialog` with drag-and-drop zone (`react-dropzone`). Accepts `.docx` files up to 10MB |
| Processing | Server parses DOCX via `mammoth.js`, splits by headings into candidate blocks. Shows preview list |
| Result | User reviews, edits titles/categories, deselects unwanted sections, confirms import. Creates blocks in Draft status |

#### 6.8 Insert Block Panel (Proposal Editor Integration)

When working in the proposal Tiptap editor, a slash command `/insert-block` or toolbar button "Insert Block" opens the Content Library as a docked side panel.

| Element | Details |
|---|---|
| Panel | 480px wide, docked right of editor. Overlays inspector if open |
| Search | Same search input as main library, scoped to Approved blocks only (unless user is viewing their own Draft blocks) |
| Filters | Category filter only (compact) |
| Block list | Compact card list — title, category badge, word count, preview (2 lines). Click to preview in-panel |
| Insert action | `Button` "Insert" on each card, or "Insert" in preview. Inserts block content at cursor position in editor |
| Link behavior | Inserted content is wrapped in a custom Tiptap node (`content-block-ref`) that stores `blockId` and `versionId`. Visual indicator: faint left border (2px, `border-primary/20`) and a small `Badge` "Linked: {block title}" above the inserted content |
| Update flow | When a source block is updated, proposals referencing it show a `toast`-style inline notification within the editor: "Content block '{title}' has been updated. [Review Changes] [Update] [Dismiss]". "Review Changes" opens a diff view. "Update" replaces content. "Dismiss" keeps current version and unlinks |

#### 6.9 Empty States

| State | Display |
|---|---|
| No blocks exist | Illustration (document icon) + "Your content library is empty" + "Create reusable blocks for company overviews, team CVs, methodology descriptions, and more." + `Button` "Create First Block" + `Button` variant `outline` "Import from DOCX" |
| No search results | "No blocks matching '{query}'" + "Try different keywords or clear filters" + `Button` "Clear Filters" |
| Tier limit reached (Free) | `Alert` variant `warning`: "You've reached the 10-block limit on the Free plan. Upgrade to Professional for unlimited blocks." + `Button` "Upgrade" |

---

### Interactions

| Action | System Response |
|---|---|
| User clicks "New Block" | Navigate to `/content-library/new`. Editor loads with empty state, title field auto-focused |
| User saves a block | POST to API. Success: `toast` "Block saved". Navigate to library. Block appears in list. Failure: `toast` destructive with error message, stay on editor |
| User toggles block to "Approved" | If user has permission: PATCH status. `toast` "Block approved — now available to all team members in the insert picker." If no permission: `toast` destructive "Only users with approval permission can approve blocks." Toggle reverts |
| User clicks "Duplicate" | POST clone endpoint. New block created with title "{original} (Copy)", status Draft. `toast` "Block duplicated". Navigate to edit view of new block |
| User clicks "Archive" | PATCH status to archived. `toast` "Block archived" with `Button` "Undo" (5 second window). Block removed from default view (visible with Archived filter) |
| User clicks "Delete" | `AlertDialog`: "Delete '{title}'? This will remove the block permanently. Proposals that reference this block will keep their current content but lose the link." Confirm: DELETE endpoint. `toast` "Block deleted". Cancel: close dialog |
| User searches | Debounced API call after 300ms. Results update in place with fade transition. Loading: skeleton cards/rows |
| User inserts block in editor | Block content injected at cursor. `content-block-ref` node wraps content. `toast` "Block inserted" |
| User clicks "Review Changes" on update notification | `Dialog` with side-by-side diff (old version left, new version right). Additions highlighted green, removals red. `Button` "Update to Latest" or "Keep Current" |
| User uploads DOCX | File uploaded to presigned URL. Server processes. Loading state with progress bar. On completion: preview list of parsed sections. User confirms: blocks created, `toast` "{N} blocks imported" |

---

### Responsive Behavior

| Breakpoint | Behavior |
|---|---|
| **Mobile (<640px)** | Single column list view forced (grid toggle hidden). Search + filters collapse into a `Sheet` triggered by filter icon button. Inspector panel opens as full-screen `Sheet` from bottom. Block editor is full-screen. Insert panel (from proposal editor) opens as full-screen overlay. Category badges show abbreviated text |
| **Tablet (640–1023px)** | 2-column grid or list view. Filters show inline but wrap. Inspector panel is 360px slide-over with backdrop. Block editor uses full width with toolbar wrapping |
| **Desktop (1024px+)** | 3–4 column grid or full table list. All filters inline. Inspector docks right (400px). Block editor centered with max-width 800px. Insert panel docks at 480px alongside editor |

---

### Data Sources

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/content-blocks` | GET | List blocks with pagination, search, filters. Query params: `q`, `category`, `status`, `author`, `sort`, `page`, `limit` |
| `/api/content-blocks` | POST | Create new block. Body: `{ title, category, content, tags, status }` |
| `/api/content-blocks/{id}` | GET | Fetch single block with version history |
| `/api/content-blocks/{id}` | PATCH | Update block (title, content, status, category, tags). Creates new version entry |
| `/api/content-blocks/{id}` | DELETE | Soft-delete block |
| `/api/content-blocks/{id}/duplicate` | POST | Clone block |
| `/api/content-blocks/{id}/versions` | GET | List version history |
| `/api/content-blocks/{id}/versions/{versionId}` | GET | Fetch specific version content |
| `/api/content-blocks/{id}/restore/{versionId}` | POST | Restore block to a previous version |
| `/api/content-blocks/import/docx` | POST | Upload and parse DOCX file. Returns candidate sections |
| `/api/content-blocks/import/proposal/{proposalId}` | GET | Parse proposal into candidate sections |
| `/api/proposals/{id}/block-refs` | GET | List content block references in a proposal (for update notifications) |

TanStack Query keys: `['content-blocks', filters]`, `['content-block', id]`, `['content-block-versions', id]`

---

### UX Requirements

| ID | Requirement |
|---|---|
| **UX-P14** | Content block cards must display a visible status indicator (color-coded badge) distinguishable without relying solely on color (include text label) |
| **UX-P15** | The search input must return results within 500ms for libraries of up to 500 blocks; display skeleton loading for longer queries |
| **UX-P16** | Auto-save in the block editor must occur every 30 seconds and on blur, with a visible "Saved" / "Saving…" / "Unsaved changes" indicator in the editor toolbar |
| **UX-P17** | When a user attempts to navigate away from the editor with unsaved changes, a confirmation dialog must appear ("You have unsaved changes. Save before leaving?") with Save / Discard / Cancel options |
| **UX-P18** | The Insert Block panel must load and be interactive within 300ms of invocation from the proposal editor, using cached data where available |
| **UX-P19** | Inserted content block references must be visually distinguishable from free-typed content in the proposal editor (left border + linked badge) without disrupting reading flow |
| **UX-P20** | When a source block is updated, all proposals containing a reference to that block must surface the update notification within the editor on next load — never silently auto-update content |
| **UX-P21** | The DOCX import flow must provide a preview of all parsed sections before creating any blocks, allowing the user to deselect, rename, or recategorize each section |
| **UX-P22** | Archived blocks must be excluded from the Insert Block picker and from default library views, but remain accessible via the "Archived" status filter for recovery |
| **UX-P23** | Version history must show a maximum of 50 versions per block with pagination, displaying author, timestamp, and a one-line change summary for each entry |

---

## SCREEN 7: ESPD Builder

**Route:** `/opportunities/{opportunityId}/espd` (new ESPD), `/opportunities/{opportunityId}/espd/{espdId}` (resume draft), `/espd/templates` (saved templates)

**Access:** Professional and Enterprise tiers. All roles that have access to the parent opportunity. ESPD generation requires `espd:create` permission. Final sign-off requires `espd:sign` permission (typically Admin or Legal role).

**Purpose:** Generate a pre-populated European Single Procurement Document by mapping stored company data to the standardized ESPD structure, allowing review, completion of gaps, validation, and export in XML or PDF format.

**Layout:** Full-width page within app shell. Wizard-style step navigation on the left (vertical stepper on desktop, horizontal compact stepper on mobile). Main content area shows the active section's form. Right panel (280px, desktop only) shows a persistent validation summary and field mapping info.

---

### Components

#### 7.1 Entry Point (on Opportunity Detail Page)

| Element | Details |
|---|---|
| Location | Opportunity Detail page, actions bar or a dedicated "Documents" tab |
| Button | `Button` "Generate ESPD" with `FileText` icon. Visible only when `opportunity.requiresEspd === true` |
| Existing ESPD | If a draft ESPD exists for this opportunity: `Button` "Resume ESPD Draft" (primary) + `Badge` "Draft — 68% complete". If a completed ESPD exists: `Button` variant `outline` "View ESPD" + `Badge` "Completed" (green) |
| Template option | `DropdownMenu` on the button: "Generate from scratch" / "Generate from template" (opens template picker) |

#### 7.2 Wizard Navigation (Vertical Stepper)

Left sidebar within the ESPD page (not the app sidebar). Width: 260px on desktop.

| Step | Label | Icon |
|---|---|---|
| Part I | Procurement Procedure | `FileSearch` |
| Part II | Economic Operator | `Building2` |
| Part III | Exclusion Grounds | `ShieldAlert` |
| Part IV | Selection Criteria | `CheckSquare` |
| Part V | Candidate Reduction | `Filter` |
| Part VI | Final Statements | `PenTool` |
| Summary | Review & Export | `Download` |

Each step shows:
- Step number in a circle (completed: green check, current: primary fill, upcoming: gray outline, has errors: red outline)
- Label text
- Completion percentage: `text-xs text-muted-foreground` e.g. "12/15 fields"
- Warning indicator if the section has validation issues

Clicking any completed or current step navigates to it. Future steps are clickable but show a note if previous required steps are incomplete.

#### 7.3 Page Header

| Element | Details |
|---|---|
| Breadcrumb | Opportunities > {Opportunity Title} > ESPD |
| Title | "ESPD — {Opportunity Title}" — `text-xl font-semibold`, truncated with tooltip for long titles |
| Status | `Badge`: "Draft" (gray) or "Complete" (green) or "Exported" (blue) |
| Last saved | `text-xs text-muted-foreground` "Last saved 2 minutes ago" with auto-save indicator |
| Actions | `Button` "Save Draft" (outline), `Button` "Export" (primary, dropdown: XML / PDF) — Export only enabled when all required fields pass validation |

#### 7.4 Part I — Procurement Procedure Information

All fields auto-filled from opportunity data. Section is read-only with an "Edit" toggle for corrections.

| Field | Source | Auto-fill | Editable |
|---|---|---|---|
| Procedure title | `opportunity.title` | Yes | Yes (override) |
| Procedure reference number | `opportunity.referenceNumber` | Yes | Yes |
| Contracting authority name | `opportunity.contractingAuthority.name` | Yes | Yes |
| Contracting authority country | `opportunity.contractingAuthority.country` | Yes | Yes |
| Procedure type | `opportunity.procedureType` | Yes | Select dropdown |
| CPV codes | `opportunity.cpvCodes` | Yes | Tag input |
| NUTS codes | `opportunity.nutsCode` | Yes | Tag input |
| TED notice number | `opportunity.tedNoticeId` | Yes | Yes |
| Lot information | `opportunity.lots[]` | Yes | Table (lot number, title, CPV) |

Visual treatment: All auto-filled fields have a `bg-green-50 dark:bg-green-950/20` background with a small `Sparkles` icon and tooltip "Auto-filled from opportunity data."

#### 7.5 Part II — Economic Operator Information

Auto-filled from the organization's company profile.

| Field Group | Fields | Source |
|---|---|---|
| Identification | Name, VAT number, registration number, address, city, postal code, country, website, phone, email, NUTS code | `companyProfile.legal.*` |
| Size classification | Micro / Small / Medium / Large enterprise | `companyProfile.sizeClassification` |
| Representatives | Name, title, date of birth, address — for each authorized representative | `companyProfile.representatives[]` |
| Consortium info | Is the EO participating as part of a group? If yes: consortium members table (name, VAT, role) | Manual or from consortium workspace |
| Subcontracting | Does the EO intend to subcontract? If yes: percentage, description of subcontracted parts | Manual input |
| ESPD reuse | Reference to previously submitted ESPD (ID, issuing body, date) | Dropdown from saved ESPDs |

Field mapping indicator: Next to each field, a small `Link2` icon with tooltip "Mapped from: Company Profile > Legal > {field}" — clicking opens the company profile field in a popover for quick verification.

#### 7.6 Part III — Exclusion Grounds

Organized into sub-sections per the ESPD standard. Each exclusion ground is a yes/no declaration with an optional explanation field.

| Sub-section | Grounds | Auto-fill Source |
|---|---|---|
| A. Criminal convictions | Participation in criminal org, corruption, fraud, terrorist offences, money laundering, child labour | `companyProfile.declarations.criminal` |
| B. Payment of taxes | Tax obligations fulfilled? | `companyProfile.declarations.tax` |
| C. Social security | Social security contributions fulfilled? | `companyProfile.declarations.socialSecurity` |
| D. Environmental, social, labour law | Breaches of obligations? | `companyProfile.declarations.environmental` |
| E. Bankruptcy/insolvency | Bankruptcy, winding up, arrangement with creditors? | `companyProfile.declarations.insolvency` |
| F. Professional misconduct | Grave professional misconduct? | `companyProfile.declarations.misconduct` |
| G. Conflict of interest | Any conflict of interest? | Manual — amber highlight |
| H. Distortion of competition | Prior involvement in preparation of procurement? | Manual — amber highlight |
| I. Early termination | Past early termination, damages, or other sanctions? | `companyProfile.declarations.earlyTermination` |

Each ground renders as:

| Element | Details |
|---|---|
| Question text | Full text of the exclusion ground question, `text-sm` |
| Answer | `RadioGroup`: "No" (not subject to this ground) / "Yes" (subject, provide details) |
| Auto-fill indicator | If auto-filled: green background. If requires manual input: amber background |
| Details field | `Textarea` — visible only when "Yes" is selected. Fields for: description, dates, remedial measures taken, whether self-cleaning applies |
| Evidence | `Button` "Attach Evidence" — file upload for supporting documents |

#### 7.7 Part IV — Selection Criteria

Dynamic section — criteria are determined by the specific procurement opportunity. The system maps known criteria to stored company data.

| Criteria Group | Examples | Auto-fill |
|---|---|---|
| A. Suitability | Professional register enrollment, authorization/membership | From `companyProfile.registrations` |
| B. Economic standing | Minimum turnover, financial ratios, professional indemnity insurance | From `companyProfile.financials` |
| C. Technical ability | Past project references, team qualifications, equipment, supply chain | From content library (Past Projects blocks), `companyProfile.team` |
| D. Quality assurance | ISO certifications, environmental management standards | From `companyProfile.certifications` |

Each criterion renders as:

| Element | Details |
|---|---|
| Criterion label | `text-sm font-medium` |
| Requirement | Contracting authority's minimum requirement, `text-xs text-muted-foreground` |
| Response field | Type varies: text input, number input, file upload, or multi-line text. Pre-filled where data exists |
| Status | `Badge` inline: "Auto-filled" (green), "Needs Input" (amber), "Missing" (red) |
| Content block link | For text responses: `Button` variant `ghost` size `sm` "Insert from Library" — opens content block picker scoped to relevant categories |

#### 7.8 Part V — Candidate Reduction

Only visible when the procurement procedure involves candidate reduction (restricted procedure, competitive dialogue, etc.).

| Element | Details |
|---|---|
| Visibility logic | Rendered only when `opportunity.procedureType` is in `['restricted', 'competitive_dialogue', 'innovation_partnership', 'competitive_negotiated']` |
| Content | Objective, non-discriminatory criteria for limiting candidates. Free-text response `Textarea` with guidance text |
| Auto-fill | Typically manual. Amber highlight |

#### 7.9 Part VI — Final Statements

| Element | Details |
|---|---|
| Declarations | Series of read-only declaration texts with `Checkbox` acknowledgment: "The information provided is accurate and complete", "Able to provide supporting evidence upon request", "Consent to access evidence from national databases (if applicable)" |
| Signatory | Name, position, date — `Input` fields. Name auto-filled from user profile |
| Electronic signature | `Button` "Sign ESPD" — if e-signature integration is configured (future feature), triggers signing flow. Otherwise, declaration checkbox serves as acknowledgment |
| Legal notice | `Alert` variant `info`: full legal notice text per ESPD regulation |

#### 7.10 Summary / Review & Export (Final Step)

| Element | Details |
|---|---|
| Section-by-section summary | `Accordion` — each part as a collapsible section showing: completion percentage, count of auto-filled vs manual fields, list of any validation errors or warnings |
| Validation summary | `Card` at top: total fields, completed fields, auto-filled count, manual count. Errors listed with links to the relevant section/field |
| Color-coded overview | Miniature section map: each section is a horizontal bar — green (complete), amber (has warnings), red (has errors) |
| Field mapping table | `Table` in a collapsible `Accordion` section: columns — ESPD Field, Company Profile Field, Value, Status. Shows every mapped field. Filterable by status |
| Export buttons | `Button` "Export XML" (primary) — generates ESPD XML per the EU schema. `Button` "Export PDF" (outline) — generates formatted PDF. Both disabled if validation errors exist. Enabled with warnings (user must acknowledge) |
| Save as template | `Button` variant `outline` "Save as Template" — saves the current ESPD with procurement-specific fields cleared. Opens `Dialog` for template name and description |
| Print | `Button` variant `ghost` "Print" — browser print with print-optimized CSS |

#### 7.11 Validation Summary Panel (Right Sidebar)

Persistent on desktop, accessible via `Sheet` on mobile.

| Element | Details |
|---|---|
| Overall progress | Circular progress indicator (radial chart) showing completion percentage |
| Section breakdown | List of Parts I–VI, each with mini progress bar and field count |
| Errors | Red items: list of missing required fields with click-to-navigate |
| Warnings | Amber items: fields that could benefit from review |
| Auto-fill stats | "34 of 52 fields auto-filled (65%)" with `Progress` bar |

#### 7.12 Template Picker Dialog

| Element | Details |
|---|---|
| Trigger | "Generate from template" option on the entry point button |
| Dialog | `Dialog` — list of saved ESPD templates. Each: name, source opportunity title, date created, completion percentage |
| Action | Select template → "Use Template" button → creates new ESPD pre-filled from template + opportunity data overlay (opportunity-specific fields overwritten from current opportunity) |

---

### Interactions

| Action | System Response |
|---|---|
| User clicks "Generate ESPD" | API call to create ESPD draft. Server runs auto-fill mapping (company profile + opportunity data). Redirects to `/opportunities/{id}/espd/{espdId}` at Part I. `toast` "ESPD generated — 65% auto-filled. Review highlighted fields." |
| User navigates between steps | Current section form state saved via auto-save (debounced 2s). Step indicator updates. Validation runs on section leave. If errors: step shows red indicator but navigation is not blocked |
| User edits an auto-filled field | Field background changes from green to white. A small "Modified" indicator appears. The original auto-filled value is accessible via tooltip ("Original: {value}") for reference |
| User clicks field mapping icon | Popover shows source field path, current value in company profile, and `Button` "Go to Profile" (opens company profile in new tab at the relevant section) |
| Auto-save triggers | PATCH endpoint. "Saving…" text appears next to last-saved timestamp. On completion: "Saved just now". On failure: `toast` destructive "Failed to save — retrying…" with automatic retry (3 attempts, exponential backoff) |
| User clicks "Export XML" | Validation runs. If all required fields pass: generate XML, download file (`espd_{opportunity_ref}_{date}.xml`). If errors: `AlertDialog` listing errors with "Go to first error" button. If warnings only: `AlertDialog` "Export has {N} warnings. Proceed anyway?" |
| User clicks "Export PDF" | Same validation flow. PDF generated server-side with proper formatting, headers, EU ESPD branding. Downloaded as `espd_{opportunity_ref}_{date}.pdf` |
| User clicks "Save as Template" | `Dialog` with name input (pre-filled: "Template — {opportunity title}") and description textarea. On save: POST to templates endpoint. `toast` "ESPD template saved" |
| User restores from template | New ESPD created. Opportunity-specific fields (Part I) filled from current opportunity (overriding template). Company-specific fields kept from template but re-validated against current company profile. Diff highlighted if company data changed since template creation |

---

### Responsive Behavior

| Breakpoint | Behavior |
|---|---|
| **Mobile (<640px)** | Stepper collapses to a horizontal compact bar at the top showing current step number/label with left/right chevrons for navigation. Validation panel hidden — accessible via floating `Button` with error count badge opening a `Sheet`. Form fields stack single-column. Field mapping icons hidden (accessible via field long-press). Export actions in a fixed bottom action bar |
| **Tablet (640–1023px)** | Stepper renders as a horizontal bar across top with abbreviated labels (Part I, Part II, etc.). Validation panel as a collapsible right panel (240px). Form fields in two-column grid where appropriate |
| **Desktop (1024px+)** | Full vertical stepper on left (260px). Validation panel docked right (280px). Form content centered (max-width 720px). All field mapping indicators visible inline |

---

### Data Sources

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/opportunities/{id}/espd` | POST | Create new ESPD draft with auto-fill. Body: `{ templateId? }` |
| `/api/opportunities/{id}/espd` | GET | List ESPDs for this opportunity |
| `/api/espd/{id}` | GET | Fetch full ESPD data (all parts) |
| `/api/espd/{id}` | PATCH | Auto-save updates. Body: partial ESPD data for changed sections |
| `/api/espd/{id}/validate` | POST | Run full validation. Returns errors and warnings per field |
| `/api/espd/{id}/export/xml` | POST | Generate and return ESPD XML file |
| `/api/espd/{id}/export/pdf` | POST | Generate and return ESPD PDF file |
| `/api/espd/templates` | GET | List saved ESPD templates |
| `/api/espd/templates` | POST | Save current ESPD as template |
| `/api/company-profile/espd-mapping` | GET | Returns field mapping table (company profile field → ESPD field path → current value) |

TanStack Query keys: `['espd', espdId]`, `['espd-list', opportunityId]`, `['espd-templates']`, `['espd-mapping']`

---

### UX Requirements

| ID | Requirement |
|---|---|
| **UX-P24** | Auto-filled fields must be visually distinct from manual fields using background color AND an icon indicator, ensuring accessibility for color-blind users |
| **UX-P25** | The wizard must allow non-linear navigation — users can jump to any section at any time without losing data; validation warnings appear but do not block navigation |
| **UX-P26** | Auto-save must trigger within 2 seconds of field change and must not interrupt the user's workflow (no modal dialogs, no focus steal); save status must be visible at all times |
| **UX-P27** | The validation summary must update in real-time as fields are completed, showing total progress, per-section progress, and an accurate count of remaining required fields |
| **UX-P28** | Export buttons must be disabled with a clear tooltip explanation when validation errors exist; warnings must allow export with explicit user acknowledgment |
| **UX-P29** | Field mapping indicators must show the exact source path from the company profile (e.g., "Company Profile > Legal > VAT Number") so users can verify and correct source data |
| **UX-P30** | When creating an ESPD from a template, fields where company profile data has changed since the template was created must be highlighted with a "Data Updated" indicator showing old vs. new values |
| **UX-P31** | The ESPD builder must function fully offline-capable for draft editing, syncing changes when connectivity is restored (using service worker + IndexedDB) |
| **UX-P32** | Part V (Candidate Reduction) must be automatically hidden for procedure types that do not require it, to avoid confusing users with irrelevant sections |

---

## SCREEN 8: Consortium Finder

**Route:** `/grants/{grantId}/consortium` (also accessible as a tab within the Grant Opportunity detail page)

**Access:** Professional and Enterprise tiers. All users with access to the parent grant opportunity. "Add to consortium" action requires `grant:edit` permission.

**Purpose:** Search a database of organizations that have participated in EU-funded grants to identify and assemble optimal consortium partners for a specific grant application, with AI-powered complementarity scoring.

**Layout:** Standard app shell. Main content area splits into two zones: left (search/results, flex-1) and right (Consortium Workspace panel, 380px, collapsible). On initial load, the workspace panel is collapsed if no partners have been added yet.

---

### Components

#### 8.1 Page Header

| Element | Details |
|---|---|
| Breadcrumb | Grants > {Grant Programme} > {Grant Title} > Consortium |
| Title | "Consortium Finder" — `text-xl font-semibold` |
| Grant context | `Card` compact: grant title, programme, deadline, budget range, topic. Collapsible on scroll |
| Primary action | `Button` "Suggest Partners" with `Sparkles` icon — triggers AI consortium suggestion agent |
| Secondary action | `Button` variant `outline` "Export Consortium" — exports assembled consortium as PDF summary |

#### 8.2 Search Interface

| Element | Details |
|---|---|
| Search input | `Input` full-width, `Search` icon prefix. Placeholder: "Search organizations by name, expertise, or keyword…". Debounced 300ms |
| Quick filters row | Horizontal scrollable row of `Toggle` buttons for common filters: "My Country", "EU-13 (Widening)", "SME", "Academic", "Research Institute", "Industry" |

#### 8.3 Advanced Filters Panel

Collapsible panel below search, opened via `Button` variant `ghost` "Advanced Filters" with `ChevronDown`.

| Filter | Type | Details |
|---|---|---|
| Country | Multi-select `Combobox` with search | All EU/EEA countries + associated countries. Country flags. Grouped by region |
| Sector / CPV | Multi-select `Combobox` with hierarchical browse | CPV code tree or sector tags |
| Organization type | Multi-select `Checkbox` group | University, Research Institute, Large Enterprise, SME, NGO, Public Body, Other |
| Past grant programmes | Multi-select `Combobox` | Horizon Europe, Horizon 2020, Structural Funds, Erasmus+, LIFE, Digital Europe, etc. |
| Role in consortium | Multi-select `Checkbox` group | Coordinator / Lead, Work Package Leader, Participant |
| Grant success rate | `Slider` range (0–100%) | Minimum success rate filter |
| Expertise area | Tag input with autocomplete | Free-text expertise tags matched against organization profiles |

Active filters displayed as removable `Badge` chips below the filter panel.

#### 8.4 Results — Card Grid (Desktop) / List (Mobile)

| Element | Details |
|---|---|
| Results count | `text-sm text-muted-foreground` "Found 234 organizations" |
| Sort | `Select`: Relevance (default), Complementarity Score, Success Rate, Grant Count, Name A–Z |
| Pagination | `Button` "Load more" at bottom (infinite scroll) or numbered pagination — configurable. 20 results per page |

Each result card:

| Element | Details |
|---|---|
| Layout | `Card` — desktop: grid `grid-cols-1 md:grid-cols-2 xl:grid-cols-3 gap-4`. Mobile: single-column list |
| Organization name | `text-sm font-semibold`. Clickable (opens detail view) |
| Country | Flag emoji + country name. `text-sm text-muted-foreground` |
| Organization type | `Badge` variant `outline`: University, SME, Research Institute, etc. |
| Expertise tags | Row of `Badge` variant `secondary`, max 4 shown + "+N more" overflow |
| Grant participation | `text-xs text-muted-foreground` with `Award` icon: "{N} EU grants" |
| Success rate | Mini radial chart (24px) or `text-xs`: "{X}% success rate" — color-coded: >60% green, 30-60% amber, <30% red |
| Complementarity score | `Badge` with score 0–100. Color: 80+ green "Excellent match", 60–79 blue "Good match", 40–59 amber "Moderate match", <40 gray "Low match". Tooltip explains scoring factors |
| Actions | `Button` size `sm` "Add to Consortium" (`Plus` icon). `Button` variant `ghost` size `sm` `Bookmark` icon (save for later) |
| Hover | `shadow-md` elevation, `ring-1 ring-ring` |

#### 8.5 Organization Detail View

Opens as a `Sheet` from the right (560px) or as a full-page modal on mobile.

| Section | Details |
|---|---|
| Header | Organization name, country flag, type badge, website link (external). `Button` "Add to Consortium" (primary) |
| Overview | Short description (if available), founding year, size (employees), annual revenue range |
| Expertise | Full list of expertise tags, organized by domain |
| Past EU Grants | `Table`: Grant title, Programme, Year, Role (Coordinator/Partner), Budget allocated, Status (completed/ongoing). Sortable. Paginated (10 per page) |
| Success metrics | Total grants applied (if known), total grants awarded, success rate, total funding received |
| Geographic coverage | Countries where the organization has operated projects — shown as a mini EU map with highlighted countries or as a tag list |
| Contact info | Public contact information (if available from EU databases): contact person, email, phone. If not available: `Alert` "Contact information is not publicly available. You may find it on their website." + website link |
| Collaboration history | If the user's organization has previously been in a consortium with this organization: `Alert` variant `info` "You have collaborated with {org} on {N} previous grants" with list |
| Actions | `Button` "Add to Consortium" (primary), `Button` "Save Partner" (outline, bookmark), `Button` "Request Intro" (outline, visible only if contact email is available — opens pre-filled email template) |

#### 8.6 Consortium Workspace Panel

Docked right panel, 380px. Collapsible via toggle button. Expands to full width on mobile as a `Sheet`.

| Element | Details |
|---|---|
| Header | "Consortium for {Grant Title}" — `text-base font-semibold`. Partner count: "{N} partners" |
| User's organization | Always shown first, pinned. `Card` compact with org name, country, role `Select` (default: Coordinator) |
| Partner list | Sortable list (`dnd-kit` drag handles) of added partners. Each entry is a compact `Card` |

Each partner entry in workspace:

| Element | Details |
|---|---|
| Organization name | `text-sm font-medium` |
| Country flag | Inline next to name |
| Role | `Select` inline: Coordinator, Work Package Leader, Participant |
| Budget allocation | `Input` type number, with currency symbol. `text-sm` |
| WP assignment | `Select` multi-select: which work packages this partner leads/participates in (WPs can be defined inline or pulled from proposal structure) |
| Remove | `Button` variant `ghost` `X` icon with confirmation tooltip "Remove from consortium?" |

#### 8.7 Consortium Summary Section (within Workspace)

Below the partner list:

| Element | Details |
|---|---|
| Total budget | Sum of all partner allocations. `text-sm font-semibold`. Warning if exceeds grant budget range |
| Country coverage | List of unique countries. `Badge` chips with flags. Visual indicator if geographic diversity is strong/weak |
| Complementarity analysis | AI-generated `Card`: overall consortium strength score (0–100), strengths list (what's well-covered), gaps list (missing expertise/capacity). Refreshes when consortium composition changes |
| Organization type balance | Mini bar chart (Recharts) showing distribution: academic, industry, SME, public body |
| Actions | `Button` "Export Consortium Summary" (PDF with partner table, budget breakdown, complementarity analysis). `Button` "Generate Partner Table" (creates a formatted table suitable for grant application forms) |

#### 8.8 AI Suggestion Flow

| Element | Details |
|---|---|
| Trigger | "Suggest Partners" button in page header |
| Loading state | `Dialog` with animated progress: "Analyzing grant requirements…", "Scanning organization database…", "Calculating complementarity scores…". Cancel button available |
| Results | `Dialog` or dedicated view: ranked list of 5–10 suggested organizations. Each with: organization details (same as result card), complementarity score, AI explanation (2–3 sentences explaining why this partner is recommended for this specific grant). Example: "Strong track record in marine biology research (WP2 requirement). Has coordinated 3 Horizon Europe projects in the Blue Economy cluster. Based in Portugal, adding geographic diversity." |
| Actions per suggestion | `Button` "Add to Consortium", `Button` "View Profile", `Button` "Dismiss" |
| Refinement | After reviewing suggestions: `Button` "Suggest More" or `Input` "Refine: e.g., 'Need an SME partner in Germany with AI expertise'" |

#### 8.9 Empty States

| State | Display |
|---|---|
| No search results | "No organizations found matching your criteria" + "Try broadening your search or removing some filters" + `Button` "Clear Filters" |
| Empty consortium workspace | Illustration (people network) + "No partners added yet" + "Search for organizations or click 'Suggest Partners' to get AI recommendations" |
| Feature unavailable (Free tier) | `Card` with lock icon: "Consortium Finder is available on Professional and Enterprise plans" + feature highlights + `Button` "Upgrade" |

---

### Interactions

| Action | System Response |
|---|---|
| User searches | Debounced API call. Results update with fade transition. Search highlights matching terms in results. Skeleton cards during load |
| User applies filters | API call with filter params. URL query params updated (shareable/bookmarkable). Results count updates. Active filters shown as chips |
| User clicks organization card | Detail `Sheet` slides in from right. Organization data loaded (may require additional API call for full profile) |
| User clicks "Add to Consortium" | Organization added to workspace panel. Panel expands if collapsed. `toast` "{Organization} added to consortium". Card in search results shows `Badge` "Added" and the add button changes to `Button` variant `outline` "Added" (disabled) with `Check` icon |
| User removes partner from workspace | Confirmation tooltip. On confirm: partner removed, `toast` "{Organization} removed". Search result card reverts to "Add" button. Complementarity analysis refreshes |
| User changes partner role or budget | Inline edit. Auto-saves on blur. Consortium summary recalculates |
| User clicks "Suggest Partners" | AI agent triggered. Loading dialog appears. On completion: suggestion dialog with ranked results. Typical wait: 10–30 seconds |
| User exports consortium | PDF generated server-side. Contains: grant details, partner table (name, country, type, role, budget), complementarity analysis, expertise coverage matrix. Downloaded as `consortium_{grant_title}_{date}.pdf` |
| User clicks "Request Intro" | Opens default email client with pre-filled template: Subject "Potential Consortium Partnership — {Grant Title}", Body template with grant description, reason for contact, user's organization intro. Uses `mailto:` link |
| User bookmarks organization | POST to bookmarks endpoint. `Bookmark` icon fills. Bookmarked orgs accessible from a "Saved Partners" section (accessible via filter or dedicated tab) |

---

### Responsive Behavior

| Breakpoint | Behavior |
|---|---|
| **Mobile (<640px)** | Single column list view for results (cards simplified: name, country, score, add button). Filters collapse into a `Sheet` triggered by filter icon. Consortium workspace as a full-screen `Sheet` accessible via floating action button with partner count badge. Organization detail as full-screen `Sheet`. AI suggestions as full-screen `Sheet` |
| **Tablet (640–1023px)** | Two-column card grid for results. Consortium workspace as a collapsible right panel (320px) or bottom sheet. Filters in a collapsible section |
| **Desktop (1024px+)** | Three-column card grid. Consortium workspace docked right (380px). All filters visible. Organization detail as 560px side sheet |

---

### Data Sources

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/organizations/search` | GET | Search organizations. Query: `q`, `countries[]`, `sectors[]`, `types[]`, `programmes[]`, `roles[]`, `successRateMin`, `expertise[]`, `sort`, `page`, `limit` |
| `/api/organizations/{id}` | GET | Full organization profile |
| `/api/organizations/{id}/grants` | GET | Past grant participation for an organization |
| `/api/grants/{grantId}/consortium` | GET | Current consortium for this grant |
| `/api/grants/{grantId}/consortium/partners` | POST | Add partner to consortium. Body: `{ organizationId, role, budget }` |
| `/api/grants/{grantId}/consortium/partners/{partnerId}` | PATCH | Update partner role/budget |
| `/api/grants/{grantId}/consortium/partners/{partnerId}` | DELETE | Remove partner from consortium |
| `/api/grants/{grantId}/consortium/complementarity` | GET | AI-generated complementarity analysis for current consortium |
| `/api/grants/{grantId}/consortium/suggest` | POST | AI partner suggestion. Body: `{ refinement? }` |
| `/api/grants/{grantId}/consortium/export` | POST | Generate consortium PDF export |
| `/api/organizations/bookmarks` | GET/POST/DELETE | Manage saved partner bookmarks |

TanStack Query keys: `['org-search', filters]`, `['organization', id]`, `['org-grants', id]`, `['consortium', grantId]`, `['consortium-complementarity', grantId]`, `['org-bookmarks']`

---

### UX Requirements

| ID | Requirement |
|---|---|
| **UX-P33** | Complementarity scores must include a visible explanation (tooltip or expandable text) so users understand the scoring rationale, not just the number |
| **UX-P34** | Organizations already added to the consortium must be clearly marked in search results (visual badge + disabled add button) to prevent duplicate additions |
| **UX-P35** | The Consortium Workspace must persist across page navigations within the grant context — navigating to org details or changing filters must not lose the assembled consortium state |
| **UX-P36** | AI partner suggestions must complete within 30 seconds; if exceeding this, show a progress update every 5 seconds and allow the user to cancel without losing any previously loaded results |
| **UX-P37** | Search results must load within 500ms for the first page, with skeleton loading for subsequent pages. Infinite scroll or pagination must pre-fetch the next page for seamless experience |
| **UX-P38** | The "Request Intro" email template must be pre-filled but fully editable before sending, and must never send automatically without explicit user action |
| **UX-P39** | Budget allocations in the Consortium Workspace must validate against the grant's total budget range and display a warning (not a block) when the sum exceeds the maximum grant budget |
| **UX-P40** | Country coverage and organization type balance visualizations must update in real-time as partners are added or removed, providing immediate feedback on consortium composition |

---

## SCREEN 9: Reporting Template Generator (Post-Award)

**Route:** `/active-grants` (list), `/active-grants/{grantId}` (detail), `/active-grants/{grantId}/reports` (reports tab), `/active-grants/{grantId}/reports/{reportId}` (individual report editor)

**Access:** Professional+ tier only. Roles: Grant Manager (full CRUD), Team Member (view + edit assigned sections), Finance (budget tracking + financial statements), Admin (all access).

**Purpose:** Manage post-award grant lifecycle including generating pre-filled periodic reports, tracking budget expenditure, and monitoring deliverables against deadlines.

**Layout:** Standard app shell with "Active Grants" as a new sidebar nav group (icon: `Trophy`), positioned below "Grants" in the navigation hierarchy. Detail page uses a tabbed layout within the main content area.

---

### Components

#### 9.1 Active Grants List Page (`/active-grants`)

| Element | Details |
|---|---|
| Page title | "Active Grants" — `text-2xl font-semibold` |
| Subtitle | Count: "{N} active grants" — `text-sm text-muted-foreground` |
| Filters | `Select` Programme filter, `Select` Status filter (Active, Completed, Suspended), search `Input` |
| View toggle | `ToggleGroup`: Card view / Table view |

Each grant card:

| Element | Details |
|---|---|
| Grant title | `text-sm font-semibold`, clickable |
| Programme badge | `Badge` variant `outline` e.g. "Horizon Europe" |
| Project acronym | `text-xs font-medium text-primary` |
| Duration | `text-xs text-muted-foreground` "Jan 2025 — Dec 2027 (36 months)" |
| Status | `Badge`: "Active" (green), "Reporting Period" (amber), "Final Reporting" (red), "Completed" (blue) |
| Budget | `text-sm` Total grant amount, with mini `Progress` bar showing budget burn |
| Next deadline | `text-xs` Nearest reporting deadline with relative time. Red text if overdue. Amber if <30 days |
| Progress indicators | Three micro-metrics: Reports (2/6), Deliverables (8/15), Budget (45% spent) |
| Quick actions | `DropdownMenu`: "Open Dashboard", "Create Report", "View Budget" |

Table view columns: Grant Title, Programme, Acronym, Status, Budget (spent/total), Next Deadline, Reports Due, Actions.

#### 9.2 Active Grant Detail Page — Overview Tab (`/active-grants/{grantId}`)

| Element | Details |
|---|---|
| Header | Grant title, programme badge, project acronym, GA number (Grant Agreement). `Button` "Settings" (gear icon), `Button` "Export All" (dropdown) |
| Tab bar | `Tabs`: Overview | Reports | Budget Tracking | Deliverables |
| Status banner | Contextual `Alert` at top: e.g., "Periodic Report #2 is due in 14 days" (amber) or "Mid-Term Review submitted successfully" (green) or "Financial Statement overdue by 3 days" (red) |

Overview tab content:

| Section | Details |
|---|---|
| Project summary card | `Card`: title, abstract, start/end dates, total budget, funding programme, Grant Agreement number, project officer contact |
| Consortium summary | Compact partner list: organization name, country, role, budget allocated, `Progress` bar of budget used. Clickable to expand |
| Timeline | Horizontal timeline (custom component or Recharts) showing project duration divided into reporting periods. Current position marked. Key milestones plotted. Deliverable due dates as dots |
| Key metrics | Four `Card` stats: Total Budget / Spent, Reports Submitted / Total, Deliverables Completed / Total, Days Remaining |
| Upcoming deadlines | `Table` of next 5 deadlines (report submissions, deliverable due dates, reviews). Each with type, due date, relative time, status, `Button` "Go to" |
| Recent activity | Feed of recent actions: report created, deliverable marked complete, budget entry added. Compact list, `text-xs`, with relative timestamps |

#### 9.3 Reports Tab (`/active-grants/{grantId}/reports`)

##### 9.3.1 Report Calendar

| Element | Details |
|---|---|
| Layout | Horizontal timeline or Gantt-style view. Each reporting period as a block. Within each period: report types due |
| Report items | Each report shows: type icon, name (e.g., "Periodic Report #2"), due date, status badge |
| Status badges | `Badge` variants: "Upcoming" (gray), "In Progress" (blue), "In Review" (amber), "Overdue" (red, pulsing dot), "Submitted" (green), "Accepted" (green with check) |
| Current period highlight | Current reporting period background highlighted with `bg-primary/5` border |

##### 9.3.2 Report Types Grid

Below the calendar. One section per report type.

| Report Type | Template Structure | AI Pre-fill Capability |
|---|---|---|
| **Periodic Technical Report** | Publishable summary, WP progress (per WP: objectives, tasks, progress, deviations), deliverable status table, milestone status table, risk register, next period plans, dissemination & exploitation | WP progress from deliverable data + previous reports. Deliverable/milestone status from tracking data. Risks from risk register |
| **Financial Statement** | Cost categories (personnel, travel, equipment, subcontracting, other direct, indirect), per partner breakdown, receipts summary, co-financing declaration | Personnel costs from budget entries. Categorized actual spend from budget tracking. Indirect cost calculation (25% flat rate for HE) |
| **Mid-Term Review** | Project summary, progress against objectives, critical risks, consortium performance, financial summary, amendment requests | Aggregated from periodic reports + budget tracking |
| **Final Report** | Comprehensive: all WP summaries, complete deliverable list, full financial summary, impact assessment, sustainability plan, lessons learned | Heavy AI pre-fill from all previous reports, deliverables, and budget data |

Each report type card:

| Element | Details |
|---|---|
| Card | `Card` with report type name, icon, and description |
| Due dates | List of due dates for this type (e.g., every 18 months for periodic reports) |
| Existing reports | List of created reports with status |
| Create button | `Button` "Create Report" — only active when a report is due or upcoming. Opens report creation flow |

##### 9.3.3 Report Creation / Editor (`/active-grants/{grantId}/reports/{reportId}`)

| Element | Details |
|---|---|
| Header | Report type + period: "Periodic Technical Report — Period 1 (M1–M18)". Status badge. Auto-save indicator |
| AI pre-fill button | `Button` "Generate Draft" with `Sparkles` icon — triggers the reporting template agent. Shows progress dialog |
| Section navigation | Left sidebar (240px): structured list of report sections (same as ESPD wizard but not linear — tree navigation). Each section shows completion indicator |
| Editor area | Tiptap rich-text editor, full-width. Section title as `h2` above editor. Guidance text in `Alert` variant `info` above each section explaining what's expected |
| Pre-fill indicators | AI-generated content highlighted with `bg-blue-50 dark:bg-blue-950/20` left border and "AI Generated — Review Required" label. User can accept (removes highlight), edit, or regenerate |
| Data insertion | `Button` "Insert Data" within sections — opens context-aware picker: for WP progress sections, shows deliverable data; for financial sections, shows budget data; for milestone sections, shows milestone tracking data |
| Collaboration | Section assignment: each section can be assigned to a team member via `Select`. Assigned sections show assignee avatar. Assignees see only their sections in their dashboard |
| Comments | Inline comments on text selections (same as proposal editor if available). Section-level comments via `Button` "Comment" |
| Validation | Per-section: word count guidance (e.g., "Recommended: 500–1000 words"), required field indicators, completeness check |
| Actions | `Button` "Save Draft" (always available), `Button` "Submit for Review" (when all required sections complete), `Button` "Export" (dropdown: PDF / DOCX, formatted per programme requirements) |

##### 9.3.4 Report Template Selection

| Element | Details |
|---|---|
| Trigger | On "Create Report" click |
| Dialog | `Dialog` showing available templates for the selected report type and programme. Templates: "Horizon Europe Periodic Technical Report", "Horizon Europe Financial Statement (Form C)", custom templates. Each with: template name, programme, section count, description |
| Action | Select template → creates report with section structure. AI pre-fill runs automatically after creation |

#### 9.4 Budget Tracking Tab (`/active-grants/{grantId}/budget`)

##### 9.4.1 Budget Overview

| Element | Details |
|---|---|
| Summary cards | Row of `Card` components: Total Budget / Spent / Remaining, Burn Rate (monthly average), Projected End Date (at current burn rate), Co-financing Status |
| Budget health indicator | `Badge` large: "On Track" (green, within 10% of planned), "Underspent" (amber, >20% below planned), "Overspent" (red, >10% above planned) |

##### 9.4.2 Actual vs Planned Chart

| Element | Details |
|---|---|
| Chart type | Recharts `AreaChart` — two series: Planned Cumulative (dashed line), Actual Cumulative (solid fill). X-axis: project months. Y-axis: EUR |
| Interaction | Hover shows tooltip with month, planned, actual, variance. Click on data point to see breakdown |
| Period markers | Vertical lines for reporting period boundaries |

##### 9.4.3 Cost Category Breakdown

| Element | Details |
|---|---|
| Table | `Table` with columns: Cost Category, Planned Budget, Actual Spend, Remaining, % Used, Status. `Progress` bar in % Used column |
| Categories | Personnel, Travel & Subsistence, Equipment, Subcontracting, Other Direct Costs, Indirect Costs |
| Expand | Each row expandable to show per-partner breakdown |
| Add entry | `Button` "Add Expense" — opens `Dialog` with: date, category, amount, partner, description, receipt upload |

##### 9.4.4 Per-Partner Budget View

| Element | Details |
|---|---|
| Toggle | `Tabs` sub-tabs: "By Category" (default) / "By Partner" |
| By Partner view | `Table`: Partner name, Allocated Budget, Spent, Remaining, % Used. Expandable to show category breakdown per partner |
| Stacked bar chart | Recharts `BarChart` — one bar per partner, stacked by cost category. Color-coded |

##### 9.4.5 Co-financing Tracking

| Element | Details |
|---|---|
| Card | Separate `Card` for co-financing: required amount, confirmed amount, gap. Sources table: source name, type (in-kind / cash), amount, status (confirmed / pending), documentation link |
| Warning | `Alert` variant `destructive` if co-financing gap exists: "Co-financing gap of EUR {X}. {N} sources pending confirmation." |

#### 9.5 Deliverables Tab (`/active-grants/{grantId}/deliverables`)

| Element | Details |
|---|---|
| View toggle | `ToggleGroup`: Table / Timeline / Kanban |

##### 9.5.1 Deliverables Table

| Column | Details |
|---|---|
| Number | D1.1, D2.1, etc. — formatted as `text-sm font-mono` |
| Title | Deliverable title, clickable to open detail |
| WP | Work package number |
| Type | `Badge`: Report, Software, Data, Demonstrator, Other |
| Lead Partner | Organization name with avatar |
| Due Date | Date + relative time. Red if overdue, amber if <30 days |
| Status | `Select` inline: Not Started, In Progress, Under Review, Submitted, Accepted. Color-coded |
| Linked Report Section | Link to the report section that references this deliverable |
| Actions | `DropdownMenu`: Edit, Upload File, Link to Report, Mark Complete |

##### 9.5.2 Deliverables Timeline

Horizontal timeline (Gantt-style) with deliverables as bars or points, grouped by work package. Color-coded by status. Current date marker.

##### 9.5.3 Deliverables Kanban

Five columns: Not Started | In Progress | Under Review | Submitted | Accepted. Draggable cards. Each card shows deliverable number, title, due date, lead partner avatar.

##### 9.5.4 Deliverable Detail (Sheet)

| Element | Details |
|---|---|
| Opens as | `Sheet` from right (480px) |
| Header | Deliverable number + title, status badge, WP reference |
| Details | Description, type, lead partner, contributors, due date, submission date (if submitted) |
| Files | Uploaded deliverable files. Drag-and-drop upload area. Version history of uploads |
| Report link | Section of the periodic report that covers this deliverable. Click to navigate to report editor at that section |
| Comments | Thread of comments from team members |
| History | Activity log: status changes, file uploads, assignments |

#### 9.6 Sidebar Navigation ("Active Grants" Section)

| Element | Details |
|---|---|
| Section header | "Active Grants" with `Trophy` icon |
| Items | List of active grants (max 5 shown, "View all" link). Each: project acronym + next deadline indicator |
| Quick create | Not applicable — grants are marked as "won" from the grants pipeline |

---

### Interactions

| Action | System Response |
|---|---|
| User navigates to Active Grants | List loads with grants where `status === 'awarded'`. Sorted by nearest deadline by default |
| User clicks "Create Report" | Template selection dialog. After selection: report created with section structure, AI pre-fill begins automatically. Progress dialog: "Analyzing project data… Generating WP summaries… Pre-filling financial data…". On completion: navigate to report editor with pre-filled sections highlighted. `toast` "Report draft generated — review highlighted sections" |
| User clicks "Generate Draft" on existing report | AI re-generates all or selected sections. Confirmation: "This will regenerate AI content in {N} sections. Your manual edits will be preserved. Regenerated sections will be highlighted for review." |
| User edits report in Tiptap | Standard rich-text editing. Auto-save every 30 seconds. Section completion status updates in real-time in the navigation sidebar |
| User clicks "Submit for Review" | Validation runs. If incomplete sections: `AlertDialog` listing incomplete sections with links. If complete: status changes to "In Review". Assigned reviewers notified. `toast` "Report submitted for review" |
| User clicks "Export" | Server-side generation. PDF formatted per programme requirements (e.g., Horizon Europe template with EU logo, GA number, report period in header). DOCX for further editing. Downloaded with structured filename: `{acronym}_periodic_report_P{N}_{date}.pdf` |
| User adds expense entry | `Dialog` with form. On save: POST to API. Budget charts and tables update. If new total exceeds planned for category: `toast` warning "Personnel costs now exceed planned budget by 8%" |
| User changes deliverable status (drag in Kanban or select in table) | PATCH to API. Status updates. If marked "Submitted": prompt to upload file. If status change triggers a report update: `toast` "Deliverable D2.1 marked as submitted — this will be reflected in your next periodic report" |
| User views budget tracking | Charts load with cached data, refresh on focus. Loading: skeleton charts. Hover interactions on charts show detailed tooltips |

---

### Responsive Behavior

| Breakpoint | Behavior |
|---|---|
| **Mobile (<640px)** | Active Grants list: single-column cards, simplified metrics. Detail page: tabs render as a scrollable horizontal tab bar. Report editor: full-width, section nav as a `Sheet` drawer. Budget charts: simplified (single series, no hover details). Deliverables: table view with horizontal scroll or Kanban with horizontal scroll. All `Sheet` components are full-screen |
| **Tablet (640–1023px)** | Two-column grant cards. Detail tabs work normally. Report editor: section nav as collapsible left panel. Charts render at full width. Deliverables Kanban shows 3 columns with horizontal scroll for remaining |
| **Desktop (1024px+)** | Three-column grant cards or full table. Detail page with spacious tab content. Report editor: persistent section nav (240px) + full editor. Budget charts with full interactivity. Deliverables Kanban shows all 5 columns |

---

### Data Sources

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/active-grants` | GET | List active (awarded) grants. Query: `programme`, `status`, `q`, `sort`, `page` |
| `/api/active-grants/{id}` | GET | Full grant detail with overview data |
| `/api/active-grants/{id}/reports` | GET | List reports for this grant. Query: `type`, `status`, `period` |
| `/api/active-grants/{id}/reports` | POST | Create new report. Body: `{ type, period, templateId }` |
| `/api/active-grants/{id}/reports/{reportId}` | GET | Fetch report with all section content |
| `/api/active-grants/{id}/reports/{reportId}` | PATCH | Update report sections. Body: partial section data |
| `/api/active-grants/{id}/reports/{reportId}/generate` | POST | Trigger AI draft generation. Body: `{ sections?: string[] }` |
| `/api/active-grants/{id}/reports/{reportId}/export` | POST | Export report. Body: `{ format: 'pdf' \| 'docx' }` |
| `/api/active-grants/{id}/reports/{reportId}/submit` | POST | Submit report for review |
| `/api/active-grants/{id}/budget` | GET | Budget overview (planned vs actual, per category, per partner) |
| `/api/active-grants/{id}/budget/expenses` | GET/POST | List/create expense entries |
| `/api/active-grants/{id}/budget/expenses/{expenseId}` | PATCH/DELETE | Update/delete expense |
| `/api/active-grants/{id}/deliverables` | GET | List deliverables with status |
| `/api/active-grants/{id}/deliverables/{deliverableId}` | PATCH | Update deliverable (status, files, etc.) |
| `/api/active-grants/{id}/deliverables/{deliverableId}/files` | POST | Upload deliverable file |
| `/api/active-grants/{id}/co-financing` | GET | Co-financing tracking data |
| `/api/active-grants/{id}/timeline` | GET | Timeline data for overview chart |

TanStack Query keys: `['active-grants', filters]`, `['active-grant', id]`, `['grant-reports', grantId]`, `['grant-report', reportId]`, `['grant-budget', grantId]`, `['grant-expenses', grantId]`, `['grant-deliverables', grantId]`, `['grant-cofinancing', grantId]`

---

### UX Requirements

| ID | Requirement |
|---|---|
| **UX-P41** | The report calendar must clearly distinguish between reporting deadlines less than 30 days away (amber), overdue deadlines (red with animation), and future deadlines (default), using both color and iconography |
| **UX-P42** | AI-generated report content must be visually distinct from user-written content (colored left border + label) and must never be submitted without explicit user review — the "Submit" action must verify all AI sections have been reviewed (accepted or edited) |
| **UX-P43** | Budget tracking charts must update within 2 seconds of a new expense entry, providing immediate visual feedback on budget impact |
| **UX-P44** | The deliverables Kanban board must support drag-and-drop with haptic feedback (visual snap) and must auto-save status changes without requiring a separate save action |
| **UX-P45** | Report export must generate documents formatted precisely to the programme's requirements (correct headers, page numbering, EU branding for Horizon Europe), not generic formatting |
| **UX-P46** | Section assignments in the report editor must trigger in-app notifications to assigned team members, including a direct link to their assigned section |
| **UX-P47** | The budget burn-rate indicator must proactively warn when the project is on track to exhaust funds before the project end date, with at least 3 months of lead time |
| **UX-P48** | Co-financing tracking must display a clear gap analysis when confirmed co-financing falls below required levels, with actionable guidance on next steps |
| **UX-P49** | Active Grant detail pages must load the Overview tab within 1 second, with progressive loading for charts and metrics (skeleton states for each card/chart independently) |

---

## SCREEN 10: Regulation Tracker (Admin)

**Route:** `/admin/regulations` (main view), `/admin/regulations/{regulationId}` (regulation detail)

**Access:** Admin role only. Enterprise tier includes full regulation tracking with automated monitoring. Professional tier includes read-only access to the regulation feed (no configuration). Free tier: not available.

**Purpose:** Monitor changes to Bulgarian public procurement law (ZOP), EU procurement directives, and grant programme rules, alerting administrators to regulatory changes that affect compliance frameworks and active opportunities.

**Layout:** Standard app shell. Page within the admin section of the sidebar (`/admin/*`). Main content area uses a two-panel layout: left panel for the regulation feed (flex-1), right panel for monitored regulations list (380px, collapsible). On selecting a feed entry, it expands inline (accordion-style) rather than navigating away.

---

### Components

#### 10.1 Page Header

| Element | Details |
|---|---|
| Breadcrumb | Admin > Regulation Tracker |
| Title | "Regulation Tracker" — `text-2xl font-semibold` |
| Subtitle | `text-sm text-muted-foreground` "Monitoring {N} regulations · {X} changes pending review" |
| Primary action | `Button` "Add Regulation" — opens dialog to add a new regulation to monitor |
| Secondary action | `Button` variant `outline` "Settings" — opens monitoring settings (check frequency, notification preferences) |
| Alert banner | If breaking changes pending: `Alert` variant `destructive` at top of page: "{X} breaking regulatory changes require immediate review" with `Button` "Review Now" |

#### 10.2 Regulation Feed (Left Panel)

Chronological feed of detected regulatory changes. Most recent first.

##### 10.2.1 Feed Toolbar

| Element | Details |
|---|---|
| Search | `Input` with `Search` icon. Placeholder: "Search regulation changes…". Searches change summaries and regulation names |
| Severity filter | `ToggleGroup`: All, Breaking, Significant, Informational. Each with count badge |
| Status filter | `Select`: All, Pending Review, Reviewed, Dismissed |
| Date range | `DateRangePicker` (shadcn calendar-based) for filtering by detection date |
| Sort | `Select`: Newest First (default), Oldest First, Severity (high to low) |

##### 10.2.2 Feed Entries

Each entry rendered as an expandable `Card` in a vertical feed.

**Collapsed state:**

| Element | Details |
|---|---|
| Severity indicator | Left border: 4px solid — Red (`border-destructive`) for Breaking, Amber (`border-warning`) for Significant, Blue (`border-info`) for Informational |
| Severity badge | `Badge` in top-right: "Breaking" (variant `destructive`), "Significant" (amber/warning styling), "Informational" (variant `secondary`) |
| Regulation name | `text-sm font-semibold` — e.g., "Law on Public Procurement (ZOP)" or "Directive 2014/24/EU" |
| Change summary | `text-sm` — one-line summary. Max 120 chars with ellipsis. e.g., "Amendment to Article 79 — new thresholds for direct award procedures effective January 2026" |
| Date detected | `text-xs text-muted-foreground` — relative time + absolute date on hover |
| Source link | `Button` variant `link` size `sm` "Source" — external link to official publication (opens in new tab) |
| Impact indicator | `text-xs`: "{N} frameworks affected" with `AlertTriangle` icon if N > 0 |
| Status | `Badge` variant `outline`: "Pending Review" (amber dot), "Reviewed" (green dot), "Dismissed" (gray dot) |
| Expand button | `ChevronDown` icon — click to expand |

**Expanded state (accordion):**

| Section | Details |
|---|---|
| Full change description | `text-sm` — complete description of the regulatory change (may be AI-summarized from source) |
| Before/After diff | `Card` with diff view: left column "Previous Text", right column "New Text". Additions highlighted green background, removals highlighted red background with strikethrough. Scrollable if long. Uses a custom diff component or `react-diff-viewer` |
| Affected compliance frameworks | `Table` compact: Framework name (linked to framework editor), Last reviewed date, Impact assessment (`Badge`: High / Medium / Low), Status (`Badge`: "Update Needed" (red), "Up to Date" (green)) |
| Affected opportunities | `Table` compact: Opportunity title (linked), Status, Compliance framework used. Only shown if active opportunities use affected frameworks |
| Analysis notes | `Textarea` — admin can write analysis notes about this change. Saved via auto-save |
| Actions | `Button` "Update Framework" (primary, opens framework editor with the affected framework pre-selected and the change context loaded), `Button` "Dismiss" (outline, requires a note in a `Popover`), `Button` "Create Alert" (outline, sends notification to affected users), `Button` "Mark Reviewed" (outline, changes status to Reviewed) |

#### 10.3 Monitored Regulations (Right Panel)

Collapsible panel showing all regulations being monitored. Width: 380px on desktop.

##### 10.3.1 Panel Header

| Element | Details |
|---|---|
| Title | "Monitored Regulations" — `text-base font-semibold` |
| Count | `Badge` "{N} active" |
| Collapse toggle | `Button` variant `ghost` with `PanelRightClose` icon |
| Search | `Input` compact, `Search` icon. Filters the regulation list |

##### 10.3.2 Regulation List

Each monitored regulation as a `Card` compact in a vertical list.

| Element | Details |
|---|---|
| Regulation name | `text-sm font-medium` — e.g., "Закон за обществените поръчки (ZOP)" |
| Jurisdiction | `Badge` variant `outline`: "BG" (Bulgarian flag), "EU" (EU flag), or specific member state |
| Category | `Badge` variant `secondary`: "Procurement Law", "Grant Programme Rules", "Financial Regulation", "Environmental Directive" |
| Monitoring status | `Badge`: "Active" (green), "Paused" (amber), "Deprecated" (gray) |
| Last checked | `text-xs text-muted-foreground` — "Checked 2 hours ago" or "Last checked: {date}" |
| Check frequency | `text-xs text-muted-foreground` — "Daily" / "Weekly" / "Monthly" |
| Linked frameworks | `text-xs` — "{N} frameworks" with count. Clickable to see list in tooltip |
| Changes count | `text-xs` — "{N} changes detected" total. Clickable to filter feed to this regulation |
| Actions | `DropdownMenu` (`...` button): Edit, Configure Check Frequency, Pause/Resume Monitoring, Remove (with confirmation) |

##### 10.3.3 Add Regulation Dialog

Triggered by "Add Regulation" button in page header.

| Field | Details |
|---|---|
| Regulation name | `Input` — required. e.g., "Law on Public Procurement (ZOP)" |
| Official reference | `Input` — e.g., "Directive 2014/24/EU", "SG No. 13/2020" |
| Jurisdiction | `Select`: Bulgaria, European Union, [other EU member states]. Required |
| Category | `Select`: Procurement Law, Grant Programme Rules, Financial Regulation, Environmental Directive, Social Policy, Data Protection, Custom |
| Source URL | `Input` type URL — link to official source for automated checking |
| Check frequency | `Select`: Daily, Weekly, Monthly. Default: Weekly |
| Linked frameworks | Multi-select `Combobox` — select existing compliance frameworks to link |
| Notes | `Textarea` — optional admin notes |
| Action | `Button` "Start Monitoring" (primary), `Button` "Cancel" (ghost) |

#### 10.4 Framework Integration Indicators

When a regulation change affects a compliance framework:

| Element | Where | Details |
|---|---|---|
| Review badge | Framework Editor page | `Badge` variant `destructive` "Regulation Changed — Review Needed" appears next to framework title. Clicking navigates to the relevant regulation feed entry |
| Framework list | Compliance Frameworks list page | Affected frameworks show a `AlertTriangle` icon with tooltip "Regulation change detected — review needed" |
| Count indicator | Sidebar nav "Compliance" item | Red dot notification with count of frameworks needing review |

#### 10.5 Notification System

| Notification Type | Channel | Content |
|---|---|---|
| Breaking change detected | In-app notification (bell icon) + email (if enabled) | "Breaking regulatory change: {summary}. {N} compliance frameworks affected. [Review Now]" |
| Significant change detected | In-app notification | "Significant regulatory change: {summary}. [View Details]" |
| Informational change detected | In-app notification (batched daily) | "Regulatory update: {N} informational changes detected today. [View Feed]" |
| Framework review reminder | In-app notification | "Reminder: {framework name} has a pending regulatory review from {date}. [Review]" — sent 7 days after change if not reviewed |
| User alert (admin-created) | In-app notification + email to selected users | Custom message from admin via "Create Alert" action. Admin selects recipients (all users, specific roles, users with affected opportunities) |

#### 10.6 Admin Dashboard Widget

| Element | Details |
|---|---|
| Location | Admin dashboard (`/admin`) — top-right widget area |
| Widget | `Card`: "Regulation Changes" — "{X} changes pending review". Breakdown: {A} breaking, {B} significant, {C} informational. `Button` "Review" linking to `/admin/regulations` filtered to pending. Shows last 3 changes as compact list |

#### 10.7 Empty States

| State | Display |
|---|---|
| No regulations monitored | Illustration (shield icon) + "No regulations monitored yet" + "Add regulations to track changes that affect your compliance frameworks" + `Button` "Add First Regulation" |
| No changes in feed | "No regulatory changes detected" + "All monitored regulations are up to date. Last check: {time}" |
| No changes matching filters | "No changes matching your filters" + `Button` "Clear Filters" |

---

### Interactions

| Action | System Response |
|---|---|
| User clicks "Add Regulation" | Dialog opens. On submit: POST to API. New regulation appears in monitored list. First check scheduled immediately. `toast` "Now monitoring {regulation name}" |
| User expands feed entry | Smooth accordion expand with 200ms transition. Full change details, diff, and affected items load. If data not cached: skeleton loading for diff and affected items (separate API calls) |
| User clicks "Update Framework" | Navigates to `/admin/compliance/frameworks/{frameworkId}/edit` with query param `?regulationChange={changeId}`. Framework editor loads with the change context displayed in an info banner at the top, showing what changed and suggesting which sections to review |
| User clicks "Dismiss" | `Popover` with `Textarea` for dismissal note (required, min 10 chars). On confirm: PATCH status to Dismissed. Entry grayed out in feed (still visible with "Dismissed" filter). `toast` "Change dismissed" |
| User clicks "Create Alert" | `Dialog`: select recipients (All Users, by role, by affected opportunities). Custom message `Textarea` (pre-filled with change summary). Preview of notification. `Button` "Send Alert". On confirm: notifications dispatched. `toast` "Alert sent to {N} users" |
| User clicks "Mark Reviewed" | PATCH status to Reviewed. Badge updates to green "Reviewed". If linked frameworks were updated: framework review badge cleared. `toast` "Change marked as reviewed" |
| User configures check frequency | `Select` inline on regulation card or in edit dialog. PATCH to API. `toast` "Check frequency updated to {frequency}" |
| User pauses monitoring | PATCH status to Paused. `toast` "Monitoring paused for {regulation}". Card shows "Paused" badge. Resume available from same dropdown |
| User removes a monitored regulation | `AlertDialog`: "Stop monitoring {regulation}? Historical changes will be preserved but no new changes will be detected." On confirm: PATCH status to inactive. Removed from monitored list. Historical feed entries remain. `toast` "Stopped monitoring {regulation}" |
| Automated check detects change | Background job creates feed entry. If breaking: immediate in-app notification + email. If significant: in-app notification. If informational: batched into daily digest. Admin dashboard widget updates |

---

### Responsive Behavior

| Breakpoint | Behavior |
|---|---|
| **Mobile (<640px)** | Single-column layout. Feed takes full width. Monitored regulations accessible via `Sheet` triggered by a `Button` "Monitored ({N})" at top of feed. Feed entries: severity badge and summary only in collapsed state; expand shows full detail in card. Diff view stacks vertically (before above, after below) instead of side-by-side. Actions as a bottom action bar in expanded entries |
| **Tablet (640–1023px)** | Feed takes full width; monitored regulations as a collapsible right panel (320px) or accessible via tab toggle ("Feed" / "Monitored"). Diff view side-by-side if width allows, otherwise stacked. Expanded feed entries at full width |
| **Desktop (1024px+)** | Two-panel layout: feed (flex-1) + monitored regulations (380px). Both visible simultaneously. Diff view always side-by-side. Full toolbar visible. All expanded entry details in spacious layout |

---

### Data Sources

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/admin/regulations/feed` | GET | List regulation changes. Query: `q`, `severity`, `status`, `regulationId`, `dateFrom`, `dateTo`, `sort`, `page`, `limit` |
| `/api/admin/regulations/feed/{changeId}` | GET | Full change detail (diff, affected frameworks, affected opportunities) |
| `/api/admin/regulations/feed/{changeId}` | PATCH | Update change status, add analysis notes. Body: `{ status, notes, dismissalReason }` |
| `/api/admin/regulations/feed/{changeId}/alert` | POST | Create and send alert to users. Body: `{ recipients, message }` |
| `/api/admin/regulations/monitored` | GET | List monitored regulations. Query: `q`, `jurisdiction`, `status` |
| `/api/admin/regulations/monitored` | POST | Add new regulation. Body: `{ name, reference, jurisdiction, category, sourceUrl, checkFrequency, linkedFrameworkIds, notes }` |
| `/api/admin/regulations/monitored/{id}` | PATCH | Update regulation settings (frequency, status, linked frameworks) |
| `/api/admin/regulations/monitored/{id}` | DELETE | Remove regulation from monitoring |
| `/api/admin/regulations/stats` | GET | Dashboard widget data: pending counts by severity, recent changes |
| `/api/admin/compliance/frameworks/{id}/regulation-review` | GET | Check if a framework has pending regulatory reviews |

TanStack Query keys: `['regulation-feed', filters]`, `['regulation-change', changeId]`, `['monitored-regulations', filters]`, `['regulation-stats']`

---

### UX Requirements

| ID | Requirement |
|---|---|
| **UX-P50** | Breaking regulatory changes must be surfaced with high-urgency visual treatment (red border, red badge, optional subtle pulse animation on first view) and must appear in the admin dashboard widget without requiring navigation to the regulation tracker |
| **UX-P51** | The before/after diff view must clearly highlight additions (green background) and removals (red background with strikethrough) at the word level, not just line level, for precise identification of changes |
| **UX-P52** | Dismissing a regulatory change must require a written justification (minimum 10 characters) to maintain an audit trail of why changes were deemed not applicable |
| **UX-P53** | The "Create Alert" flow must show a preview of the notification as recipients will see it, and must require explicit confirmation before dispatching to prevent accidental mass notifications |
| **UX-P54** | Affected compliance frameworks must be directly linked from the regulation feed entry — clicking a framework name must navigate to the framework editor with the regulatory change context pre-loaded |
| **UX-P55** | Monitoring check timestamps must be visible for each regulation, showing when the last automated check occurred and when the next check is scheduled, so admins can trust the system is actively monitoring |
| **UX-P56** | The regulation feed must support real-time updates via WebSocket or SSE — when a new regulatory change is detected while an admin is viewing the feed, a banner must appear ("New regulatory change detected — [Refresh]") without auto-refreshing and losing scroll position |
| **UX-P57** | Feed entries must remain accessible after being dismissed or reviewed, filterable by status, to maintain a complete audit history of all detected regulatory changes and administrative responses |
| **UX-P58** | Framework review badges triggered by regulation changes must persist until explicitly cleared by an admin marking the change as reviewed, even across page navigations and sessions |

---

## SCREEN 11: Company Profile Vault

**Route:** `/settings/company-profile`

**Access:** All authenticated users. Free tier: basic profile only (company name, sector, region). Starter+: full profile. Enterprise: multi-entity profiles (parent company + subsidiaries). Roles: Admin and Bid Manager have full CRUD. Contributors can edit their own CV/credentials. Read-only users can view but not edit.

**Purpose:** Centralized management of all company credentials, team CVs, certifications, past project references, and financial data that auto-populates into proposals, ESPD forms, and eligibility checks via RAG.

### Layout

Standard app shell — sidebar with "Settings" expanded, "Company Profile" selected. Main content area with a vertical tab navigation on the left (desktop) or horizontal scrollable pills (mobile).

### Profile Sections (Vertical Tabs)

| Tab | Content | Auto-Populates Into |
|-----|---------|-------------------|
| **Overview** | Company name, legal name, registration number, VAT, address, sector(s), size, founding year, website | ESPD Part II, proposal headers, eligibility checks |
| **Team & CVs** | Team member cards: name, role, qualifications, experience summary, attached CV (PDF). Add/edit/archive members. | Proposal team composition section, ESPD Part IV |
| **Certifications** | ISO standards, environmental certs, security clearances, professional licenses. Each: name, issuer, number, issue date, expiry date, scan (PDF). | ESPD Part IV, compliance checks, eligibility |
| **Past Projects** | Reference projects: title, client, value, dates, description, outcomes, relevance tags. Filterable. | Proposal past experience section, eligibility |
| **Financial** | Annual turnover (last 3 years), balance sheet summary, insurance policies (type, provider, coverage, expiry). Upload financial statements (PDF). | ESPD Part IV, eligibility, budget builder |
| **Legal & Compliance** | Exclusion ground declarations, tax compliance status, legal representatives, beneficial owners. | ESPD Part III, compliance checks |
| **Content Defaults** | Default company description, quality management overview, sustainability policy, methodology overview — short text blocks auto-inserted into proposals when relevant. | Proposal boilerplate sections via RAG |

### Components Per Tab

#### Overview Tab

| Component | Details |
|-----------|---------|
| Form layout | Two-column grid (desktop), single column (mobile). React Hook Form + Zod validation. |
| Fields | Company name (`Input`), legal name (`Input`), registration no. (`Input`), VAT number (`Input` with EU VAT format validation), address (structured: street, city, postal code, country `Select`), primary sector (`Select` with CPV search), secondary sectors (`MultiSelect`), company size (`Select`: micro/small/medium/large), founding year (`Input` type number), website (`Input` URL). |
| Logo upload | Dropzone (128x128 preview) for company logo. Used in proposals and white-label. |
| Completeness indicator | Progress ring at top: "Profile 72% complete" with list of missing fields below. Links to the specific tab/field. |
| Save | Auto-save with debounce (1s after last edit). "Saved" indicator with timestamp. Manual "Save" button also available. |

#### Team & CVs Tab

| Component | Details |
|-----------|---------|
| Team list | Card grid (desktop: 3 columns, tablet: 2, mobile: 1). Each card: avatar (initials if no photo), name, role badge, qualification tags, "Last updated" date. |
| Add member | `Button` "+ Add Team Member" → `Dialog` with form: name, email (optional, for linking to platform user), role (`Select`), qualifications (`TagInput`), experience summary (`Textarea` 500 chars), CV upload (`FileInput` PDF, max 10MB). |
| Edit member | Click card → right inspector panel (desktop) or full-screen `Sheet` (mobile). Same fields as create, plus: full CV preview (inline PDF viewer), edit history. |
| Archive | Cards can be archived (not deleted) — archived members don't appear in proposal auto-population but data is preserved. Toggle "Show archived" filter. |
| Proposal integration | Badge on each card: "Used in 5 proposals" — click shows list of proposals where this CV was included. |
| Drag to reorder | Team order determines default display order in proposals. Drag handles on cards. |

#### Certifications Tab

| Component | Details |
|-----------|---------|
| Cert list | `DataTable` with columns: name, issuer, certificate number, issued date, expiry date, status badge, actions. |
| Status badges | `Badge` — Valid (green), Expiring Soon (amber, ≤90 days to expiry), Expired (red), Pending Renewal (blue). |
| Add cert | `Button` "+ Add Certification" → `Dialog`: cert name (`Select` with common options + custom), issuer (`Input`), certificate number (`Input`), issue date (`DatePicker`), expiry date (`DatePicker`, optional for perpetual), scan upload (`FileInput` PDF). |
| Expiry alerts | Certs expiring within 90 days show amber warning row. Expiring within 30 days: red + notification queued. Auto-notification to admin when cert expires. |
| Bulk upload | "Import certifications" button — upload structured CSV or manually enter batch. |

#### Past Projects Tab

| Component | Details |
|-----------|---------|
| Project list | `DataTable` with columns: title, client, value (EUR), dates, relevance tags, "Used in X proposals". Sortable, filterable. |
| Add project | `Button` "+ Add Project" → full form page: title (`Input`), client name (`Input`), contract value (`Input` EUR), start date (`DatePicker`), end date (`DatePicker`), description (`Textarea` 2000 chars), key outcomes (`TagInput`), sector tags (`MultiSelect` CPV), attachments (reference letters, completion certificates). |
| Relevance tagging | Each project tagged with CPV sectors and capability areas. Used by the Smart Matching Agent to compute relevance scores and by the Proposal Generator to select relevant references. |
| Quick-add from bid outcome | When recording a bid outcome as "Won", prompt: "Add this to your Past Projects?" Pre-fills from opportunity data. |

#### Financial Tab

| Component | Details |
|-----------|---------|
| Turnover table | 3-row table: Year, Annual Turnover (EUR), Net Profit (EUR). Editable inline. |
| Statement upload | Upload area for annual financial statements (PDF). One per year. Virus scanned. |
| Insurance list | `DataTable`: policy type (Professional Indemnity, Public Liability, etc.), provider, coverage amount, expiry date, status badge. |
| Threshold indicator | Shows which tender budget ranges the company qualifies for based on turnover (e.g., "Your turnover qualifies you for tenders up to €5M based on standard 3x annual turnover rule"). |

### Responsive Behavior

| Breakpoint | Adaptation |
|-----------|-----------|
| Desktop (1024+) | Vertical tab nav (200px) + content area. Inspector panel for team member detail. |
| Tablet (768–1024) | Horizontal scrollable pill tabs above content. Inspector → Sheet from right. |
| Mobile (<768) | Horizontal scrollable pill tabs. Cards single-column. Forms full-width. Team cards stack vertically. DataTables → card lists. |

### Data Sources

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/v1/companies/{id}/profile` | GET/PUT | Company overview fields |
| `/api/v1/companies/{id}/team-members` | GET/POST/PUT/DELETE | Team CRUD |
| `/api/v1/companies/{id}/team-members/{id}/cv` | POST/GET | CV upload/download |
| `/api/v1/companies/{id}/certifications` | GET/POST/PUT/DELETE | Certification CRUD |
| `/api/v1/companies/{id}/projects` | GET/POST/PUT/DELETE | Past project CRUD |
| `/api/v1/companies/{id}/financials` | GET/PUT | Financial data |
| `/api/v1/companies/{id}/legal` | GET/PUT | Legal & compliance declarations |
| `/api/v1/companies/{id}/content-defaults` | GET/PUT | Default content blocks |
| `/api/v1/companies/{id}/profile/completeness` | GET | Completeness score + missing fields |

### Empty State

First visit (no data): "Build your company profile" illustration + "A complete profile powers AI proposals, ESPD auto-fill, and relevance matching. Start with your company overview." + `Button` "Get Started" → scrolls to Overview tab with first field focused.

### UX Requirements

| ID | Requirement |
|----|-------------|
| **UX-PV01** | Profile completeness indicator must be visible on every tab as a persistent progress ring, showing percentage and listing specific missing fields as clickable links that navigate to the relevant tab and field |
| **UX-PV02** | Team member cards must support drag-to-reorder for controlling display order in generated proposals. Archived members must be visually distinct (grayscale, "Archived" badge) and excluded from proposal auto-population |
| **UX-PV03** | Certification expiry tracking must show status badges (Valid/Expiring Soon/Expired) calculated from the current date. Certifications expiring within 90 days must trigger an amber warning; within 30 days, a red warning and an in-app notification to the admin |
| **UX-PV04** | Past projects must be taggable with CPV sectors and capability areas. The tag system must match the CPV picker used in alert preferences for consistency. Projects tagged as relevant to a specific tender must appear highlighted in proposal generation |
| **UX-PV05** | Financial threshold indicator must dynamically calculate qualification thresholds (e.g., "qualifies for tenders up to €X") based on entered turnover data, using the standard EU 3x annual turnover rule |
| **UX-PV06** | All profile sections must auto-save with a 1-second debounce after the last edit. A "Saved at HH:MM" indicator must confirm persistence. Manual save must also be available for explicit confirmation |
| **UX-PV07** | When a bid outcome is recorded as "Won", the system must prompt "Add to Past Projects?" and pre-fill project fields from the opportunity record (title, client from contracting authority, contract value from award data) |

---


## SECTION 3: Component Additions to Existing Screens

These are components and panels that must be added to screens already defined in the UX specification. Each entry specifies the component, its host screen, interactive behavior, data dependencies, and responsive adaptation.

---

### 3.1 Relevance Score Badge (Opportunity List + Detail)

**Host screens:** Opportunity List (`/opportunities`), Opportunity Detail (`/opportunities/[id]`)

**List view — badge component:**

| Attribute | Specification |
|---|---|
| Shape | 32 × 32 px circular ring (SVG `<circle>` with `stroke-dasharray` proportional to score). Numeric score centered inside in `text-xs font-semibold`. |
| Color mapping | 80–100 → `text-green-600 stroke-green-500` / 60–79 → `text-amber-600 stroke-amber-500` / 0–59 → `text-muted-foreground stroke-muted` |
| Placement — table layout | Rightmost column before the row-action kebab menu, column header "Match". |
| Placement — card layout | Top-right corner of the card, overlapping the card border by 4 px (absolute positioned). |
| Empty state | When no company profile exists: gray ring with `?` inside. |

**Tooltip (hover / long-press on mobile):**

Uses shadcn `TooltipProvider` + `Tooltip`. Content is a 200 px-wide micro-card:

```
Overall: 82 / 100
─────────────────
Sector match       ████████░░  78%
Region match       █████████░  92%
Budget fit         ████████░░  80%
Capability match   ████████░░  79%
```

Each factor row: label (`text-xs`), inline mini progress bar (4 px tall, same color logic), percentage.

**Detail page — expanded relevance card:**

- Placed at the top of the right sidebar (desktop) or as a collapsible section below the header (mobile).
- Card (`shadcn Card`) titled "Relevance Analysis".
- Four horizontal bar charts (Recharts `BarChart` with `layout="vertical"`), one per factor. Each bar is colored per the score-color mapping. Value label at the end of each bar.
- Below bars: brief AI-generated sentence explaining the strongest and weakest factors ("Strong sector match in IT services. Budget exceeds your typical range.").
- Footer: "Last calculated [relative time]" + "Recalculate" ghost button (triggers re-scoring against current profile).

**No-profile CTA:**

- Replaces the relevance card entirely when `companyProfile.isComplete === false`.
- Illustration placeholder (64 × 64 SVG icon of a building with a question mark).
- Heading: "Get relevance scores" (`text-sm font-semibold`).
- Body: "Set up your company profile so we can match opportunities to your capabilities." (`text-xs text-muted-foreground`).
- Button: `<Button size="sm">Set up profile</Button>` → navigates to `/settings/company-profile`.

**Data source:** `GET /api/opportunities/[id]/relevance` returns `{ overall: number, factors: { sector: number, region: number, budget: number, capability: number }, explanation: string, calculatedAt: ISO8601 }`. List endpoint includes `relevanceScore` on each opportunity object.

**Responsive behavior:**

- ≥ 1024 px (desktop): badge in table column; detail card in sidebar.
- 768–1023 px (tablet): badge in card layout top-right; detail card full-width below header.
- < 768 px (mobile): badge inline next to opportunity title as a small pill ("82%"); detail card as collapsible accordion section.

**UX Requirements:**

- **UX-C01:** Every opportunity card/row SHALL display a relevance score badge when a complete company profile exists. The badge SHALL use the three-tier color mapping (green ≥ 80, amber 60–79, gray < 60).
- **UX-C02:** Hovering (desktop) or long-pressing (mobile) the relevance badge SHALL reveal a tooltip showing the four match-factor breakdown with mini progress bars.
- **UX-C03:** The Opportunity Detail page SHALL display an expanded relevance card with vertical bar charts for each factor and an AI-generated explanation sentence.
- **UX-C04:** When no company profile is configured, all relevance badge locations SHALL display a CTA prompting the user to set up their profile. The CTA SHALL link to `/settings/company-profile`.

---

### 3.2 Template Selector (Proposal Generation Flow)

**Host screen:** Proposal Generation within Opportunity Detail (`/opportunities/[id]` → "Generate Proposal Draft" action)

**Trigger:** User clicks the "Generate Proposal Draft" button anywhere it appears (opportunity detail action bar, proposal tab empty state). Instead of immediately invoking WF5, the Template Selector dialog opens.

**Dialog structure (shadcn `Dialog` — `max-w-3xl`):**

```
┌──────────────────────────────────────────────────────┐
│  Choose a Starting Template              [X close]   │
│                                                      │
│  [Filter: sector dropdown] [Sort: popularity ▾]      │
│                                                      │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐   │
│  │ preview │ │ preview │ │ preview │ │ preview │   │
│  │  thumb  │ │  thumb  │ │  thumb  │ │  thumb  │   │
│  │─────────│ │─────────│ │─────────│ │─────────│   │
│  │ Title   │ │ Title   │ │ Title   │ │ Title   │   │
│  │ [tag]   │ │ [tag]   │ │ [tag]   │ │ [tag]   │   │
│  │ 3 wins  │ │ 1 win   │ │ New     │ │ 5 wins  │   │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘   │
│                                                      │
│  ── Or ──────────────────────────────────────────    │
│  ( ) Start from scratch                              │
│  ( ) Use a previous proposal as template → [list]    │
│                                                      │
│                          [Cancel]  [Continue →]      │
└──────────────────────────────────────────────────────┘
```

**Template card detail:**

| Element | Specification |
|---|---|
| Container | 160 × 200 px card. `border rounded-lg`. Selected state: `ring-2 ring-primary`. |
| Preview thumbnail | 160 × 100 px placeholder showing a miniaturized document preview (first-page render or static icon per sector). Background `bg-muted`. |
| Title | `text-sm font-medium`, max 2 lines, truncate with ellipsis. |
| Sector tag | shadcn `Badge variant="secondary"` — e.g., "IT Services", "Construction". |
| Success indicator | `text-xs text-muted-foreground` — "Used in 3 winning proposals" or "New template" if none. |

**Filter bar:**

- Sector dropdown: `Select` component, options populated from the user's active sectors + "All sectors".
- Sort: `Select` — options: "Most popular", "Most recent", "Highest win rate".

**"Use previous proposal" flow:**

- Selecting this radio reveals an inline searchable list (shadcn `Command` / combobox pattern).
- Each item: proposal title, opportunity name, date, outcome badge (Won / Submitted / Draft).
- Filtered to proposals authored by the current user (or all team proposals for bid managers).

**Actions:**

- "Cancel" → closes dialog, no generation triggered.
- "Continue →" → disabled until a selection is made. On click: closes dialog, passes `templateId` (or `previousProposalId`, or `null` for scratch) into the WF5 generation kickoff.

**Data source:** `GET /api/templates?sector=&sort=` for system templates. `GET /api/proposals?mine=true&status=submitted,won` for previous proposals.

**Responsive behavior:**

- ≥ 1024 px: 4-column grid of template cards.
- 768–1023 px: 3-column grid.
- < 768 px: dialog becomes a full-screen sheet (`SheetContent side="bottom"` with `h-[90vh]`). Template cards in a 2-column grid. Scroll within the sheet.

**UX Requirements:**

- **UX-C05:** Clicking "Generate Proposal Draft" SHALL open the Template Selector dialog before any AI generation begins. The dialog SHALL NOT be skippable.
- **UX-C06:** The template grid SHALL support filtering by sector and sorting by popularity, recency, or win rate.
- **UX-C07:** The dialog SHALL offer "Start from scratch" and "Use a previous proposal as template" as alternatives to system templates. The previous-proposal option SHALL present a searchable list of the user's past proposals.
- **UX-C08:** The "Continue" button SHALL remain disabled until the user has made a selection. Upon confirmation, the selected template identifier SHALL be passed to the WF5 generation workflow.

---

### 3.3 Language Selector

**Host screens:** Global header (all pages), AI operation dialogs (summarize, generate proposal, compliance check, translate)

#### 3.3.1 Global Header Language Selector

**Placement:** Right side of the top header bar, between the notification bell and the user avatar menu.

**Component:** shadcn `DropdownMenu` triggered by a button showing the current language's 2-letter code inside a rounded rectangle (e.g., `[EN]` or `[BG]`). No flag icons (flags can be politically sensitive; use ISO codes).

```
┌──────────────────────────────────────────────────┐
│  [logo]  ... nav ...  [🔔] [EN ▾] [avatar]      │
└──────────────────────────────────────────────────┘
                              │
                         ┌─────────┐
                         │ ● EN    │
                         │   BG    │
                         └─────────┘
```

**Behavior:**

- Selection triggers `PATCH /api/user/preferences { uiLanguage: "en" | "bg" }`.
- UI language switches via Next.js i18n routing (URL prefix `/en/`, `/bg/` or cookie-based, depending on architecture — cookie-based recommended for MVP to avoid URL churn).
- Active language shown with a filled radio indicator.
- Transition: page content re-renders; no full reload (leverages Next.js `useRouter` locale switching).

#### 3.3.2 AI Operation Language Selector

**Placement:** Inside every AI action dialog/modal — below the primary action description, above the "Generate" / "Summarize" button.

**Component:** Inline `Select` labeled "Output language":

```
Output language: [English (EN) ▾]
```

- Options: English, Bulgarian (MVP). Architecture supports adding more.
- Default: mirrors current UI language.
- Stored per-operation (not persisted globally — each new dialog defaults to UI language).

#### 3.3.3 Translation Indicator

**Placement:** Document viewer, proposal editor, summary display — anywhere content is rendered in a language different from the UI language.

**Component:** A slim banner (`h-8 bg-blue-50 dark:bg-blue-950 text-blue-700 dark:text-blue-300 text-xs`) pinned at the top of the content area:

```
ℹ Translated from Bulgarian  •  [View original]
```

- "View original" toggles to the source-language version inline (no navigation).

**Data source:** User preferences stored in user profile object. AI output language passed as parameter to generation endpoints.

**Responsive behavior:**

- Header selector: identical at all breakpoints (compact by design).
- AI operation selector: full-width `Select` on mobile, inline on desktop.
- Translation banner: full-width at all breakpoints, text wraps on narrow screens.

**UX Requirements:**

- **UX-C09:** The global header SHALL include a language selector allowing the user to switch the UI language between Bulgarian and English. The selected language SHALL persist in the user's profile across sessions.
- **UX-C10:** Every AI operation dialog (summarize, generate, compliance check) SHALL include an "Output language" selector defaulting to the current UI language. The user SHALL be able to choose a different output language without changing the UI language.
- **UX-C11:** When displayed content was generated in a language different from the current UI language, a translation indicator banner SHALL appear with the source language name and a toggle to view the original.

---

### 3.4 Document Manager (Opportunity Detail Tab)

**Host screen:** Opportunity Detail (`/opportunities/[id]`) — new "Documents" tab added to the existing tab bar.

**Tab position:** After "Proposals" and before "Settings" in the tab order: Overview | Evaluation Criteria | Proposals | **Documents** | Settings.

**Layout (desktop):**

```
┌──────────────────────────────────────────────────────────┐
│  Documents                                               │
│                                                          │
│  ┌─ Storage: 124 MB / 500 MB used ─── ████████░░░ ──┐   │
│  │  Documents retained until 15 Mar 2027             │   │
│  └───────────────────────────────────────────────────┘   │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │  📂 Drop files here or [Upload files]              │  │
│  │     Max 50 MB per file · PDF, DOCX, XLSX, ZIP      │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  Filter: [All categories ▾]  Search: [__________]        │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │ ☐  Name          Category     Size   Date   ⋮    │    │
│  │ ☐  📄 Tender_spec Tender docs  2.4MB  Mar 1  ⋮    │    │
│  │ ☐  📄 Draft_v2    AI-generated  890KB  Mar 3  ⋮    │    │
│  │ ☐  📄 Budget.xlsx User upload   1.1MB  Mar 3  ⋮    │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  Showing 3 of 3 documents                                │
└──────────────────────────────────────────────────────────┘
```

**Upload zone:**

- Drag-and-drop area: 100 % width, 80 px height, dashed border (`border-2 border-dashed border-muted-foreground/25 rounded-lg`). On drag-over: border becomes `border-primary`, background `bg-primary/5`.
- "Upload files" button inside the zone: `<Button variant="outline" size="sm">`.
- Accepts: `.pdf`, `.docx`, `.xlsx`, `.pptx`, `.zip`, `.jpg`, `.png`. Max 50 MB per file.
- On upload: file appears immediately in the list with an animated progress bar in place of the size column. Status column shows "Uploading… 45%". After upload completes, status transitions to "Scanning…" (virus scan), then "Clean ✓" or "Infected ✗" (file quarantined, row tinted red, download disabled).

**File list table columns:**

| Column | Width | Content |
|---|---|---|
| Checkbox | 40 px | Bulk select for batch download/delete. |
| Type icon | 40 px | File-type icon (PDF red, DOCX blue, XLSX green, image purple, other gray). Using Lucide icons. |
| Name | flex | File name, truncated with tooltip. Click opens preview. |
| Category | 120 px | Badge — "Tender docs" (blue), "AI-generated" (purple), "User upload" (gray), "Export" (green). Auto-assigned; editable via dropdown. |
| Size | 80 px | Human-readable (e.g., "2.4 MB"). |
| Uploaded | 100 px | Relative date ("2 days ago"). Tooltip: absolute date + uploader name. |
| Scan status | 80 px | Icon + text: `✓ Clean` (green), `⏳ Scanning` (amber spinner), `✗ Infected` (red). |
| Actions | 40 px | Kebab menu (`DropdownMenu`): Download, Preview, Share link, Rename, Change category, Delete. |

**Bulk actions bar:** Appears when ≥ 1 checkbox is selected. Sticky at bottom of the file list. Buttons: "Download selected", "Delete selected". Delete triggers a confirmation dialog listing file names.

**Preview:** Clicking a file name or "Preview" opens a right-side slide-over panel (`Sheet side="right"` at 50 % width on desktop). PDF rendered inline via `<iframe>` or `react-pdf`. DOCX/XLSX show a "Preview not available — download to view" message with a download button. Images render directly.

**Share link:** Generates a time-limited (7-day default, configurable) signed URL. Copies to clipboard with toast confirmation.

**Delete confirmation:** shadcn `AlertDialog` — "Delete [filename]? This action cannot be undone." Destructive button styled `variant="destructive"`.

**Storage meter:**

- Positioned above the upload zone.
- Progress bar: `h-2 rounded-full`. Color: green < 70 %, amber 70–89 %, red ≥ 90 %.
- Text: "[used] MB / [limit] MB used" left-aligned. Retention date right-aligned.

**Data source:** `GET /api/opportunities/[id]/documents` returns paginated file list. `POST /api/opportunities/[id]/documents` for upload (multipart). `DELETE /api/opportunities/[id]/documents/[docId]`. Storage metadata from opportunity object.

**Responsive behavior:**

- ≥ 1024 px: full table layout as described. Preview in side sheet.
- 768–1023 px: table drops "Scan status" column. Preview as bottom sheet.
- < 768 px: file list becomes card layout — each file is a card with name, category badge, size, and date. Actions via long-press context menu or a visible kebab button. Upload zone height reduced to 60 px. Preview opens as full-screen overlay.

**UX Requirements:**

- **UX-C12:** The Opportunity Detail page SHALL include a "Documents" tab containing a file list, drag-and-drop upload zone, storage meter, and retention date indicator.
- **UX-C13:** File uploads SHALL display per-file progress bars during upload and a virus scan status indicator upon completion. Files flagged as infected SHALL be visually distinguished and have downloads disabled.
- **UX-C14:** Each file row SHALL provide actions for download, inline preview (PDF and images), share link generation, rename, category change, and deletion with confirmation.
- **UX-C15:** The storage meter SHALL use three-tier color coding (green < 70 %, amber 70–89 %, red ≥ 90 %) and display both the used/limit values and the document retention date.

---

### 3.5 Per-Tender Permissions Panel

**Host screen:** Opportunity Detail (`/opportunities/[id]`) — "Settings" tab.

**Placement:** First section within the Settings tab, above any other tender-level configuration.

**Layout:**

```
┌──────────────────────────────────────────────────────┐
│  Team Access                              [+ Invite] │
│                                                      │
│  Inheriting from company defaults.                   │
│  [Override for this tender]                           │
│                                                      │
│  ┌──────────────────────────────────────────────┐    │
│  │  👤 Maria Ivanova    Admin      Full Access  │    │
│  │  👤 Georgi Petrov    Bid Mgr    Full Access  │    │
│  │  👤 Elena Todorova   Analyst    Edit Props   │    │
│  │  👤 Ivan Dimitrov    Viewer     Review Only  │    │
│  └──────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────┘
```

**Default (inherited) state:**

- Team member list rendered from company-level roles.
- Each row: avatar (32 px circle, initials fallback), name, company role badge (`Badge variant="outline"`), effective permission level (read-only text).
- Banner text: "Inheriting from company defaults." with a link/button "Override for this tender" that switches to editable mode.

**Override state:**

- Banner changes to: "Custom permissions for this tender. [Reset to defaults]".
- Permission column becomes a `Select` dropdown per member with options:
  - **Full Access** — can edit proposals, documents, settings.
  - **Edit Proposals** — can edit proposals and upload documents, cannot change settings.
  - **Review Only** — read-only access to all tender content.
  - **No Access** — tender is hidden from this member entirely.
- Changes are auto-saved on selection (optimistic UI with toast: "Permission updated").

**Invite flow:**

- "+ Invite" button opens a popover/dialog.
- Email input field (with autocomplete from existing company members).
- Permission level selector (same four options).
- If email is not an existing company member: "This person will be invited to join your organization and will only see this tender."
- "Send Invite" button → triggers email invitation.

**Access control:** Only users with the Admin or Bid Manager company role can view and modify this panel. For other users, the Settings tab either hides this section or shows a read-only "Your access: [level]" badge.

**Data source:** `GET /api/opportunities/[id]/permissions` returns array of `{ userId, name, avatar, companyRole, tenderPermission, isOverridden }`. `PATCH /api/opportunities/[id]/permissions/[userId]` to update. `POST /api/opportunities/[id]/permissions/invite` for new invites.

**Responsive behavior:**

- ≥ 1024 px: table layout as described.
- < 1024 px: card layout per member — name and avatar top, company role and permission dropdown stacked below. Invite button becomes full-width at top.

**UX Requirements:**

- **UX-C16:** The Opportunity Detail Settings tab SHALL display a "Team Access" panel listing all team members with their company role and effective permission level for the current tender.
- **UX-C17:** A bid manager or admin SHALL be able to override company-default permissions on a per-tender basis, choosing from Full Access, Edit Proposals, Review Only, or No Access for each team member.
- **UX-C18:** The panel SHALL support inviting new members to a specific tender via email, with permission level selection at invite time. Invitees who are not existing company members SHALL receive an organization invitation scoped to the single tender.

---

### 3.6 Usage Meter Widget

**Host screens:** Global header (all pages), Subscription Settings (`/settings/subscription`)

#### 3.6.1 Header Compact Widget

**Placement:** Header bar, left of the language selector. Visible on Professional and Enterprise tiers. Hidden on Starter tier (they see only the subscription settings breakdown).

**Component:** A 28 × 28 px circular progress ring (SVG) with the most constrained meter's percentage, plus a label:

```
[◔] 8/10 summaries
```

- Ring color: green (< 70 %), amber (70–89 %), red (≥ 90 %).
- "Most constrained" = the metered feature with the highest usage-to-limit ratio.
- Text: `text-xs text-muted-foreground`, showing `used/limit featureName`.
- Click: navigates to `/settings/subscription#usage`.

**Update frequency:** Polled every 60 seconds via TanStack Query background refetch, or invalidated immediately after any AI operation completes.

#### 3.6.2 Subscription Settings — Detailed Breakdown

**Placement:** `/settings/subscription` — section titled "Current Usage" below the plan summary card.

**Layout:**

```
┌──────────────────────────────────────────────────┐
│  Current Usage                 Resets: Apr 1      │
│                                                   │
│  AI Summaries          8 / 10    ████████░░ 80%  │
│  Proposal Drafts       2 / 5     ████░░░░░░ 40%  │
│  Compliance Checks     0 / 3     ░░░░░░░░░░  0%  │
│  Document Parses      12 / 20    ██████░░░░ 60%  │
│  Team Members          3 / 5     ██████░░░░ 60%  │
│  Storage             124 / 500 MB ██░░░░░░░ 25%  │
└──────────────────────────────────────────────────┘
```

**Per meter row:**

- Feature name (`text-sm font-medium`).
- Used / limit (`text-sm tabular-nums`).
- Progress bar (`h-2 rounded-full w-48`). Color per three-tier threshold.
- Percentage (`text-xs text-muted-foreground`).

**Reset date:** Displayed once at section level — "Resets: [date]" right-aligned in the section header.

**Data source:** `GET /api/subscription/usage` returns `{ resetDate: ISO8601, meters: [{ feature: string, used: number, limit: number }] }`.

**Responsive behavior:**

- Header widget: identical at all breakpoints (always compact). On screens < 768 px, the text label is hidden; only the ring is shown. Tapping the ring navigates to subscription settings.
- Subscription detail: full layout on desktop. On mobile, progress bars stretch to full width, each meter stacks vertically as its own card row.

**UX Requirements:**

- **UX-C19:** The global header SHALL display a compact usage meter widget showing the most constrained metered feature as a colored ring with a numeric used/limit label. Clicking the widget SHALL navigate to the subscription settings page.
- **UX-C20:** The subscription settings page SHALL display a detailed usage breakdown with one row per metered feature, each showing a progress bar with three-tier color coding, numeric used/limit values, and the billing period reset date.

---

### 3.7 Self-Service Submission Guide

**Host screen:** Opportunity Detail (`/opportunities/[id]`) — triggered from the export dialog (UX-P03) and re-accessible from the Opportunity Detail action bar.

**Trigger:** After the user completes an export from the proposal export dialog, a "Continue to Submission Guide" button appears in the export success state. Alternatively, a "Submission Guide" button is always available in the Opportunity Detail action bar once at least one export exists.

**Component:** Full-screen dialog or dedicated route panel (`/opportunities/[id]/submit`).

**Wizard structure (4 steps, vertical stepper on left / horizontal on mobile):**

```
┌──────────────────────────────────────────────────────┐
│  Submission Guide for: [Opportunity Title]           │
│                                                      │
│  ● Step 1: Download documents                        │
│  ○ Step 2: Prepare attachments                       │
│  ○ Step 3: Submit on portal                          │
│  ○ Step 4: Confirm submission                        │
│                                                      │
│  ┌──────────────────────────────────────────────┐    │
│  │  Step 1: Download Your Documents              │    │
│  │                                               │    │
│  │  ✓ Proposal (PDF)         [Download]          │    │
│  │  ✓ Submission Checklist   [Download]          │    │
│  │  ○ ESPD (XML)             [Generate + DL]     │    │
│  │                                               │    │
│  │             [Back]  [Next: Prepare →]         │    │
│  └──────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────┘
```

**Step 1 — Download Documents:**

- List of all exportable/generated documents for this opportunity.
- Each row: checkbox (auto-checked if already downloaded in this session), document name, format badge (PDF/DOCX/XML), "Download" button.
- ESPD: if applicable to the tender, show with a "Generate" action if not yet created.
- Progress tracking: once downloaded, row shows green check. Session-local state (not persisted to server until Step 4).

**Step 2 — Prepare Attachments:**

- Checklist of mandatory documents extracted from the tender specification (from parsed data — Section 3.10).
- Each item: document name, requirement source ("Section 4.2 of tender spec"), status: "Ready" (file uploaded to Document Manager), "Missing" (not yet uploaded), "Not applicable" (user can mark).
- "Upload" button inline for missing items → opens the Document Manager upload flow.
- All-ready indicator: green banner "All required documents are prepared" when every item is "Ready" or "Not applicable".

**Step 3 — Submit on Portal:**

- Portal identification: "[Portal name]" with logo if available, direct URL link button "Open [portal name] →" (opens in new tab).
- Step-by-step written instructions specific to the portal type (TED eSender, national portal, etc.), stored as structured content:
  1. "Log in to [portal] with your organization credentials"
  2. "Navigate to tender reference [reference number]"
  3. "Upload the following documents in order: …"
  4. "Fill in the required form fields: …"
  5. "Review and submit before the deadline"
- Deadline reminder: prominent callout — "Deadline: [date] at [time] ([timezone]) — [X days remaining]" with color coding (green > 7 days, amber 3–7, red < 3).

**Step 4 — Confirm Submission:**

- Form fields:
  - "Date submitted" — date picker, defaults to today.
  - "Confirmation reference" — text input for portal confirmation number.
  - "Upload confirmation receipt" — optional file upload.
  - "Notes" — optional textarea.
- "Mark as Submitted" primary button.
- On confirm: opportunity status updates to "Submitted". Deadline tracking shifts to evaluation phase. Toast: "Submission recorded. We'll track the evaluation timeline for you."

**Dismissal and re-access:**

- "Close" / "I'll do this later" dismisses the wizard. State is not lost — returning to the guide resumes where the user left off (steps completed are remembered in the opportunity's local state via Zustand).
- Re-access: "Submission Guide" button in opportunity detail action bar, with a progress indicator ("Step 2/4").

**Data source:** Opportunity detail object for portal info and deadlines. Parsed document data for attachment checklist. `POST /api/opportunities/[id]/submission` to record submission confirmation.

**Responsive behavior:**

- ≥ 1024 px: two-column layout — vertical stepper on left (200 px), step content on right.
- 768–1023 px: horizontal stepper above step content.
- < 768 px: horizontal stepper collapses to "Step X of 4" text with progress bar. Step content full-width. Navigation buttons sticky at bottom.

**UX Requirements:**

- **UX-C21:** After a successful proposal export, the system SHALL present a "Continue to Submission Guide" call-to-action. The guide SHALL also be accessible from the Opportunity Detail action bar at any time after an export exists.
- **UX-C22:** The Submission Guide SHALL present a 4-step wizard: Download Documents, Prepare Attachments, Submit on Portal, and Confirm Submission. Step progress SHALL be preserved if the user dismisses and returns.
- **UX-C23:** Step 2 (Prepare Attachments) SHALL display a checklist of mandatory documents derived from parsed tender requirements, with upload actions for missing items.
- **UX-C24:** Step 4 (Confirm Submission) SHALL allow the user to record the submission date, portal confirmation reference, and optional receipt upload. Confirming SHALL update the opportunity status to "Submitted" and initiate evaluation deadline tracking.

---

### 3.8 Per-Bid Add-On Purchase Flow

**Host screens:** Any screen where a user hits a feature gate (opportunity detail, proposal editor, compliance check dialog).

**Trigger:** User attempts an action that exceeds their tier limit (e.g., 11th AI summary on a 10-summary plan). Instead of a hard block, the system presents the add-on purchase option.

**Modal (shadcn `Dialog`, `max-w-md`):**

```
┌──────────────────────────────────────────────┐
│  Purchase Add-On                    [X]      │
│                                              │
│  ┌──────────────────────────────────────┐    │
│  │  🔓 Additional AI Summary             │    │
│  │                                      │    │
│  │  Your plan includes 10 summaries/mo. │    │
│  │  Purchase 5 additional for €4.99.    │    │
│  │                                      │    │
│  │  Payment: Visa •••• 4242             │    │
│  │  [Change payment method]             │    │
│  └──────────────────────────────────────┘    │
│                                              │
│  Total: €4.99 (excl. VAT)                   │
│                                              │
│  [Cancel]        [Purchase & Continue]       │
└──────────────────────────────────────────────┘
```

**Content:**

- Feature icon + name (e.g., "Additional AI Summary").
- Explanation: "Your plan includes [limit] [feature]/mo. Purchase [qty] additional for [price]."
- Payment method: shows the default Stripe card on file. "Change payment method" link opens Stripe Customer Portal or an inline payment method selector.
- Price: displayed excluding VAT; VAT line shown if applicable ("+ VAT: €1.00").
- Total: bold, prominent.

**Actions:**

- "Cancel" → closes modal, action is not performed. User returns to the previous screen.
- "Purchase & Continue" → initiates Stripe PaymentIntent via `POST /api/billing/add-on`. Button shows loading spinner during processing. On success: modal transitions to a success state ("Purchased! Continuing…" with check icon for 1.5 seconds), then auto-closes and the originally attempted action proceeds. On failure: inline error message ("Payment failed. Please check your card details.") with retry option.

**Receipt:**

- Stripe sends email receipt automatically.
- In-app: purchase appears in `/settings/subscription` under an "Add-On Purchases" section — table with columns: date, feature, quantity, amount, receipt link (Stripe-hosted).

**Data source:** `POST /api/billing/add-on { feature, quantity }` → returns `{ success, paymentIntentId, receiptUrl }`. `GET /api/billing/add-ons` for purchase history.

**Responsive behavior:**

- ≥ 768 px: centered dialog as described.
- < 768 px: bottom sheet. "Purchase & Continue" button is full-width and sticky at the bottom.

**UX Requirements:**

- **UX-C25:** When a user attempts an action beyond their tier limit, the system SHALL present a purchase add-on modal instead of a hard block. The modal SHALL display the feature name, quantity, price, and the stored payment method.
- **UX-C26:** The "Purchase & Continue" action SHALL process payment via Stripe, display a loading state during processing, and upon success, automatically proceed with the originally attempted action without requiring the user to re-trigger it.
- **UX-C27:** All add-on purchases SHALL appear in the Subscription Settings page under an "Add-On Purchases" history section with date, feature, amount, and receipt link.

---

### 3.9 Near-Limit Usage Warnings

**Host screens:** All screens where metered features are consumed (opportunity detail, proposal editor, compliance dialogs, analytics). Also global header and subscription settings.

**Warning tiers:**

| Threshold | Visual Treatment | Scope |
|---|---|---|
| 70 % used | Counter text and progress bar turn amber (`text-amber-600`, `bg-amber-500`). | Appears on: header usage widget, subscription settings meters, and any inline usage counters near the action button. |
| 90 % used | Amber treatment continues. Additionally, a dismissable banner appears at the top of the affected feature's view. | Banner text: "[1] [feature] remaining this month. [Upgrade]". Banner style: `bg-amber-50 border-amber-200 text-amber-800` with a close (`X`) button. Dismissed state stored in sessionStorage (re-appears next session). |
| 100 % used | Action button becomes disabled. Button label changes to "Upgrade to unlock". An inline card replaces the action area. | Card: "You've used all [limit] [feature] this month. Upgrade to [next tier] for [new limit], or purchase an add-on." Two CTAs: "View plans" (→ `/settings/subscription`), "Purchase add-on" (→ add-on modal from 3.8). |

**Banner component (90 % warning):**

```
┌─────────────────────────────────────────────────────────────┐
│ ⚠ 1 AI summary remaining this month.  [Upgrade]       [X] │
└─────────────────────────────────────────────────────────────┘
```

- Positioned below the page header, above the content area.
- Full width with 16 px horizontal padding.
- "Upgrade" is a text link navigating to `/settings/subscription`.

**100 % gate card:**

```
┌──────────────────────────────────────────────┐
│  Monthly limit reached                       │
│                                              │
│  You've used all 10 AI summaries this month. │
│  Resets on April 1, 2026.                    │
│                                              │
│  [View plans]    [Purchase add-on]           │
└──────────────────────────────────────────────┘
```

- Replaces or overlays the disabled action button area.
- `Card` with `border-red-200 bg-red-50`.

**Data source:** Same as usage meter widget (Section 3.6). Warning thresholds calculated client-side from usage data.

**Responsive behavior:**

- Banners: full-width at all breakpoints.
- Gate card: full-width on mobile, max-width 400 px centered on desktop.
- Button label change ("Upgrade to unlock"): identical across breakpoints.

**UX Requirements:**

- **UX-C28:** At 70 % usage of any metered feature, the corresponding progress bars and counter text SHALL transition from green to amber across all locations where they appear.
- **UX-C29:** At 90 % usage, a dismissable warning banner SHALL appear at the top of the affected feature's interface, stating the remaining count and linking to the upgrade page. Dismissal SHALL persist for the current session only.
- **UX-C30:** At 100 % usage, the feature's action button SHALL be disabled and replaced with an "Upgrade to unlock" label. An inline card SHALL present options to view upgrade plans or purchase an add-on, along with the reset date.

---

### 3.10 Parsed Document Results on Opportunity Detail

**Host screen:** Opportunity Detail (`/opportunities/[id]`) — enriches the existing Overview tab and adds new sub-sections/tabs.

**Trigger:** When the AI document parser (WF2) completes processing the uploaded tender documents, the Opportunity Detail page automatically enriches with structured data. The page listens for parser completion via a TanStack Query subscription to the opportunity object (field `parsingStatus` transitions from `processing` to `complete`).

**Processing state indicator:**

While parsing is in progress, a banner appears below the tab bar:

```
┌──────────────────────────────────────────────────────┐
│ ⚙ AI is analyzing tender documents… (2 of 4 files)  │
│   [████████░░░░░░░░] 50%                             │
└──────────────────────────────────────────────────────┘
```

- Animated progress bar. File count indicator.
- On completion: banner transitions to "Analysis complete — [X] sections extracted" (green, auto-dismisses after 5 seconds).

**"AI Parsed" badge:**

- Small badge (`Badge variant="secondary" className="text-[10px]"`) reading "AI Parsed" shown next to section headings that contain auto-extracted data.
- Distinguishes AI-extracted content from manually entered data.
- Tooltip on hover: "This section was automatically extracted from tender documents by AI. Verify for accuracy."

**Enriched Overview section:**

The existing overview area gains new data fields (inserted below any existing manually entered fields):

| Field | Display |
|---|---|
| Contracting Authority | Name, address, contact person, contact email — rendered as a mini-card with copy-to-clipboard on email. |
| Budget | Formatted currency value with "estimated" qualifier if applicable. `text-lg font-semibold`. |
| Duration | "24 months" or "Start: [date] — End: [date]". |
| Key Dates | Inline mini-table: publication date, clarification deadline, submission deadline, estimated evaluation, estimated award. Each with countdown badge ("in 14 days"). |

**New "Evaluation Criteria" section:**

Appears as a content section within the Overview tab (or as a dedicated sub-tab if the page uses sub-navigation).

| Criterion | Type | Max Points | Weight |
|---|---|---|---|
| Technical merit | Quality | 60 | 60% |
| Proposed methodology | Quality | 20 | 20% |
| Price | Price | 20 | 20% |

- Table component (shadcn `Table`).
- "Type" column uses badges: "Quality" (blue), "Price" (green), "Pass/Fail" (gray).
- Footer row: totals (should sum to 100 points / 100 %).
- If no criteria extracted: "No evaluation criteria could be extracted. [Add manually]" empty state.

**New "Required Documents" section:**

A checklist integrated with the Document Manager (Section 3.4):

```
☐  Company registration certificate      [Upload]
☐  Financial statements (last 3 years)   [Upload]
☑  Technical proposal                     ✓ Uploaded
☐  ESPD                                   [Generate]
☑  Pricing schedule                       ✓ Uploaded
```

- Each item: checkbox (auto-checked if matching document exists in Document Manager by category/name heuristic), document name, action button.
- "Upload" navigates to or opens the Document Manager with the category pre-selected.
- "Generate" triggers document generation flow (e.g., ESPD generation).
- Summary line: "3 of 5 required documents ready".

**New "Timeline" section:**

Visual timeline of the procurement lifecycle:

```
●────────●────────●────────○────────○────────○
Published  Questions  Submission  Evaluation  Award  Contract
Feb 15     Mar 1      Apr 15      May 2026    Jun    Jul
           PAST       UPCOMING    PREDICTED
```

- Horizontal timeline (desktop), vertical timeline (mobile).
- Implemented with custom SVG or a simple div-based component (not Recharts — this is a lifecycle diagram, not a data chart).
- Node states: past (filled circle, solid line), upcoming/confirmed (filled circle, dashed line to next), predicted (empty circle, dotted line). Predicted dates appended with "est." label.
- Current position indicated by a vertical "TODAY" marker.
- Hover on any node: tooltip with exact date, time, and source ("Extracted from tender spec" or "Predicted based on typical timeline").

**Data source:** `GET /api/opportunities/[id]` returns enriched fields when `parsingStatus === "complete"`: `parsedData: { budget, duration, contractingAuthority, keyDates, evaluationCriteria[], requiredDocuments[], timelineEvents[] }`. Each parsed field includes `source: "ai_parsed" | "manual"` metadata.

**Responsive behavior:**

- ≥ 1024 px: all sections visible in scrollable single-column layout within the Overview tab. Timeline horizontal.
- 768–1023 px: identical layout, timeline may compress node spacing.
- < 768 px: sections stack vertically with collapsible headers (accordion). Timeline rotates to vertical. Evaluation criteria table becomes a card list. Required documents checklist remains a list (naturally mobile-friendly).

**UX Requirements:**

- **UX-C31:** While the AI document parser is processing, the Opportunity Detail page SHALL display a progress banner showing the file count and completion percentage. The banner SHALL auto-dismiss upon completion with a brief success message.
- **UX-C32:** All sections populated by AI-extracted data SHALL display an "AI Parsed" badge next to the section heading. The badge SHALL include a tooltip advising the user to verify accuracy.
- **UX-C33:** Upon parser completion, the Overview section SHALL be enriched with contracting authority details, budget, duration, and key dates extracted from the tender documents.
- **UX-C34:** A "Required Documents" checklist SHALL be generated from parsed tender requirements, with each item's status automatically reflecting whether a matching document exists in the Document Manager. Items missing documents SHALL offer inline upload or generation actions.
- **UX-C35:** A visual "Timeline" section SHALL display the procurement lifecycle with past, upcoming, and predicted dates distinguished by visual styling. A "TODAY" marker SHALL indicate the current position in the lifecycle.

---

### 3.11 Real-Time Collaborative Proposal Editing

**Host screen:** Proposal Editor (`/opportunities/[id]/proposals/[id]`)

**Purpose:** Enable multiple team members to work on the same proposal simultaneously with presence awareness, conflict prevention, and change attribution — supporting the PRD's FR18 (multi-user editing with RBAC).

#### Collaboration Architecture

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Sync model | **Operational Transformation (OT)** via Tiptap's built-in Collaboration extension (Hocuspocus/y-websocket backend) | Tiptap natively supports Y.js CRDT via its Collaboration extension. Avoids building custom conflict resolution. |
| Transport | **WebSocket** connection per open proposal | Low-latency bidirectional sync for cursor positions and text operations |
| Persistence | Periodic Y.js document snapshots to PostgreSQL (every 30s + on disconnect) | Survives server restarts; WebSocket is ephemeral |
| Max concurrent editors | 10 per proposal (aligned with Enterprise team size) | Prevents performance degradation |

#### Presence Indicators

| Component | Details |
|-----------|---------|
| Active user avatars | Top-right of editor toolbar: row of avatar circles (max 5 visible + "+N" overflow). Each avatar has a colored ring matching their cursor color. Tooltip on hover: name + role. |
| Cursor labels | Each remote user's cursor is a colored vertical bar with a name label floating above it (`text-xs`, colored background matching their assigned color). Colors auto-assigned from a palette of 8 distinct colors. |
| Selection highlights | When a remote user selects text, the selection is highlighted in their cursor color at 20% opacity. |
| Typing indicator | When a remote user is actively typing, their avatar in the toolbar row pulses gently. |
| Idle/away detection | After 5 minutes of no activity, cursor dims and avatar shows "Away" badge. After 15 minutes, cursor is removed (user still connected but not shown as active). |

#### Section-Level Locking

| Component | Details |
|-----------|---------|
| Lock model | **Soft advisory locks** — not hard blocks. When a user is actively editing a section (h2-level), that section shows a colored left border matching their cursor color + "Elena is editing" label. |
| Concurrent editing | Two users CAN edit the same section simultaneously (Y.js handles merge). The advisory lock is a courtesy signal, not a gate. |
| Role-based section access | Bid Manager: all sections. Technical Writer: technical sections. Financial Analyst: pricing/budget sections. Legal Reviewer: terms/compliance sections. Contributors can only edit sections assigned to them. |
| Section assignment | Bid Manager can assign sections to team members via a "Section Assignments" dropdown in each section's header toolbar. Assigned sections show the assignee's avatar. |

#### Change Attribution

| Component | Details |
|-----------|---------|
| Inline attribution | Text added by each user is invisibly attributed (stored in Y.js metadata). Visible on demand via "Show Authors" toggle in editor toolbar — each author's text gets a subtle colored underline. |
| Version history integration | Existing UX-P08 version history enhanced: each version snapshot annotated with contributing authors and their change summary. Diff view shows per-author color coding. |
| Activity feed | Collapsible panel at bottom of editor: "Elena added 3 paragraphs to Technical Approach (2 min ago)" / "Georgi edited Pricing Table (5 min ago)". Last 20 activities. |

#### Conflict & Connection Handling

| Scenario | Behavior |
|----------|----------|
| Network disconnect | Local edits continue in offline buffer. Reconnect auto-syncs via Y.js merge. Toast: "Connection lost — your changes are saved locally" → "Reconnected — changes synced". |
| Conflicting edits | Y.js CRDT handles character-level merge automatically. No manual conflict resolution needed for text. For structured data (metadata fields, status changes): last-write-wins with "X changed Y — your version was overwritten" toast. |
| Editor limit reached | 11th user sees read-only view + banner: "10 editors are active. You can view the proposal or wait for a slot." Real-time count updates. |
| Force-save | Any editor can click "Save Version" to create an explicit version snapshot (beyond auto-save). This is useful before major changes. |

#### Responsive Behavior

| Breakpoint | Adaptation |
|-----------|-----------|
| Desktop (1024+) | Full collaboration UI — avatar row, cursor labels, section lock indicators, activity feed panel. |
| Tablet (768–1024) | Avatar row visible. Cursor labels shortened to initials only. Activity feed hidden (accessible via button). |
| Mobile (<768) | Collaboration simplified: avatar row shows active user count only ("3 editing"). No remote cursors shown (screen too small). Activity feed accessible via Sheet. Edits still sync in real-time. |

#### Data Sources

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `wss://eusolicit.com/api/v1/proposals/{id}/collab` | WebSocket | Y.js document sync + cursor positions |
| `/api/v1/proposals/{id}/presence` | GET | Current active editors list (fallback if WebSocket unavailable) |
| `/api/v1/proposals/{id}/versions` | POST | Create explicit version snapshot |
| `/api/v1/proposals/{id}/sections/assignments` | GET/PUT | Section-to-user assignments |

#### UX Requirements

| ID | Requirement |
|----|-------------|
| **UX-CO01** | Active collaborator avatars must be visible in the editor toolbar at all times, each with a unique color ring matching their cursor color. Hovering an avatar must show the user's name and role. |
| **UX-CO02** | Remote cursors must display as colored vertical bars with a floating name label. Cursor colors must be distinct and auto-assigned from a palette of 8 colors. Cursors of idle users (>5 min) must dim; cursors of away users (>15 min) must be removed. |
| **UX-CO03** | Section-level advisory locks must show a colored left border and "[Name] is editing" indicator when a user is actively typing in a section. Advisory locks must not prevent other users from editing the same section. |
| **UX-CO04** | Role-based section access must restrict editing to assigned sections per team role. Unauthorized edit attempts must show a tooltip: "This section is assigned to [Role]. Contact your Bid Manager for access." |
| **UX-CO05** | Network disconnection must be handled gracefully: local edits buffered, reconnection auto-syncs, and a toast must inform the user of connection status transitions. No edits must be lost. |
| **UX-CO06** | On mobile (<768px), collaboration must be simplified to an active editor count and real-time sync without remote cursors. Full collaboration features must be available on tablet and desktop. |
| **UX-CO07** | An activity feed must show the last 20 edit actions with author, section, summary, and timestamp. On desktop it must be a collapsible bottom panel; on mobile, accessible via a Sheet. |

---

## SECTION 4: Analytics Dashboard Breakdown

The analytics hub is a tabbed interface at `/analytics` providing market intelligence, ROI tracking, team performance, competitor insights, pipeline forecasting, usage analytics, and custom reporting. Each tab is a distinct view with dedicated data sources, chart components, and access controls.

---

### 4.1 Analytics Shell

**Route:** `/analytics` (redirects to the first accessible tab for the user's tier)

**Layout:**

```
┌──────────────────────────────────────────────────────────────┐
│  Analytics                                                    │
│                                                               │
│  [Market Intel] [ROI] [Team] [Competitor] [Pipeline] [Usage] [Reports] │
│                                                               │
│  Date range: [This quarter ▾]  [Jan 1 – Mar 31]   [Export ▾] │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐     │
│  │                                                      │     │
│  │              Active tab content area                  │     │
│  │                                                      │     │
│  └──────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────┘
```

**Tab bar:**

- Horizontal tab bar using shadcn `Tabs` component.
- Each tab: text label, optional icon (16 px Lucide icon left of label).
- Active tab: `border-b-2 border-primary text-foreground`. Inactive: `text-muted-foreground`.
- Tier-locked tabs: visible but grayed out with a lock icon. Clicking shows a tooltip: "Available on [tier] plan. [Upgrade]".

**Tier access matrix:**

| Tab | Starter | Professional | Enterprise |
|---|---|---|---|
| Market Intelligence | — | ✓ | ✓ |
| ROI Tracker | — | ✓ | ✓ |
| Team Performance | — | ✓ | ✓ |
| Competitor Intelligence | — | — | ✓ |
| Pipeline Forecast | — | ✓ | ✓ |
| Usage Analytics | ✓ | ✓ | ✓ |
| Custom Reports | — | ✓ | ✓ |

**Global controls bar:**

- Date range picker: shadcn `Popover` with a calendar component. Presets dropdown: "This month", "This quarter", "This year", "Last 12 months", "Custom". Custom opens dual calendar for start/end date selection. Selected range displayed as formatted text.
- Export button: `DropdownMenu` — "Export as PDF", "Export as CSV", "Export as PNG" (chart image). Export scope = the currently active tab's data.

**State management:** Active tab stored in URL search params (`/analytics?tab=roi`) for shareability. Date range stored in Zustand analytics store, shared across all tabs. Tab content lazy-loaded using React.lazy or Next.js dynamic imports.

**Responsive behavior:**

- ≥ 1280 px: all tabs visible in horizontal bar.
- 1024–1279 px: tabs scroll horizontally with fade indicators on the edges.
- < 1024 px: tab bar collapses into a `Select` dropdown showing the active tab name. Date range picker and export button stack below.
- < 768 px: global controls (date range, export) move into a collapsible "Filters" section accessed via a button.

**UX Requirements:**

- **UX-AN01:** The analytics hub SHALL be accessible at `/analytics` and present a horizontal tabbed interface with tabs for Market Intelligence, ROI Tracker, Team Performance, Competitor Intelligence, Pipeline Forecast, Usage Analytics, and Custom Reports.
- **UX-AN02:** Tabs not available for the user's subscription tier SHALL be visible but disabled, displaying a lock icon and an upgrade prompt on interaction.
- **UX-AN03:** A global date range picker and export control SHALL persist across all tabs. The selected date range SHALL apply to all tab content. The active tab SHALL be encoded in the URL for shareability.

---

### 4.2 Market Intelligence View

**Route:** `/analytics?tab=market`

**Purpose:** Provide sector-level analytics on EU procurement volumes to help users identify market opportunities and trends.

**Layout (desktop):**

```
┌──────────────────────────────────────────────────────────┐
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │
│  │ 1,247   │ │ €842M   │ │ €675K   │ │ 89      │       │
│  │ Total    │ │ Total   │ │ Avg     │ │ New this │       │
│  │ tracked  │ │ value   │ │ value   │ │ month    │       │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘       │
│                                                          │
│  Filters: [Region ▾] [CPV Sector ▾] [Budget ▾]          │
│                                                          │
│  ┌──────────────────────┐ ┌────────────────────────┐     │
│  │ Tender Volume by     │ │ Monthly Volume Trends   │     │
│  │ Sector (bar)         │ │ (line)                  │     │
│  │                      │ │                         │     │
│  │ ████████ IT          │ │     /\    /\            │     │
│  │ ██████ Construction  │ │    /  \  /  \           │     │
│  │ █████ Healthcare     │ │   /    \/    \          │     │
│  │ ████ Transport       │ │  /            \         │     │
│  │ ...                  │ │ /              \        │     │
│  └──────────────────────┘ └────────────────────────┘     │
│                                                          │
│  ┌──────────────────────┐ ┌────────────────────────┐     │
│  │ Value Distribution   │ │ Most Active Authorities │     │
│  │ by Sector (treemap)  │ │ (table)                 │     │
│  │                      │ │                         │     │
│  │ ┌──────┐┌───┐┌──┐   │ │ Name     Count  Avg Val │     │
│  │ │ IT   ││Con││HC│   │ │ EC DG…   42     €1.2M   │     │
│  │ │      ││   │└──┘   │ │ ECHA     38     €890K   │     │
│  │ └──────┘└───┘       │ │ EIB      27     €2.1M   │     │
│  └──────────────────────┘ └────────────────────────┘     │
└──────────────────────────────────────────────────────────┘
```

**KPI cards (top row):**

Four cards in a horizontal row using shadcn `Card`:

| Card | Value format | Icon (Lucide) | Subtext |
|---|---|---|---|
| Total opportunities tracked | Integer with comma separator | `Search` | "+[X]% vs previous period" (green/red delta) |
| Total estimated value | Currency (€) with magnitude suffix (K/M/B) | `Euro` | "+[X]% vs previous period" |
| Average contract value | Currency (€) with magnitude suffix | `TrendingUp` | "+[X]% vs previous period" |
| New this month | Integer | `PlusCircle` | Absolute count, no comparison |

**Charts:**

1. **Tender Volume by CPV Sector (horizontal bar chart):**
   - Recharts: `<BarChart layout="vertical">` with `<Bar>`, `<XAxis type="number">`, `<YAxis type="category" dataKey="sector">`.
   - Top 10 sectors by volume. Bar color: primary palette with decreasing opacity.
   - Tooltip: sector name, count, percentage of total.
   - Click bar: filters the entire view to that sector.

2. **Monthly Volume Trends (line chart):**
   - Recharts: `<LineChart>` with `<Line type="monotone">`.
   - X-axis: months. Y-axis: tender count.
   - Two lines if comparison enabled: current period (solid, primary color) vs previous period (dashed, muted color).
   - Tooltip: month, count, delta vs previous.
   - `<CartesianGrid strokeDasharray="3 3" />` for readability.

3. **Contract Value Distribution by Sector (treemap):**
   - Recharts: `<Treemap>` with data proportional to total contract value per sector.
   - Each cell: sector name + total value. Color intensity proportional to average contract value.
   - Tooltip: sector, total value, count, average value.
   - Click cell: filters to that sector (same as bar chart click).

4. **Most Active Contracting Authorities (data table):**
   - shadcn `Table` with sortable columns.
   - Columns: rank (#), authority name, tender count, average value, total value.
   - Default sort: by tender count descending.
   - Rows: top 20. "Show all" expands or paginates.
   - Click row: navigates to a filtered opportunity list for that authority.

**Filter bar:**

- Region: multi-select dropdown (EU member states + EU-wide).
- CPV Sector: searchable multi-select (CPV code + description).
- Budget range: dual slider or preset ranges ("< €100K", "€100K–€1M", "€1M–€10M", "> €10M").
- Filters are additive (AND). Active filters shown as removable pills below the filter bar.

**Data source:** `GET /api/analytics/market?dateFrom=&dateTo=&region=&sector=&budgetMin=&budgetMax=` returns `{ kpis: {...}, sectorVolumes: [...], monthlyTrends: [...], valueDistribution: [...], topAuthorities: [...] }`.

**Responsive behavior:**

- ≥ 1280 px: 2×2 grid of charts as shown. KPI cards in a 4-column row.
- 1024–1279 px: KPI cards in 4-column row. Charts in 2-column grid, but treemap and table stack if space is tight.
- 768–1023 px: KPI cards in 2×2 grid. Charts stack vertically, each full-width.
- < 768 px: KPI cards in a horizontal scrollable row (snap scroll). Charts stack vertically, full-width. Table switches to a card list format. Filter bar collapses behind a "Filters" toggle button.

**UX Requirements:**

- **UX-AN04:** The Market Intelligence view SHALL display four KPI summary cards showing total opportunities tracked, total estimated value, average contract value, and new opportunities this month, each with a delta comparison to the previous period.
- **UX-AN05:** The view SHALL include a horizontal bar chart of tender volume by CPV sector (top 10), a monthly trend line chart, a treemap of contract value distribution by sector, and a sortable table of the most active contracting authorities.
- **UX-AN06:** Clicking a sector bar or treemap cell SHALL filter the entire view to that sector. Clicking a contracting authority row SHALL navigate to a filtered opportunity list for that authority.
- **UX-AN07:** The filter bar SHALL support multi-select for region and CPV sector, and a budget range selector. Active filters SHALL be displayed as removable pills.

---

### 4.3 ROI Tracker View

**Route:** `/analytics?tab=roi`

**Purpose:** Track preparation costs against contract values to measure return on investment per bid and in aggregate.

**Layout (desktop):**

```
┌──────────────────────────────────────────────────────────┐
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │
│  │ €47,200 │ │ €1.8M   │ │ 38:1    │ │ €2,360  │       │
│  │ Total    │ │ Total   │ │ Overall │ │ Avg cost│       │
│  │ invested │ │ won     │ │ ROI     │ │ per bid │       │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘       │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │  Bid ROI Table                                    │    │
│  │  Opportunity   Cost    Value    ROI    Status     │    │
│  │  IT Platform   €3.2K   €280K   87:1   Won ✓      │    │
│  │  HVAC Maint.   €1.8K   —       —      Pending    │    │
│  │  Legal Svcs    €4.1K   €0      -100%  Lost ✗     │    │
│  │  ...                                    [Log $]   │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  ┌────────────────────────┐ ┌──────────────────────┐     │
│  │ ROI by Sector (bar)    │ │ ROI Trend (line)     │     │
│  └────────────────────────┘ └──────────────────────┘     │
└──────────────────────────────────────────────────────────┘
```

**KPI cards:**

| Card | Calculation | Format |
|---|---|---|
| Total invested | Sum of all preparation costs (logged + auto-tracked) | Currency (€) |
| Total won | Sum of contract values for bids with "Won" status | Currency (€) |
| Overall ROI | (Total won − Total invested) / Total invested | Ratio (X:1) or percentage |
| Avg cost per bid | Total invested / number of bids | Currency (€) |

Each card shows a delta vs previous period with directional arrow and green/red coloring.

**Bid ROI table:**

shadcn `Table` with the following columns:

| Column | Content | Width |
|---|---|---|
| Opportunity | Name (linked to opportunity detail), sector badge | flex |
| Preparation cost | Currency. Editable inline (click to edit). | 120 px |
| Contract value | Currency if won; "—" if pending/lost. | 120 px |
| ROI | Calculated ratio if won; "—" if no value. | 80 px |
| Status | Badge: "Won" (green), "Lost" (red), "Submitted" (blue), "In Progress" (amber) | 100 px |
| Actions | "Log cost" button (opens cost entry dialog) | 80 px |

**Cost logging dialog (shadcn `Dialog`, `max-w-sm`):**

```
┌──────────────────────────────────────┐
│  Log Preparation Cost         [X]   │
│                                      │
│  Category         Amount             │
│  ┌──────────────┐ ┌──────────┐       │
│  │ Team hours ▾ │ │ €1,200   │       │
│  └──────────────┘ └──────────┘       │
│                                      │
│  [+ Add line item]                   │
│                                      │
│  AI usage fees (auto):   €48.00      │
│  ─────────────────────────────       │
│  Total:                  €1,248.00   │
│                                      │
│  [Cancel]              [Save]        │
└──────────────────────────────────────┘
```

- Categories: "Team hours", "External consultants", "Translation", "Travel", "Other".
- AI usage fees: auto-calculated from the system's tracking of AI operations consumed for this tender.
- Multiple line items supported.
- "Save" persists and updates the table immediately.

**Charts:**

1. **ROI by Sector (bar chart):**
   - Recharts: `<BarChart>` with sectors on X-axis, ROI ratio on Y-axis.
   - Color: bars with positive ROI in green, negative in red.
   - Only includes sectors with won bids (sufficient data).

2. **ROI Trend Over Time (line chart):**
   - Recharts: `<LineChart>` — X-axis: months/quarters. Y-axis: cumulative ROI.
   - Two lines: investment (area fill, muted) and returns (area fill, primary).
   - Tooltip: period, invested amount, won amount, ROI for that period.

**Tooltip (info icon next to "Preparation cost" header):** "Preparation cost includes AI usage fees (tracked automatically), team hours, and any external costs you've logged manually."

**Data source:** `GET /api/analytics/roi?dateFrom=&dateTo=` returns `{ kpis: {...}, bids: [...], sectorRoi: [...], trendData: [...] }`. Cost entries: `POST /api/opportunities/[id]/costs`, `GET /api/opportunities/[id]/costs`.

**Responsive behavior:**

- ≥ 1024 px: KPI cards in row, table full-width, charts side by side below.
- 768–1023 px: KPI cards 2×2, table horizontal scrollable, charts stacked.
- < 768 px: KPI cards horizontal scrollable. Table collapses to card view — each bid as a card showing opportunity name, cost, value, ROI, status. Charts stacked full-width. Cost logging dialog becomes a bottom sheet.

**UX Requirements:**

- **UX-AN08:** The ROI Tracker SHALL display aggregate KPI cards for total investment, total value won, overall ROI ratio, and average cost per bid.
- **UX-AN09:** A bid-level table SHALL list each opportunity with preparation cost, contract value (if won), calculated ROI, and outcome status. Each row SHALL provide a "Log cost" action opening a categorized cost entry dialog.
- **UX-AN10:** AI usage fees SHALL be automatically tracked and included in the preparation cost calculation, displayed as a separate auto-calculated line item in the cost logging dialog.
- **UX-AN11:** The view SHALL include a bar chart comparing ROI by sector and a line chart showing ROI trend over time with investment and return area fills.

---

### 4.4 Team Performance View

**Route:** `/analytics?tab=team`

**Access:** Visible only to users with Admin or Bid Manager roles. Other roles see: "Team performance data is available to bid managers and administrators."

**Purpose:** Provide per-team-member performance metrics to help bid managers allocate resources and identify coaching opportunities.

**Layout (desktop):**

```
┌──────────────────────────────────────────────────────────┐
│  Team Performance                                        │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │  Member     Assigned  Submitted  Win Rate  Avg    │    │
│  │                                            Time   │    │
│  │  M. Ivanova   12        10        40%      5.2d   │    │
│  │  G. Petrov     8         7        28%      7.1d   │    │
│  │  E. Todorova   6         6        50%      4.0d   │    │
│  │  I. Dimitrov   4         3        33%      6.5d   │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  ┌────────────────────────┐ ┌──────────────────────┐     │
│  │ Win Rate Comparison    │ │ Preparation Time     │     │
│  │ (bar)                  │ │ Trend (line)         │     │
│  └────────────────────────┘ └──────────────────────┘     │
└──────────────────────────────────────────────────────────┘
```

**Performance table:**

shadcn `Table` with sortable columns:

| Column | Content | Notes |
|---|---|---|
| Team member | Avatar (24 px) + name | Click → drill-down to individual view |
| Bids assigned | Integer | Total opportunities assigned to this member |
| Bids submitted | Integer | Opportunities where a proposal was submitted |
| Win rate | Percentage | Won / (Won + Lost). "—" if no outcomes yet |
| Avg preparation time | Days (1 decimal) | Average time from assignment to submission |
| Avg quality score | Numeric (0–100) | Average AI compliance/quality score across proposals |

**Charts:**

1. **Win Rate Comparison (bar chart):**
   - Recharts: `<BarChart>` — X-axis: team member names. Y-axis: win rate percentage.
   - Bar color: gradient from amber (low) to green (high).
   - Reference line at team average.

2. **Preparation Time Trend (line chart):**
   - Recharts: `<LineChart>` — X-axis: months. Y-axis: average preparation days.
   - One line per team member (different colors), with a team average dashed line.
   - Legend: shows/hides individual members.

**Drill-down (click team member name):**

- Navigates to `/analytics?tab=team&member=[userId]` or opens a slide-over panel.
- Shows: individual bid history table (opportunity name, dates, status, score, time spent), personal win rate trend over time (line chart), sector breakdown of their bids (donut chart).
- "Back to team view" breadcrumb.

**Data source:** `GET /api/analytics/team?dateFrom=&dateTo=` returns `{ members: [{ userId, name, avatar, assigned, submitted, won, lost, avgPrepTime, avgScore }] }`. `GET /api/analytics/team/[userId]` for drill-down.

**Responsive behavior:**

- ≥ 1024 px: table full-width, charts side by side.
- 768–1023 px: table scrollable horizontally. Charts stacked.
- < 768 px: table replaced with member cards. Each card: avatar, name, key metrics in a 2×2 mini-grid. Tap card → drill-down. Charts stacked, legends truncated to 3 members + "more".

**UX Requirements:**

- **UX-AN12:** The Team Performance view SHALL be accessible only to Admin and Bid Manager roles. Unauthorized users SHALL see a descriptive access-restriction message.
- **UX-AN13:** A sortable team member table SHALL display bids assigned, bids submitted, win rate, average preparation time, and average quality score per member.
- **UX-AN14:** Clicking a team member's name SHALL open a drill-down view showing their individual bid history, personal win rate trend, and sector breakdown.
- **UX-AN15:** The view SHALL include a bar chart comparing win rates across team members (with a team average reference line) and a multi-line chart showing preparation time trends by member over time.

---

### 4.5 Competitor Intelligence View

**Route:** `/analytics?tab=competitor`

**Access:** Enterprise tier only. Professional sees the locked tab with upgrade prompt.

**Purpose:** Aggregate publicly available award data to provide insights into competitor bidding patterns, win rates, and pricing.

**Layout (desktop):**

```
┌──────────────────────────────────────────────────────────┐
│  Competitor Intelligence                                  │
│                                                          │
│  Search: [Search competitor by name...        ] [Search] │
│                                                          │
│  Watchlist (3)                                           │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                 │
│  │ Acme Ltd │ │ Beta Co  │ │ Gamma SA │  [+ Add]        │
│  │ IT Svcs  │ │ Constr.  │ │ Consult. │                 │
│  │ WR: 32%  │ │ WR: 28%  │ │ WR: 41%  │                 │
│  └──────────┘ └──────────┘ └──────────┘                 │
│                                                          │
│  ── Selected: Acme Ltd ──────────────────────────        │
│                                                          │
│  ┌──────────────────────┐ ┌────────────────────────┐     │
│  │ Win Rate Over Time   │ │ Pricing Comparison     │     │
│  │ (line)               │ │ (bar)                  │     │
│  └──────────────────────┘ └────────────────────────┘     │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │ Tenders Where You Both Competed                   │    │
│  │ Opportunity     Your Bid   Their Bid   Winner     │    │
│  │ IT Platform     €240K      €220K       Them ✗     │    │
│  │ Data Center     €1.1M      €1.3M       You ✓      │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**Search:**

- Full-width search input with debounced query (300ms).
- Results dropdown: company name, sector, country. Click selects and loads competitor profile.
- "No results" state: "No competitor data found for '[query]'. Competitor data is derived from public award notices."

**Watchlist:**

- Horizontal row of compact competitor cards (shadcn `Card`, 160 × 100 px).
- Each card: company name (`text-sm font-semibold`), primary sector badge, win rate.
- "+ Add" button: opens the search to add a new competitor to the watchlist.
- Remove: hover reveals an `X` button in the card corner.
- Max watchlist size: 10 competitors. "Remove a competitor to add more" message at limit.

**Competitor profile (selected competitor):**

Displayed below the watchlist when a competitor is selected.

- Header: company name, country, primary sector, total bids tracked, overall win rate.

**Charts:**

1. **Win Rate Over Time (line chart):**
   - Recharts: `<LineChart>` — X-axis: quarters. Y-axis: win rate %.
   - Two lines: competitor (red/orange) and your company (primary blue). Enables direct visual comparison.
   - Tooltip: quarter, competitor win rate, your win rate.

2. **Pricing Comparison (grouped bar chart):**
   - Recharts: `<BarChart>` with grouped bars per tender.
   - X-axis: tender names (abbreviated). Two bars per group: your bid (blue), their bid (red/orange).
   - Only shows tenders where both companies bid and pricing data is publicly available.
   - Tooltip: tender name, your price, their price, winner.

**Overlap analysis table:**

- Columns: opportunity name (linked), your bid amount, their bid amount, winner ("You" green badge / "Them" red badge / "Other" gray badge), date.
- Sortable by any column.
- "No overlap found" empty state if no co-competed tenders exist.

**Data source:** `GET /api/analytics/competitors/search?q=` for search. `GET /api/analytics/competitors/[competitorId]` for profile and charts. `GET /api/analytics/competitors/watchlist` for watchlist management. All data derived from TED award notices and other public procurement databases.

**Data freshness notice:** "Data sourced from public award notices. Last updated: [date]. Historical data may be incomplete." — displayed as a muted footnote below the competitor profile.

**Responsive behavior:**

- ≥ 1024 px: full layout as described.
- 768–1023 px: watchlist wraps to 2 rows if needed. Charts stacked.
- < 768 px: watchlist becomes a horizontal scrollable row. Search input full-width. Charts stacked. Overlap table switches to card view. Competitor profile header stacks vertically.

**UX Requirements:**

- **UX-AN16:** The Competitor Intelligence view SHALL provide a search function to find competitors by name, returning results from publicly available award data.
- **UX-AN17:** Users SHALL be able to maintain a watchlist of up to 10 competitors, displayed as summary cards showing company name, sector, and win rate.
- **UX-AN18:** Selecting a competitor SHALL display a win rate comparison line chart (competitor vs. user's company), a pricing comparison bar chart for co-competed tenders, and an overlap analysis table listing tenders where both companies bid.
- **UX-AN19:** A data freshness notice SHALL be displayed indicating the source and last update date of competitor data.

---

### 4.6 Pipeline Forecasting View

**Route:** `/analytics?tab=pipeline`

**Purpose:** Predict upcoming tender publications based on historical procurement cycles and display them alongside confirmed upcoming opportunities.

**Layout (desktop):**

```
┌──────────────────────────────────────────────────────────┐
│  Pipeline Forecast                                        │
│                                                          │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐                    │
│  │ 23      │ │ €14.2M  │ │ 8       │                    │
│  │ Predicted│ │ Est.    │ │ High    │                    │
│  │ next mo. │ │ value   │ │ confid. │                    │
│  └─────────┘ └─────────┘ └─────────┘                    │
│                                                          │
│  View: [Timeline ▾]  Filter: [Sector ▾] [Authority ▾]   │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │ Apr         May         Jun         Jul          │    │
│  │ ██ IT Plat  ╌╌ HVAC     ╌╌ Legal                │    │
│  │ ██ Data Ctr ██ Security ╌╌ Transport             │    │
│  │ ╌╌ Consult.              ╌╌ Cloud                │    │
│  │                                                  │    │
│  │ ██ = Confirmed   ╌╌ = Predicted (75%)            │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │ Predicted vs Actual Volume (bar chart)            │    │
│  └──────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

**KPI cards:**

| Card | Content |
|---|---|
| Predicted next month | Count of predicted opportunities with publication date in the next 30 days |
| Estimated total value | Sum of estimated values for all predicted opportunities in selected timeframe |
| High confidence | Count of predictions with confidence ≥ 75 % |

**View toggle:**

- "Timeline" (default): Gantt-style horizontal timeline.
- "Calendar": calendar month view with opportunity dots on predicted dates.
- "List": traditional table sorted by predicted publication date.

**Timeline component:**

- Custom implementation (not Recharts — this is a scheduling/Gantt visualization).
- Horizontal axis: months (scrollable for 3/6/12 month range based on global date picker).
- Each row: one opportunity. Row height: 32 px.
- Confirmed opportunities: solid bar, full opacity, `bg-primary`.
- Predicted opportunities: dashed border, reduced opacity (50 %), `bg-primary/50 border-dashed`. Confidence percentage badge at the end of the bar.
- Hover on any bar: tooltip with opportunity title, contracting authority, estimated value, predicted/confirmed publication date, confidence level (for predicted).
- Click: navigates to opportunity detail (for confirmed) or shows a prediction detail popover (for predicted).

**Prediction detail popover:**

```
┌────────────────────────────────────────┐
│ Predicted: HVAC Maintenance Contract   │
│ Authority: Municipality of Sofia       │
│ Est. value: €450K                      │
│ Predicted publication: May 2026        │
│ Confidence: 72%                        │
│ Basis: Published annually since 2022   │
│                                        │
│ [Set alert for publication]            │
└────────────────────────────────────────┘
```

"Set alert for publication" creates a monitoring alert that notifies the user when a matching tender is published.

**Predicted vs Actual chart:**

- Recharts: `<BarChart>` with grouped bars.
- X-axis: months (past 12 + future 6). Y-axis: tender count.
- Two bar groups per month: "Predicted" (striped pattern or reduced opacity) and "Actual" (solid).
- Past months show both (for accuracy calibration). Future months show only predictions.
- Accuracy annotation: "Prediction accuracy (last 6 months): 78%" displayed as a badge above the chart.

**Filters:**

- Sector: multi-select CPV.
- Contracting authority: searchable select.
- Value range: preset ranges.
- Confidence threshold: slider (0–100) — "Show predictions with ≥ [X]% confidence".

**AI disclaimer:** Persistent footnote: "Predictions are based on historical procurement patterns and cycle analysis. Actual publication dates and tender availability may vary. Predictions do not constitute guarantees."

**Data source:** `GET /api/analytics/pipeline?dateFrom=&dateTo=&sector=&authority=&confidenceMin=` returns `{ kpis: {...}, opportunities: [{ id, title, authority, value, publicationDate, isConfirmed, confidence, basis }], accuracyScore: number, historicalComparison: [...] }`.

**Responsive behavior:**

- ≥ 1024 px: full timeline layout as shown. KPI cards in row.
- 768–1023 px: timeline scrolls horizontally. KPI cards in row. Chart below.
- < 768 px: default view switches to "List" (table is most readable on small screens). Timeline available but requires horizontal scroll with visible scroll indicator. KPI cards stack vertically. Calendar view shows a compact month grid. Prediction chart full-width, labels angled 45°.

**UX Requirements:**

- **UX-AN20:** The Pipeline Forecast view SHALL display KPI cards for predicted opportunities next month, estimated total value, and count of high-confidence predictions.
- **UX-AN21:** The view SHALL offer three display modes — Timeline (Gantt-style), Calendar, and List — toggled via a view selector. The timeline SHALL visually distinguish confirmed opportunities (solid bars) from predicted ones (dashed, semi-transparent) with confidence percentages.
- **UX-AN22:** Clicking a predicted opportunity SHALL display a detail popover showing the prediction basis, estimated value, and an option to set a publication alert.
- **UX-AN23:** A historical accuracy chart SHALL display predicted vs. actual tender volumes for past months alongside future predictions, with an overall accuracy score annotation.
- **UX-AN24:** A persistent AI disclaimer SHALL be displayed noting that predictions are based on historical patterns and do not constitute guarantees.

---

### 4.7 Usage Analytics View

**Route:** `/analytics?tab=usage`

**Access:** All tiers (Starter, Professional, Enterprise).

**Purpose:** Provide detailed visibility into feature consumption, usage trends, and projected limit dates.

**Layout (desktop):**

```
┌──────────────────────────────────────────────────────────┐
│  Usage Analytics                  Billing period: Mar 1–31│
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │ Current Period Usage                              │    │
│  │                                                   │    │
│  │ AI Summaries        8 / 10   ████████░░  80%     │    │
│  │ Proposal Drafts     2 / 5    ████░░░░░░  40%     │    │
│  │ Compliance Checks   0 / 3    ░░░░░░░░░░   0%     │    │
│  │ Document Parses    12 / 20   ██████░░░░  60%     │    │
│  │ Storage           124/500MB  ██░░░░░░░░  25%     │    │
│  │                                                   │    │
│  │ ⚡ At current pace, you'll reach your summary     │    │
│  │   limit by March 22. [Upgrade]                    │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │ Usage Over Time (line chart, past 6 months)       │    │
│  │                                                   │    │
│  │       ___                                         │    │
│  │      /   \    /\                                  │    │
│  │  ___/     \__/  \___                              │    │
│  │                                                   │    │
│  │ — Summaries  — Proposals  — Compliance            │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │ Usage by Team Member                              │    │
│  │                                                   │    │
│  │ Member        Summaries  Proposals  Checks        │    │
│  │ M. Ivanova    4          1          0             │    │
│  │ G. Petrov     3          1          0             │    │
│  │ E. Todorova   1          0          0             │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │ 💡 Recommendation: You've exceeded your summary   │    │
│  │ limit in 3 of the last 6 months. Upgrading to    │    │
│  │ Professional (€149/mo) would give you 50          │    │
│  │ summaries and save ~€30/mo in add-on purchases.  │    │
│  │                                [View plans]       │    │
│  └──────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

**Current period usage section:**

Identical to the subscription settings meter (Section 3.6.2) but placed in the analytics context. Same progress bars with three-tier color coding.

**Projected limit date:**

- Calculated client-side: `(limit - used) / (used / daysElapsedInPeriod)` = days remaining.
- If projected to hit limit before period end: amber callout box with the projected date and an "Upgrade" link.
- If unlikely to hit limit: no callout displayed.

**Usage Over Time chart:**

- Recharts: `<LineChart>` — X-axis: months (past 6). Y-axis: usage count.
- One `<Line>` per metered feature. Different colors per feature, with legend.
- Horizontal dashed reference lines at each feature's limit (labeled on the right axis).
- Tooltip: month, usage per feature, limit.
- Area fill below each line at low opacity for visual weight.

**Team breakdown table (Professional + Enterprise only):**

- shadcn `Table` — columns: team member name, and one column per metered feature showing their individual consumption count.
- Sortable by any column.
- Footer row: totals (should match the period usage).
- Not shown for Starter tier (single user).

**Upgrade recommendation card:**

- Shown conditionally: when the user has exceeded limits in ≥ 2 of the last 6 months, OR when current month projection exceeds the limit.
- Card with `bg-blue-50 border-blue-200` styling.
- Content: personalized recommendation based on usage pattern, naming the suggested tier and estimated savings.
- CTA: "View plans" → navigates to `/settings/subscription`.
- Dismissable (X button, stored in localStorage, re-shown next billing period).

**Data source:** `GET /api/analytics/usage?months=6` returns `{ currentPeriod: { start, end, meters: [...] }, history: [{ month, meters: [...] }], teamBreakdown: [...], recommendation: { show: boolean, tier: string, message: string, estimatedSavings: number } }`.

**Responsive behavior:**

- ≥ 1024 px: layout as shown, single column.
- 768–1023 px: identical layout, chart may reduce horizontal padding.
- < 768 px: all sections stack vertically. Usage bars full-width. Chart: legend below chart instead of inline. Team table: horizontal scrollable or switches to per-member cards. Recommendation card full-width.

**UX Requirements:**

- **UX-AN25:** The Usage Analytics view SHALL display current billing period consumption for all metered features with progress bars, numeric used/limit values, and three-tier color coding.
- **UX-AN26:** When projected usage indicates a limit will be reached before the billing period ends, a projected limit date callout SHALL be displayed with an upgrade link.
- **UX-AN27:** A line chart SHALL show usage trends for each metered feature over the past 6 months, with horizontal reference lines at each feature's plan limit.
- **UX-AN28:** For multi-user plans, a team breakdown table SHALL show per-member consumption of each metered feature.
- **UX-AN29:** When usage patterns indicate consistent limit-exceeding behavior, a personalized upgrade recommendation card SHALL be displayed with the suggested tier, estimated savings, and a link to view plans.

---

### 4.8 Custom Report Builder

**Route:** `/analytics?tab=reports`

**Access:** Professional and Enterprise tiers.

**Purpose:** Allow users to create, preview, and export management reports combining selected metrics and visualizations.

**Layout (desktop):**

```
┌──────────────────────────────────────────────────────────┐
│  Reports                                    [+ New report]│
│                                                          │
│  Saved Reports                                           │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                 │
│  │ Pipeline │ │ Monthly  │ │ Team     │                  │
│  │ Summary  │ │ Win Rate │ │ Activity │                  │
│  │ Mar 2026 │ │ Q1 2026  │ │ Weekly   │                  │
│  │ [Edit]   │ │ [Edit]   │ │ [Edit]   │                  │
│  └──────────┘ └──────────┘ └──────────┘                 │
│                                                          │
│  ── Report Builder ──────────────────────────────        │
│                                                          │
│  ┌──── Configuration ────┐ ┌──── Live Preview ─────┐     │
│  │                       │ │                        │     │
│  │ Template: [Pipeline▾] │ │  ┌─────────────────┐  │     │
│  │                       │ │  │  Pipeline Summary│  │     │
│  │ Metrics:              │ │  │  Mar 2026        │  │     │
│  │ ☑ Win rate            │ │  │                  │  │     │
│  │ ☑ Pipeline value      │ │  │  KPIs...         │  │     │
│  │ ☐ Team breakdown      │ │  │  Chart...        │  │     │
│  │ ☑ Sector analysis     │ │  │  Table...        │  │     │
│  │                       │ │  └─────────────────┘  │     │
│  │ Date range:           │ │                        │     │
│  │ [This month ▾]        │ │                        │     │
│  │                       │ │                        │     │
│  │ Group by:             │ │                        │     │
│  │ [Sector ▾]            │ │                        │     │
│  │                       │ │                        │     │
│  │ [Export ▾] [Save]     │ │                        │     │
│  └───────────────────────┘ └────────────────────────┘     │
└──────────────────────────────────────────────────────────┘
```

**Saved reports section:**

- Grid of report cards. Each card: report name, template type badge, date range, last generated date.
- Actions per card: "Edit" (opens in builder), "Export" (quick re-export with current data), "Duplicate", "Delete", "Schedule".
- "New report" button: opens the builder with a blank configuration.

**Template selection:**

- Dropdown `Select` with options:
  - **Pipeline Summary:** active opportunities by stage, value, and sector.
  - **Success Rate Report:** win/loss breakdown by sector, region, team member.
  - **Deadline Calendar:** upcoming deadlines in a calendar/table format.
  - **Team Activity:** per-member activity log — proposals edited, documents uploaded, tasks completed.
  - **Custom:** start with an empty canvas and select individual metric blocks.

- Selecting a template pre-populates the metrics checkboxes and grouping.

**Metrics configuration:**

- Checkbox list of available metrics. Grouped by category:
  - **Pipeline:** open opportunities count, total pipeline value, stage distribution, sector breakdown.
  - **Performance:** win rate, loss reasons, average preparation time, average quality score.
  - **Financial:** total invested, total won, ROI, cost per bid.
  - **Team:** per-member metrics (only if team plan).
  - **Timeline:** upcoming deadlines, overdue items.
- Each checked metric adds a corresponding section to the preview.

**Date range:** Inherits from the global analytics date picker by default. Can be overridden per report.

**Grouping:** Dropdown — "By sector", "By region", "By team member", "By contracting authority", "None".

**Live preview:**

- Right panel (50 % width on desktop) shows a real-time rendering of the report as it will appear when exported.
- Updates dynamically as the user changes configuration.
- Scrollable if content exceeds viewport.
- Header: report title (editable inline), date range, generation timestamp.
- Content: KPI summary row, charts (rendered as static images for export fidelity), data tables.

**Export:**

- Dropdown menu with format options:
  - **PDF**: formatted report with branding (company name + EU Solicit watermark). Generated server-side via headless rendering.
  - **DOCX**: structured Word document with headings, tables, and embedded chart images.
  - **CSV**: raw data tables only (no charts). One CSV per data section, bundled in a ZIP.
- Export triggers download. Toast: "Report exported as [format]".

**Save:**

- "Save" persists the report configuration to the user's saved reports.
- Name input (auto-populated from template name + date range, editable).
- Saved reports appear in the saved reports grid.

**Scheduling (optional, Enterprise only):**

- "Schedule" action on saved reports.
- Dialog: frequency ("Weekly — every Monday", "Monthly — 1st of month"), recipients (email addresses), format (PDF/DOCX).
- Scheduled reports: indicated by a clock icon on the report card. "Next delivery: [date]".

**Data source:** `GET /api/reports/templates` for template definitions. `POST /api/reports/generate` with configuration → returns report data. `GET /api/reports/saved` for saved reports. `POST /api/reports/saved` to save. `POST /api/reports/[id]/export?format=pdf` for export. `POST /api/reports/[id]/schedule` for scheduling.

**Responsive behavior:**

- ≥ 1280 px: side-by-side layout — configuration (40 %) and preview (60 %).
- 1024–1279 px: same layout with slightly narrower preview.
- 768–1023 px: configuration full-width, preview below in a collapsible "Preview" section (expanded by default).
- < 768 px: configuration full-width. Preview hidden behind a "Show preview" button that opens a full-screen overlay. Saved reports grid becomes a vertical list. Export and save buttons sticky at bottom of screen.

**UX Requirements:**

- **UX-AN30:** The Custom Report Builder SHALL display a grid of saved reports and provide a "New report" action that opens a configuration interface with live preview.
- **UX-AN31:** Report configuration SHALL include template selection (Pipeline Summary, Success Rate, Deadline Calendar, Team Activity, Custom), metric checkboxes grouped by category, date range override, and grouping options.
- **UX-AN32:** The live preview panel SHALL update dynamically as the user modifies the report configuration, rendering KPI summaries, charts, and data tables as they will appear in the exported document.
- **UX-AN33:** Reports SHALL be exportable in PDF, DOCX, and CSV formats. PDF and DOCX SHALL include formatted charts and branding. CSV SHALL contain raw data tables bundled in a ZIP archive.
- **UX-AN34:** Saved reports SHALL support scheduling for automatic email delivery at weekly or monthly intervals. Scheduling SHALL be available to Enterprise tier users.agentId: a8b63d811d7a8ef1c (use SendMessage with to: 'a8b63d811d7a8ef1c' to continue this agent)
<usage>total_tokens: 34119
tool_uses: 0
duration_ms: 667378</usage>


---


# Section 5: Cross-Cutting UX Patterns

## 5.1 Empty States

Every list view, dashboard, and library screen has a designed empty state that guides the user toward their first meaningful action. Empty states are never just blank space. They consist of four elements rendered centered in the content area.

**Empty State Component Structure**

```
┌────────────────────────────────────────────────┐
│                                                │
│              [Illustration/Icon]               │
│              48px, text-tertiary               │
│                                                │
│              Headline (text-lg)                │
│         Description (text-sm, secondary)       │
│                                                │
│            [ Primary CTA Button ]              │
│         Optional secondary text link           │
│                                                │
└────────────────────────────────────────────────┘
```

**Empty State Specifications by Screen**

| Screen | Icon | Headline | Description | Primary CTA | Secondary |
|--------|------|----------|-------------|-------------|-----------|
| Dashboard (new user) | `Rocket` | Welcome to EU Solicit | Complete a few setup steps to get started. We'll help you find relevant opportunities and win more bids. | Complete Your Profile | Skip for now |
| Dashboard (new user) — inline | — | (3-step checklist below headline) | 1. Complete your company profile 2. Configure alert preferences 3. Explore opportunities | — | — |
| Opportunity List (no matches) | `SearchX` | No opportunities match your filters | Try adjusting your filters or broadening your search criteria. New opportunities are ingested daily. | Adjust Filters | Clear all filters |
| Opportunity List (no data at all) | `Inbox` | No opportunities available yet | Opportunities are being ingested from AOP, TED, and EU Grants portals. Check back shortly. | Configure Alerts | — |
| Proposal List (no proposals) | `FileText` | No proposals yet | Generate your first proposal from an opportunity you're tracking. Our AI assistant will draft a structured response based on the tender requirements. | Browse Opportunities | — |
| Bid Outcomes (no outcomes) | `Target` | No bid outcomes recorded | Recording bid outcomes helps you track your win rate and enables AI-powered lessons learned analysis. | Record an Outcome | Learn more about bid tracking |
| Analytics (insufficient data) | `BarChart3` | Not enough data yet | Analytics will populate as you track opportunities, submit proposals, and record bid outcomes. Start by exploring opportunities. | Explore Opportunities | — |
| Lessons Learned (no entries) | `Lightbulb` | No lessons learned yet | Record a bid outcome — win or loss — to unlock AI-powered analysis of what worked and what to improve. | Go to Bid Outcomes | — |
| Task Board (no tasks) | `CheckSquare` | No active tasks | Tasks are created automatically when you start working on a bid. Begin a proposal to see your task board populate. | Start a Proposal | — |
| Content Blocks (empty library) | `Library` | Build your content library | Save frequently used sections — company descriptions, team bios, past performance — to speed up future proposals. | Create First Block | Import from existing proposal |
| Calendar (no events) | `CalendarDays` | No upcoming deadlines | Track opportunities to see their submission deadlines, clarification periods, and evaluation timelines here. | Browse Opportunities | — |
| Alert Preferences (not configured) | `BellRing` | Set up your alert preferences | Tell us what you're looking for — CPV sectors, budget ranges, regions — and we'll notify you when matching opportunities appear. | Configure Alerts | — |
| Saved Filters (no saved filters) | `Bookmark` | No saved filters | Save your current filter combination to quickly access your preferred opportunity views. | — (contextual, appears in filter panel) | — |
| Team Members (solo user) | `Users` | You're the only team member | Invite colleagues to collaborate on proposals and share opportunity tracking. | Invite Team Member | — |

---

## 5.2 AI Operation States

All AI-powered features follow a single, consistent state machine. This ensures users always know what is happening, can cancel long operations, and understand usage impact. The state machine applies uniformly to all eleven AI features.

**State Machine**

```
IDLE ──[click]──► REQUESTING ──► STREAMING ──► COMPLETE
                       │              │              │
                       │              ├──► PARTIAL   │
                       │              │    FAILURE   │
                       └──► ERROR ◄───┘              │
                              │                      │
                              └──[retry]──► REQUESTING
```

**State Definitions**

| State | UI Treatment | Controls Available |
|-------|--------------|-------------------|
| **Idle** | Action button visible and enabled. Label describes the action: "Generate Summary", "Run Compliance Check", "Simulate Scoring", etc. Usage counter shown below button if within 30% of limit (e.g., "3 of 10 used this month"). | Click to start |
| **Requesting** | Button replaced with disabled state showing spinner + "Starting analysis...". No result area change yet. Timeout: if no response in 10 seconds, transition to Error with "Request timed out" message. | Cancel (returns to Idle, no usage charged) |
| **Streaming** (SSE operations) | Result area appears with progressive text rendering. Characters stream in at received rate. Above result area: stage indicator pill showing current phase. Phases are operation-specific (see table below). Below result area: "Stop generating" button. | Stop generating (keeps partial result, transitions to Complete with partial content) |
| **Processing** (non-streaming operations) | Result area shows pulsing skeleton/shimmer blocks matching expected output layout. Above skeleton: status text "Analyzing... this usually takes 10–30 seconds" with elapsed time counter. Progress bar (indeterminate) below status text. | Cancel (returns to Idle, no usage charged) |
| **Complete** | Result fully rendered. Success toast: "[Operation] complete". Usage counter incremented. Action button returns to Idle state with label changed to "Regenerate [Operation]". Timestamp shown: "Generated just now". If result replaces a previous result, "Previous version" link available. | Regenerate (returns to Requesting, charges usage), Copy to clipboard, Edit result (where applicable) |
| **Error** | Error alert banner in result area: red background, error icon, error message, "Retry" button. Below: reassurance text "This attempt won't count against your usage." Action button returns to Idle state. | Retry (returns to Requesting), Dismiss error |
| **Partial Failure** | Partial results rendered normally. Warning banner above results: amber background, warning icon, "Some sections could not be generated. You can retry the failed sections or edit manually." Failed sections show placeholder with "Retry this section" button. | Retry failed sections, Edit manually, Regenerate all |

**Streaming Stage Indicators by Operation**

| AI Operation | Stage 1 | Stage 2 | Stage 3 | Stage 4 |
|-------------|---------|---------|---------|---------|
| AI Summary | Analyzing document... | Extracting key points... | Drafting summary... | Finalizing... |
| Proposal Generation | Analyzing requirements... | Structuring response... | Drafting technical approach... | Drafting financial section... |
| Compliance Check | Loading framework... | Checking requirements... | Assessing gaps... | Generating report... |
| Scoring Simulator | Parsing evaluation criteria... | Scoring technical merit... | Scoring financial offer... | Calculating total... |
| Bid/No-Bid Analysis | Analyzing opportunity fit... | Evaluating capacity... | Assessing competition... | Generating recommendation... |
| Eligibility Check | Checking basic criteria... | Verifying certifications... | Checking financial thresholds... | Summarizing results... |
| Budget Builder | Analyzing cost categories... | Estimating labor costs... | Calculating overheads... | Generating budget table... |
| Logframe Generator | Identifying objectives... | Defining activities... | Setting indicators... | Generating matrix... |
| Clause Risk Analysis | Extracting clauses... | Assessing risk level... | Identifying red flags... | Generating report... |
| Pricing Assistant | Analyzing market rates... | Benchmarking competitors... | Modeling scenarios... | Generating recommendation... |
| Lessons Learned | Analyzing bid outcome... | Identifying patterns... | Extracting insights... | Generating recommendations... |

**Usage Counter Component**

Displayed inline below or beside the AI action button when relevant.

| State | Display |
|-------|---------|
| 0–69% used | Muted text: "3 of 10 used this month" |
| 70–89% used | Amber text: "8 of 10 used this month" |
| 90–99% used | Amber text + warning icon: "9 of 10 used — 1 remaining" |
| 100% used | Button disabled. Red text: "Monthly limit reached." Inline upgrade link: "Upgrade to Professional for 50/month." |

---

## 5.3 In-App Notification Center

**Trigger**

Bell icon in the top bar. When unread notifications exist, a red badge displays the count (capped at "99+"). The badge uses the `--transition-spring` animation to pop in when new notifications arrive.

**Notification Drawer**

| Property | Specification |
|----------|---------------|
| Component | Sheet, side="right", width 380px (desktop), 100vw (mobile) |
| Header | "Notifications" title + "Mark all as read" text button (right-aligned) + close button |
| Footer | "Notification Preferences" link → navigates to Settings > Notifications |
| List | Scrollable list of notification items, ordered by timestamp (newest first) |
| Grouping | Grouped by date: "Today", "Yesterday", "This Week", "Earlier" |
| Empty state | Bell icon + "You're all caught up" + "No new notifications" |

**Notification Item Structure**

```
┌─────────────────────────────────────────────────┐
│ [Icon]  Title text                    2h ago    │
│         Description text spanning               │
│         up to two lines max                     │
│                                       [● unread]│
└─────────────────────────────────────────────────┘
```

| Element | Specification |
|---------|---------------|
| Icon | Category-specific icon in a 32px colored circle (color matches notification type) |
| Title | Bold, text-sm, single line, truncated with ellipsis |
| Description | Regular, text-sm, text-secondary, max 2 lines |
| Timestamp | Relative time ("2h ago", "yesterday"), text-xs, text-tertiary |
| Unread indicator | 8px indigo dot, right-aligned. Disappears when read. |
| Click behavior | Navigates to the relevant screen and entity. Marks notification as read. Closes drawer. |
| Hover | Background highlight (`--color-bg-tertiary`) |

**Notification Types**

| Type | Icon | Color | Title Pattern | Description Pattern | Navigation Target |
|------|------|-------|---------------|--------------------|--------------------|
| Deadline approaching (3d) | `Clock` | Amber | Deadline in 3 days | "[Opportunity title]" submission deadline is [date]. | Opportunity detail |
| Deadline approaching (1d) | `AlertTriangle` | Red | Deadline tomorrow | "[Opportunity title]" submission deadline is tomorrow at [time]. | Opportunity detail |
| Deadline overdue | `XCircle` | Red | Deadline passed | The submission deadline for "[Opportunity title]" has passed. | Opportunity detail |
| Approval request | `UserCheck` | Indigo | Approval requested | [User name] requested your approval on "[Proposal title]". | Proposal detail, approval section |
| Approval decision | `CheckCircle` / `XCircle` | Green / Red | Proposal approved / rejected | [User name] [approved/rejected] "[Proposal title]". [Comment preview]. | Proposal detail |
| Compliance framework assigned | `Shield` | Blue | Compliance framework assigned | "[Framework name]" has been assigned to "[Opportunity title]". | Opportunity compliance tab |
| AI operation complete | `Sparkles` | Indigo | [Operation] complete | Your [AI summary/compliance check/etc.] for "[entity title]" is ready. | Relevant entity detail tab |
| Trial expiry warning | `AlertTriangle` | Amber | Trial ending soon | Your Professional trial ends in [N] days. Upgrade to keep your features. | Billing settings |
| Usage limit approaching (80%) | `AlertTriangle` | Amber | Usage limit approaching | You've used [N] of [M] [feature] this month. | Billing settings, usage tab |
| Usage limit reached (100%) | `XCircle` | Red | Usage limit reached | You've reached your monthly limit for [feature]. Upgrade for more. | Billing settings, upgrade flow |
| Team invite | `UserPlus` | Indigo | You've been invited | [User name] invited you to join [Organization name] on EU Solicit. | Invite acceptance page |
| Bid outcome reminder | `Target` | Slate | Record your bid outcome | It's been 30 days since the deadline for "[Opportunity title]". Have you heard back? | Bid outcomes, pre-filled form |
| New matching opportunity | `Search` | Green | New matching opportunity | "[Opportunity title]" matches your alert preferences. Budget: [amount]. Deadline: [date]. | Opportunity detail |

**Real-Time Delivery**

Notifications are delivered in real-time via Server-Sent Events (SSE). When a new notification arrives while the app is open:
1. Bell badge count increments with spring animation
2. If drawer is open, new notification slides in at the top of the list with a subtle highlight animation
3. If drawer is closed, no toast is shown for non-critical notifications (to avoid interruption). Critical notifications (deadline tomorrow, approval request) also trigger a toast.

---

## 5.4 Global Search and Filter System

### Global Search (Cmd+K)

**Behavior**

| Property | Specification |
|----------|---------------|
| Trigger | `Cmd+K` (Mac) / `Ctrl+K` (Win/Linux), or click the search input in the top bar |
| Component | shadcn/ui Command (wraps cmdk) rendered in a Dialog overlay |
| Width | 640px max, centered horizontally, positioned at 20% from top of viewport |
| Input | Text input, auto-focused on open. Placeholder: "Search opportunities, proposals, companies..." |
| Debounce | 200ms after last keystroke before issuing search request |
| Min query length | 2 characters (below that, show only recent searches and quick actions) |
| Max results per category | 5, with "View all [N] results" link at bottom of each category |
| Keyboard | Arrow Up/Down to navigate items (category headers skipped), Enter to select, Escape to close, Tab to jump between categories |

**Result Categories and Display**

| Category | Icon | Displayed Fields | Action on Select |
|----------|------|------------------|------------------|
| Recent Searches | `Clock` | Search query text | Re-execute that search |
| Opportunities | `Search` | Title, CPV code, contracting authority, deadline, source badge | Navigate to Opportunity detail |
| Proposals | `FileText` | Title, linked opportunity, status badge | Navigate to Proposal detail |
| Organizations | `Building2` | Organization name, country | Navigate to Organization detail (admin) or Company profile |
| Quick Actions | `Zap` | Action label (e.g., "Go to Settings", "Create New Proposal", "Open Calendar") | Execute action / navigate |

**Search Index**

The search query matches against:
- Opportunity: title, description, CPV code and description, contracting authority name, reference number
- Proposal: title, linked opportunity title
- Organization: name, registration number
- Quick Actions: hardcoded list of navigable pages and common actions

**No Results State**

When no results match: "No results for '[query]'" with suggestions: "Check your spelling", "Try more general terms", "Search by CPV code (e.g., 72000000)".

### Opportunity List Filters

The opportunity list is the most filter-intensive view in the application. The filter system is designed for power users managing hundreds of opportunities while remaining approachable for new users.

**Filter Bar (Always Visible)**

Horizontal bar above the opportunity list:

```
┌────────────────────────────────────────────────────────────────────┐
│ [🔍 Search...]  [Source ▾] [Region ▾] [Sector ▾] [More ▾]  [⚙]  │
│ Active: [CPV: 72000000 ✕] [Region: Bulgaria ✕] [Budget: >50k ✕] │
└────────────────────────────────────────────────────────────────────┘
```

| Element | Behavior |
|---------|----------|
| Search input | Free text search across opportunity title, description, authority name, reference number. Debounced 300ms. |
| Quick filter dropdowns | Source, Region, and Sector open as small Popover panels with checkbox lists. Selections immediately filter the list. |
| "More" button | Opens the full filter panel (Sheet on mobile, popover or inline sidebar on desktop). |
| Settings icon | Column visibility toggle for the data table. |
| Active filter chips | Below the bar, each active filter is shown as a chip with the filter name, value, and an X to remove. "Clear all" link appears when 2+ filters are active. |

**Full Filter Panel**

| Filter | Control | Specification |
|--------|---------|---------------|
| Source | Checkbox group | Options: AOP, TED, EU Grants. Multi-select. |
| Region / Country | Multi-select with search | Searchable dropdown listing all EU countries + "All EU" option. Tier-gated: Free tier sees only home country; Professional sees configured countries; Enterprise sees all. Locked countries show lock icon + "Upgrade" tooltip. |
| CPV Sector | Hierarchical tree picker | CPV codes organized in a tree structure (Division > Group > Class > Category). Search input at top filters the tree. Selecting a parent selects all children. Selected items shown as chips. |
| Budget Range | Dual-handle range slider | Min and max handles. Input fields for exact values. Preset buttons: "< 50k", "50k–250k", "250k–1M", "> 1M". Currency display in EUR. |
| Deadline | Date range picker | Calendar-based range selector. Preset buttons: "Closing this week", "Closing this month", "Next 3 months", "Next 6 months". Custom range via calendar UI. |
| Status | Checkbox group | Options: Open, Closing Soon (< 7 days), Closed, Awarded. Default: Open + Closing Soon. |
| Type | Checkbox group | Options: Tender, Grant. |
| Relevance Score | Slider | Minimum relevance score (0–100). Default: 0 (show all). Label: "Minimum relevance: 65%". |

**Saved Filters**

| Property | Specification |
|----------|---------------|
| Save action | "Save current filters" button in filter panel or bar. Opens a small dialog: name input + Save button. |
| Storage | Saved per user in the database. Available across devices. |
| Access | Dropdown in the filter bar listing saved filters. Click to apply. |
| Management | Edit name, delete. Accessible from the dropdown or from Settings > Saved Filters. |
| Default filter | One saved filter can be marked as "default" — automatically applied when the user opens the Opportunity list. |
| Limit | Free: 3 saved filters. Professional: 20. Enterprise: unlimited. |

**URL Synchronization**

All active filters are encoded in the URL query string. This enables:
- Sharing a filtered view via URL (copy-paste)
- Browser back/forward navigating filter states
- Bookmarking specific filter combinations

URL parameter format: `?source=aop,ted&region=BG,RO&cpv=72000000&budget_min=50000&budget_max=250000&deadline=2026-04-01,2026-06-30&status=open,closing&type=tender&relevance=65`

Filter state is managed in Zustand and synced bidirectionally with URL params via a custom `useFilterSync` hook that wraps `next/navigation`'s `useSearchParams` and `useRouter`.

---

## 5.5 Mobile Responsive Adaptations

This section specifies concrete responsive behavior for each major UI component when viewed on screens below the mobile breakpoint (< 768px) unless otherwise noted.

### Sidebar to Bottom Tab Bar

| Property | Specification |
|----------|---------------|
| Component | Fixed bottom bar, 56px height + safe area inset (for phones with home indicators) |
| Background | `--color-bg-primary` with top border `--color-border-default` and subtle `--shadow-md` upward |
| Items | 5 icon tabs: Dashboard (`LayoutDashboard`), Opportunities (`Search`), Proposals (`FileText`), Calendar (`CalendarDays`), More (`Menu`) |
| Active state | Icon + label colored with `--color-primary-600`. Inactive: `--color-text-tertiary`. |
| Labels | 10px text below each icon. Always visible (not icon-only). |
| "More" tab | Opens a full-screen Sheet (from bottom) listing: Grants, Analytics, Content Library, Lessons Learned, Settings, Help, Sign Out. |
| Badge indicators | Unread/count badges render as small dots on the relevant tab icon. |
| Hiding on scroll | Bottom bar hides on scroll-down, reappears on scroll-up or scroll-to-top (with 150ms slide transition). This reclaims 56px of viewport for content. |

### DataTable to Card List

When viewport width is below 768px, the `DataTable` component renders an alternative card-list layout.

**Opportunity Card (Mobile)**

```
┌────────────────────────────────────────────┐
│ [AOP]  Open                    ⋯ (actions) │
│                                             │
│ Title of the Opportunity                    │
│ Truncated to 2 lines maximum               │
│                                             │
│ Contracting Authority Name                  │
│                                             │
│ 💰 €50,000 – €100,000                      │
│ 📅 Deadline: Apr 15, 2026 (10 days)        │
│ 📊 Relevance: 85%                          │
└─────────────────────────────────────────────┘
```

| Element | Specification |
|---------|---------------|
| Source badge + Status badge | Top row, left-aligned. Small badges. |
| Actions menu | Top row, right-aligned. `MoreHorizontal` icon opening DropdownMenu. |
| Title | text-base, font-semibold. Max 2 lines, truncated with ellipsis. Full card is tappable (navigates to detail). |
| Authority | text-sm, text-secondary. Single line, truncated. |
| Metadata row: Budget | text-sm. Range or single value. |
| Metadata row: Deadline | text-sm. Formatted date + relative countdown. Color-coded: green (> 7 days), amber (3–7 days), red (< 3 days). |
| Metadata row: Relevance | text-sm. Percentage with small inline progress bar. |
| Swipe actions | Swipe left reveals: "Track" (indigo), "Summarize" (slate). 60px each. Spring-back if not swiped far enough. |
| Spacing | 8px gap between cards. Cards have 16px internal padding. |

### Detail Pages

| Property | Specification |
|----------|---------------|
| Layout | Full-screen, no sidebar. Top: back button (left arrow + "Back") + page title (truncated). |
| Tabs | Horizontal scrollable pill tabs below the header. Active tab auto-scrolls into view. Scroll fade indicators on edges. |
| Content | Single column, full width. Sections stack vertically. |
| Actions | Primary action as a sticky bottom bar (e.g., "Start Proposal" button fixed at bottom). Secondary actions in the `MoreHorizontal` menu in the header. |
| Inspector / sidebar content | Content that appears in the right inspector on desktop is either integrated into the main flow (above the fold) or accessible via a dedicated tab. |

### Right Inspector Panel

On screens below 1024px, the inspector panel renders as a Sheet from the bottom (on mobile) or from the right (on tablet).

| Viewport | Behavior |
|----------|----------|
| < 768px | Sheet from bottom, 90vh max height, drag handle for resize, swipe-down to dismiss |
| 768px–1023px | Sheet from right, 60vw width (max 420px), backdrop scrim, click outside to dismiss |

### Modals and Dialogs

| Viewport | Behavior |
|----------|----------|
| >= 768px | Standard centered modal with backdrop. Width per dialog size spec (sm/md/lg). |
| < 768px | Full-screen dialog. No backdrop (fills viewport). Header becomes a top bar with back/close button. Content scrolls vertically. Action buttons pinned at bottom of screen (sticky footer). |

### Filter Panel

| Viewport | Behavior |
|----------|----------|
| >= 1024px | Popover panels for quick filters. "More" opens an inline sidebar section or wide popover. |
| 768px–1023px | Full filter panel opens as a Sheet from right (60vw). |
| < 768px | Full filter panel opens as a Sheet from bottom (90vh). Filter bar simplified to: search input + single "Filters" button (showing active filter count badge). |

### Charts (Recharts)

| Adaptation | Specification |
|------------|---------------|
| Container | Horizontal scrollable container with min-width set to maintain readable axis labels. Scroll indicators (fade edges) on left/right. |
| Axis labels | Abbreviated: month names to 3-letter, large numbers to "50k" format. Rotated 45 degrees if needed. |
| Legend | Moved from side to below the chart. Horizontal wrap layout. |
| Tooltips | Tap-triggered (not hover). Tap on data point to show tooltip; tap elsewhere to dismiss. |
| Bar/line sizing | Bars: min-width 24px. Lines: stroke-width 2px. Touch target for data points: 44px invisible hit area. |

### Proposal Editor (Tiptap)

| Adaptation | Specification |
|------------|---------------|
| Toolbar | Simplified single row: Bold, Italic, List (bullet/numbered toggle), Heading (dropdown), Link, AI Assist. Secondary tools (table, image, code block) in an overflow menu ("+" button). |
| Toolbar position | Sticky at top of editor viewport. Does not float or reposition on selection (mobile keyboards make floating toolbars unreliable). |
| Slash commands | Available via "/" keypress as on desktop, but rendered as a bottom Sheet menu for easier thumb access. |
| Selection | Standard OS text selection handles. No custom selection UI. |
| Block controls | Block drag handle hidden on mobile. Reorder blocks via a "Move up/down" option in the block menu (long-press or "..." button per block). |

### Multi-Column Layouts

| Desktop Layout | Mobile (< 768px) | Tablet (768–1023px) |
|----------------|-------------------|---------------------|
| 3-column grid (dashboard metrics) | Single column stack | 2-column grid |
| 2-column form (label left, input right) | Single column (label above input) | Single column (label above input) |
| Side-by-side comparison (tiers) | Horizontal scroll or tab-based switching | 2-column (if 2 tiers) or horizontal scroll (if 3+) |
| Content + sidebar (settings) | Single column, sidebar content at top or in "More" tab | Single column, sidebar becomes horizontal tabs at top |

### Drag-and-Drop

| Desktop Interaction | Mobile Equivalent |
|--------------------|-------------------|
| Click-drag to reorder (task board columns, document list) | Long-press (500ms) to initiate drag. Haptic feedback on drag start. Drop targets highlighted with colored borders. |
| Drag files from OS to upload zone | Tap upload zone to open system file picker. |
| Drag to resize (panels, columns) | Not available on mobile. Fixed layouts used instead. |

---

## 5.6 Trial and Downgrade Experience

### Trial Countdown

The trial period is 14 days on the Professional tier. The countdown is communicated through progressively urgent UI treatments.

**Trial Banner**

| Days Remaining | Banner Style | Banner Text | Position | Dismissible |
|----------------|-------------|-------------|----------|-------------|
| 14–4 days | Subtle: `--color-bg-secondary` background, `--color-text-secondary` text, 40px height | "You have [N] days left on your Professional trial." + "Upgrade" link | Top of main content area, below top bar | Yes (dismissed for session; reappears next session) |
| 3–2 days | Amber: `--color-warning-bg` background, `--color-warning` text, 44px height | "Your trial ends in [N] days. Upgrade to keep AI features, multi-country tracking, and team collaboration." + "Upgrade Now" button | Same | No (cannot be dismissed) |
| 1 day | Red: `--color-error-bg` background, `--color-error` text, 44px height | "Your trial ends tomorrow. Upgrade now to avoid losing access to Professional features." + "Upgrade Now" button | Same | No |
| 0 (trial active today) | Red with emphasis: solid `--color-error` background, white text, 48px height | "Your trial ends today." + "Upgrade Now" button + "What happens when my trial ends?" link | Same | No |

**Trial Expiry Interstitial**

When a user logs in after their trial has expired and they have not upgraded:

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│                 Your Professional Trial Has Ended             │
│                                                              │
│  You still have access to all your data. Here's what changes │
│  on the Free plan:                                           │
│                                                              │
│  ┌─────────────┬──────────────────┬────────────────────┐     │
│  │             │ Free (current)   │ Professional       │     │
│  ├─────────────┼──────────────────┼────────────────────┤     │
│  │ Countries   │ 1                │ Up to 5            │     │
│  │ AI Summ.    │ 10/month         │ 50/month           │     │
│  │ Proposals   │ 3 active         │ Unlimited          │     │
│  │ Team        │ 1 user           │ Up to 10           │     │
│  │ Analytics   │ Basic            │ Full suite         │     │
│  └─────────────┴──────────────────┴────────────────────┘     │
│                                                              │
│  Your 12 proposals created during the trial will become      │
│  read-only on the Free plan.                                 │
│                                                              │
│  [ Upgrade to Professional — €49/month ]                     │
│  [ Continue on Free Plan ]                                   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

| Property | Specification |
|----------|---------------|
| Display | Full-screen interstitial overlaying the app. Cannot be bypassed without choosing an option. |
| "Upgrade" CTA | Primary button, indigo. Links to billing/checkout flow. |
| "Continue on Free" CTA | Ghost/text button. Acknowledges downgrade and enters the app. |
| Comparison table | Shows exactly the features that change. Dynamically populated based on what the user actually used during the trial (e.g., if they created 12 proposals, that number is shown). |
| Frequency | Shown once on first login after trial expiry. If "Continue on Free" is chosen, a smaller "Upgrade" banner persists for 30 days but the interstitial does not reappear. |

### Post-Trial State

| Aspect | Behavior |
|--------|----------|
| Proposals over limit | Proposals created during the trial that exceed the Free tier limit (3 active) are set to read-only. User sees a "Read-only — Upgrade to edit" badge on each. They can view and export but not modify. |
| AI operations over limit | Any AI-generated content (summaries, compliance checks, etc.) created during the trial remains visible. New AI operations are subject to Free tier limits. |
| Multi-country alerts | Alerts for countries beyond the Free tier's single-country allowance are paused (not deleted). Paused alerts show a "Paused — Upgrade to resume" indicator in Settings > Alerts. |
| Team members | If the org had multiple users during the trial, all users retain access but only the owner can make changes. Other users are set to "viewer" role. Banner on their dashboard: "Your organization's trial has ended. Contact [owner name] to upgrade." |

### Downgrade Confirmation Flow (Voluntary Downgrade)

When a user on Professional or Enterprise initiates a downgrade from Settings > Billing:

**Step 1: Impact Preview**

```
┌──────────────────────────────────────────────────────────────┐
│  You're about to downgrade to Free                           │
│                                                              │
│  Here's what will change:                                    │
│                                                              │
│  Feature             Current (Pro)    After (Free)           │
│  ─────────────────   ─────────────    ────────────           │
│  Active proposals    Unlimited        3 (12 will be locked)  │
│  AI summaries/mo     50               10                     │
│  Countries tracked   5                1                      │
│  Team members        10               1                      │
│  Compliance checks   Unlimited        5/month                │
│  Analytics           Full             Basic                  │
│                                                              │
│  [ Next: Review Data Impact ]        [ Cancel ]              │
└──────────────────────────────────────────────────────────────┘
```

**Step 2: Data Impact Summary**

| Data Type | Impact Message |
|-----------|---------------|
| Proposals | "Your 12 active proposals will become read-only. The 3 most recently modified will remain editable." |
| Alerts | "Alerts for Romania, Germany, France, and Italy will be paused. Your Bulgaria alerts will remain active." |
| Team | "9 team members will lose edit access and become viewers. Only you (owner) will retain full access." |
| Content blocks | "Your 24 content blocks will remain accessible (content library is not tier-gated)." |
| Analytics | "Historical analytics data is retained but advanced charts (win/loss trends, competitor analysis) will be hidden." |

Below the summary: "We recommend exporting your data before downgrading." + "Export All Data" button (triggers a full data export as JSON/CSV zip).

**Step 3: Confirmation**

| Element | Specification |
|---------|---------------|
| Confirmation text | "Type 'DOWNGRADE' to confirm" (for destructive action safety) |
| Confirm button | Red, disabled until text matches. Label: "Downgrade to Free" |
| Cancel button | Outlined, always available |

**Grace Period**

After a downgrade takes effect:
- 7-day grace period during which data created on the higher tier remains accessible (read-only)
- After 7 days: same as post-trial state (over-limit proposals locked, alerts paused, team restricted)
- Banner during grace period: "You have [N] days left to access your Professional data. Upgrade to restore full access."

---

## 5.7 Usage Limit Warnings

Usage limits are communicated progressively to avoid surprise and minimize friction, while encouraging upgrades at the moment of highest perceived value.

**Warning Thresholds**

| Threshold | UI Treatment | Location |
|-----------|-------------|----------|
| 0–69% used | No indicator unless user is on the relevant screen. On the feature screen: muted counter in normal text. Example: "3 of 10 AI summaries used this month" below the action button. | Inline, near the AI action button |
| 70–79% used | Counter text turns slightly emphasized: same position, `--color-text-primary` instead of `--color-text-tertiary`. | Inline, near the AI action button |
| 80–89% used | Counter turns amber (`--color-warning`). In the top bar: a small usage ring appears in the notification area showing the most-constrained resource. | Inline + top bar usage widget |
| 90–99% used | Counter is amber with warning icon. Text changes to emphasize scarcity: "1 remaining this month" or "2 remaining this month". Top bar usage ring is amber. | Inline + top bar usage widget |
| 100% used | Feature action button is disabled (grayed out, cursor: not-allowed). Inline upgrade prompt replaces the counter: "You've used all 10 AI summaries this month. [Upgrade to Professional →] for 50/month." Top bar usage ring is red. | Inline + top bar usage widget |

**Top Bar Usage Widget**

| Property | Specification |
|----------|---------------|
| Component | Small circular progress ring (20px diameter) + resource icon |
| Position | Top bar, between language switcher and notification bell |
| Visibility | Only visible when any resource is at 80%+ usage |
| Display | Shows the most-constrained resource (highest % used). If multiple are at limit, shows the most recently used one. |
| Interaction | Click opens a popover showing all tracked usage meters |
| Popover content | List of metered resources: name, usage bar (filled proportionally), "X of Y used" text, upgrade link if any are at 100% |

**Usage Meters Tracked**

| Resource | Free Limit | Professional Limit | Enterprise Limit |
|----------|------------|--------------------|--------------------|
| AI Summaries | 10/month | 50/month | Unlimited |
| Proposal Generations | 3/month | 25/month | Unlimited |
| Compliance Checks | 5/month | Unlimited | Unlimited |
| Scoring Simulations | 3/month | 25/month | Unlimited |
| Bid/No-Bid Analyses | 5/month | 25/month | Unlimited |
| Active Proposals | 3 (total, not per month) | Unlimited | Unlimited |
| Tracked Countries | 1 | 5 | All EU |

**Reset Behavior**

Usage counters reset on the first of each month at 00:00 UTC. When a counter resets:
- Top bar widget disappears if all resources drop below 80%
- Previously disabled buttons become enabled
- No notification sent for reset (to avoid noise)

---

## 5.8 Data Source Attribution

Every opportunity in the system originates from one of three sources: AOP (Annual Procurement Plans), TED (Tenders Electronic Daily), or EU Grants Portal. Source attribution is visible at every level of the UI where opportunities appear.

**Source Badge Specifications**

| Source | Label | Color | Background | Border |
|--------|-------|-------|------------|--------|
| AOP | AOP | White text | `--color-source-aop` (#2563eb, blue-600) | None |
| TED | TED | White text | `--color-source-ted` (#16a34a, green-600) | None |
| EU Grants | EU Grants | White text | `--color-source-eu-grants` (#9333ea, purple-600) | None |

Badge size: `text-xs`, `px-2 py-0.5`, `rounded-sm` (4px). Font weight: medium (500).

**Attribution Placement**

| Context | Placement | Additional Detail |
|---------|-----------|-------------------|
| Opportunity card (list view) | Top-left of card, before status badge | — |
| Opportunity table row | Dedicated "Source" column, or inline badge in the title column | — |
| Opportunity detail page header | Badge beside the opportunity title | + "View on [source portal]" link to the original listing URL |
| Inspector panel (opportunity) | Below title, above metadata | + original reference number |
| Search results (Cmd+K) | Small badge after the opportunity title | — |
| Calendar event | Badge on the event card | — |
| Notification | Mentioned in description text | "from [source]" |

**Source Filter**

In the opportunity list filter bar, "Source" is one of the quick-access filter dropdowns (always visible, not hidden behind "More"). It renders as a checkbox group with the three source badges as labels, making it visually distinctive and fast to use.

**Source-Specific Metadata**

Each source may provide different metadata fields. The opportunity detail page adapts:

| Field | AOP | TED | EU Grants |
|-------|-----|-----|-----------|
| Reference number | AOP plan reference | TED notice number | Call identifier |
| Publication date | Plan publication date | Notice publication date | Call opening date |
| Submission deadline | Estimated (from plan) | Firm deadline | Firm deadline |
| Budget | Estimated budget range | Contract value / range | Available funding envelope |
| Procedure type | Indicated procedure | Specific procedure (open, restricted, etc.) | Grant type (action grant, operating grant, etc.) |
| Contracting authority | Procuring entity | Contracting authority | DG / Agency |
| Original URL | Link to AOP portal | Link to TED notice | Link to Funding & Tenders portal |

When a field is not available from a particular source, the UI shows "Not specified" in muted text rather than omitting the field entirely, maintaining consistent layout.

---

## 5.9 Parsed Document Results Display

When the AI parser processes a tender package (uploaded documents or linked source), the extracted structured data is displayed on the Opportunity Detail page across five tabs. Each tab has a specific layout optimized for its content type.

### Overview Tab

The default tab shown when opening an opportunity detail. Presents the highest-value extracted information at a glance.

| Section | Content | Layout |
|---------|---------|--------|
| Summary | AI-generated 3–5 sentence summary of the opportunity. Boxed in a subtle card with a `Sparkles` icon indicating AI-generated content. "Regenerate" link below. | Full-width card at top |
| Key Dates | Publication date, clarification deadline, submission deadline, estimated evaluation period, estimated award date. Each with relative countdown ("in 12 days"). Color-coded: green > 7 days, amber 3–7 days, red < 3 days, gray if passed. | 2-column grid (1-col on mobile) |
| Financial | Budget / contract value (range or fixed), currency, funding source, co-financing requirements if applicable. | Card with labeled rows |
| Contracting Authority | Name, address, contact person (if available), authority type, link to authority profile (if exists in system). | Card with labeled rows |
| Quick Actions | Row of buttons: "Check Eligibility", "Bid/No-Bid Analysis", "Start Proposal". Gated by tier (locked buttons for unavailable features). | Horizontal button group (stacked on mobile) |

### Requirements Tab

Displays the structured list of all extracted requirements from the tender documents.

| Element | Specification |
|---------|---------------|
| Requirement list | Vertical list of requirement items. Each item is an expandable card. |
| Requirement card (collapsed) | Requirement number + title + category tag (Technical, Financial, Administrative, Legal) + compliance status indicator (if compliance check has been run: green check, amber warning, red X, gray not-checked). |
| Requirement card (expanded) | Full requirement text + source reference (document name + page number) + notes field (user-editable, for team annotations). |
| Category filter | Horizontal pill filter at top: All, Technical, Financial, Administrative, Legal. Counts shown on each pill. |
| Search | Search input above the list for filtering by requirement text. |
| Export | "Export Requirements" button: downloads as CSV or copies to clipboard for pasting into compliance matrices. |
| Empty state | "No requirements extracted yet. Upload tender documents to enable AI parsing." + Upload CTA. |

### Evaluation Criteria Tab

Displays the scoring methodology and weight breakdown extracted from tender documents.

| Element | Specification |
|---------|---------------|
| Total points display | Large text at top: "Total: 100 points" (or whatever the scale is). |
| Criteria table | Table with columns: Criterion Name, Category (Technical / Financial / Qualitative), Max Points, Weight (%), Description. Sortable by weight. |
| Visual breakdown | Horizontal stacked bar chart (Recharts) showing proportional weights. Color-coded by category. |
| Scoring detail | Each criterion row is expandable. Expanded view shows: full description of what evaluators are looking for, sub-criteria if any, scoring guidance (how points are allocated). |
| Simulate button | "Simulate Your Score" button links to the Scoring Simulator AI feature. Shows the criteria as input fields where the user estimates their score per criterion. |
| Empty state | "No evaluation criteria extracted. This may be a price-only tender, or the criteria might be in a document we haven't parsed yet." + "Upload Documents" CTA. |

### Documents Tab

Displays the list of documents associated with the opportunity: both source documents (from the tender package) and required submission documents.

**Source Documents Section**

| Column | Content |
|--------|---------|
| Document name | File name or title as extracted |
| Type | Contract notice, Terms of Reference, Technical specifications, Financial forms, etc. |
| Source | Where it was obtained (uploaded by user, fetched from TED, etc.) |
| Status | Parsed (green) / Parsing (amber spinner) / Not parsed (gray) / Error (red) |
| Actions | View (opens in browser/viewer), Download, Re-parse (if error), Delete |

**Required Submission Documents Section**

| Column | Content |
|--------|---------|
| Document name | Required document as extracted from tender requirements |
| Category | Administrative, Technical, Financial |
| Status | Available (user has uploaded/linked), Missing (not yet provided), Not Applicable (user-marked) |
| Actions | Upload, Link from library, Mark N/A |

A progress indicator at the top of this section: "8 of 12 required documents ready" with a progress bar.

### Timeline Tab

Displays key dates as a visual timeline, providing a chronological view of the opportunity lifecycle.

**Timeline Visualization**

```
Publication ──── Clarification ──── Submission ──── Evaluation ──── Award
   Mar 1           Mar 20             Apr 15         May–Jun         Jul 1
     ●───────────────●──────────────────●──────────────●──────────────●
                          ▲ TODAY (Apr 5)
```

| Property | Specification |
|----------|---------------|
| Component | Custom SVG or div-based horizontal timeline. Not a Recharts chart (too rigid for this layout). |
| Milestones | Circles on the line. Past milestones: filled, muted color. Current/next: filled, primary color, larger. Future: outlined, muted. |
| Today marker | Vertical dashed line labeled "Today" positioned proportionally on the timeline. |
| Labels | Date below each milestone. Event name above. |
| Detail | Click/tap a milestone to see a popover with full details: exact date/time, description, any notes. |
| Responsive | On mobile (< 768px): vertical timeline (top to bottom) instead of horizontal. Each milestone is a row: date on left, content on right. |
| Dates | Publication date, Clarification request deadline, Submission deadline, Estimated evaluation period (range), Estimated award date, Contract start date (if known). |
| Missing dates | If a date is not available, the milestone is shown as a dashed outline with "TBD" label. |

**Countdown Cards**

Below the timeline: 2–3 cards highlighting the most important upcoming dates with large countdown numbers.

| Card | Content |
|------|---------|
| Next deadline | "Submission Deadline" + date + large countdown: "10 days, 4 hours" + "Add to Calendar" button |
| Clarification deadline | "Clarification Deadline" + date + countdown (if still in future) |
| Evaluation period | "Estimated Evaluation" + date range + "We'll remind you to check for results" |

---

## 5.10 White-Label DNS Provisioning Flow (Admin)

This flow is accessed by admin users from the Admin Panel under White-Label management. It provisions a custom-branded instance of EU Solicit for a client organization.

### Step 1: Subdomain Configuration

| Element | Specification |
|---------|---------------|
| Input | Text input with `.eusolicit.com` suffix fixed. User types the subdomain portion. Example: user types `acme`, preview shows `acme.eusolicit.com`. |
| Validation | Real-time: checks for valid subdomain characters (a-z, 0-9, hyphens, no leading/trailing hyphens). Async: checks availability against existing subdomains (debounced 500ms). |
| Availability feedback | Green check + "Available" or red X + "Already taken — try another" below the input. |
| Custom domain option | Toggle: "Use a custom domain instead". Reveals a second input for full domain (e.g., `tenders.acme.com`). Shows CNAME record instructions: "Point `tenders.acme.com` CNAME to `proxy.eusolicit.com`". |
| Navigation | "Next" button (disabled until subdomain is valid and available) |

### Step 2: Branding

| Element | Specification |
|---------|---------------|
| Logo upload | Drag-and-drop zone or click to browse. Accepts SVG, PNG, JPEG. Max 2MB. Shows preview at sidebar scale (32px height) and login page scale (120px height). Two variants requested: icon mark (for collapsed sidebar) and full logo (for expanded sidebar + login). |
| Primary color | Color picker (HSL sliders + hex input). Default: `--color-primary-600`. Generates a full 50–900 palette automatically from the selected hue. Preview swatches shown. |
| Accent color | Same as primary color picker. Used for secondary actions, highlights. |
| Preview | Right side of the screen: live preview panel showing the login page and a mock dashboard with the selected branding applied. Switches between "Login Page", "Dashboard", and "Sidebar" preview states. |
| Font | Dropdown: Inter (default), system font, or custom (upload). Custom fonts must be WOFF2 format. |
| Navigation | "Back" and "Next" buttons |

### Step 3: Email Configuration

| Element | Specification |
|---------|---------------|
| Sender domain input | Text input for the domain portion of the sender email (e.g., `acme.com`). Sender address format: `noreply@[domain]` or custom prefix input. |
| DNS records table | After domain is entered, display a table of required DNS records: |

| Record Type | Host | Value | Purpose | Status |
|-------------|------|-------|---------|--------|
| TXT | `_dkim._domainkey.acme.com` | `v=DKIM1; k=rsa; p=MIGf...` (generated) | DKIM signing | Pending / Verified |
| TXT | `acme.com` | `v=spf1 include:spf.eusolicit.com ~all` | SPF authorization | Pending / Verified |
| TXT | `_dmarc.acme.com` | `v=DMARC1; p=none; rua=mailto:dmarc@eusolicit.com` | DMARC policy | Pending / Verified |

| Element | Specification |
|---------|---------------|
| Copy buttons | Each DNS record value has a "Copy" button for easy pasting into DNS management. |
| Instructions | Expandable help text: "Add these records in your domain's DNS management panel (e.g., Cloudflare, GoDaddy, Namecheap). DNS changes can take up to 48 hours to propagate." |
| Navigation | "Back" and "Next" buttons. "Next" available even if DNS is not yet verified (verification can happen later). |

### Step 4: DNS Verification

| Element | Specification |
|---------|---------------|
| Status display | Each DNS record from Step 3 shown with a real-time verification status: |

| Status | Visual | Meaning |
|--------|--------|---------|
| Pending | Gray clock icon + "Pending" | Record not yet detected |
| Verifying | Amber spinner + "Checking..." | Verification in progress |
| Verified | Green check + "Verified" | Record correctly configured |
| Error | Red X + "Error: [reason]" | Record found but incorrect (e.g., wrong value, missing) |

| Element | Specification |
|---------|---------------|
| Auto-check | System automatically checks DNS every 60 seconds. Countdown timer shown: "Next check in 45s". |
| Manual check | "Check Now" button for immediate verification. Debounced to prevent abuse (min 10s between manual checks). |
| Progress summary | "2 of 3 records verified" with progress bar. |
| Troubleshooting | If a record fails after 3 checks: expandable troubleshooting panel with common issues (propagation delay, incorrect record type, conflicting records). |
| Navigation | "Back" and "Next" buttons. "Next" (labeled "Activate") is only enabled when all records are verified. |

### Step 5: Activation

| Element | Specification |
|---------|---------------|
| Summary | Full configuration summary: subdomain/domain, branding preview (small), email sender address, DNS status (all green). |
| Test email | "Send Test Email" button: sends a branded test email to the admin's address. |
| Activate button | Large primary button: "Go Live". Requires all DNS records to be verified. |
| Confirmation dialog | AlertDialog: "Activate white-label instance for [subdomain].eusolicit.com? Users will be able to access the platform at this address." + "Activate" / "Cancel" |
| Post-activation | Success toast + redirect to the white-label management list showing the new instance as "Active". |

**Preview Mode**

At any step, a "Preview" toggle in the header allows the admin to see the current configuration rendered in a preview frame. The preview updates in real-time as settings change. Preview options: Login Page, Dashboard (mock data), Email (sample notification).

---

## 5.11 Audit Log Diff Display

The audit log (accessible to admins and organization owners) records all significant actions in the system. Each entry can be expanded to show a detailed diff of what changed.

### Audit Log List

| Column | Content |
|--------|---------|
| Timestamp | Date + time (user's local timezone). Format: "Apr 5, 2026, 14:32:05". Relative time on hover tooltip ("2 hours ago"). |
| User | Avatar + name. System actions show a robot icon + "System". |
| Action | Verb-based label: "Updated", "Created", "Deleted", "Assigned", "Approved", "Rejected", "Exported", "Logged in", "Upgraded", "Downgraded". Color-coded: green (create), blue (update), red (delete), amber (system/security). |
| Entity | Entity type + name/identifier. E.g., "Proposal: Technical Offer for TED-2026-12345". Linked to the entity (navigates on click). |
| Summary | One-line summary of the change. E.g., "Changed status from Draft to Under Review". Truncated if long. |
| Expand | Chevron icon on right. Click to expand and show the diff. |

### Diff Display Patterns

Three diff display modes are used depending on the data type:

**Structured Field Changes (Key-Value Pairs)**

Used for: profile updates, opportunity metadata changes, subscription changes, settings modifications.

```
┌──────────────────────────────────────────────────────────────┐
│  Field              Before                After              │
│  ─────────────────  ──────────────────    ──────────────     │
│  Company Name       Acme Ltd              Acme Corp Ltd      │
│  Country            Bulgaria              Bulgaria (no chg)  │
│  VAT Number         BG123456789           BG987654321         │
│  Sector             (not set)             IT Services         │
└──────────────────────────────────────────────────────────────┘
```

| Property | Specification |
|----------|---------------|
| Layout | Three-column table: Field Name, Before, After |
| Changed fields | Highlighted row background (`--color-warning-bg` very subtle). Before value has a faint red background; After value has a faint green background. |
| Unchanged fields | Not shown by default. "Show all fields" toggle at bottom includes unchanged fields in muted text. |
| Null → value | Before column shows "(not set)" in italic muted text. |
| Value → null | After column shows "(removed)" in italic red text. |

**JSON Diff (Structured Objects)**

Used for: compliance framework assignment, API configuration changes, complex settings objects.

| Property | Specification |
|----------|---------------|
| Layout | Side-by-side panels: "Before" (left) and "After" (right). On mobile: stacked vertically. |
| Syntax | JSON formatted with indentation. Syntax-highlighted (keys in one color, strings in another, numbers in another). |
| Diff highlighting | Added lines: green left-border + green background tint. Removed lines: red left-border + red background tint. Changed lines: amber left-border, with the specific changed value highlighted. Unchanged lines: no highlight. |
| Collapse | Long JSON objects are collapsed by default (show first 10 lines + "Show more"). Key paths that contain changes are auto-expanded. |
| Copy | "Copy Before" and "Copy After" buttons in each panel header. |

**Rich Text Diff (Proposal Content Changes)**

Used for: proposal section edits, content block modifications, description updates.

| Property | Specification |
|----------|---------------|
| Layout | Single panel showing the unified diff (not side-by-side — prose is easier to read inline). |
| Added text | Green background highlight (`--color-success-bg`), slightly darker than structural green. |
| Removed text | Red background highlight (`--color-error-bg`) with strikethrough text. |
| Unchanged text | Normal rendering. Context lines (2 lines before and after each change) shown; large unchanged sections collapsed with "[... 24 unchanged lines ...]" separator. |
| Formatting changes | If only formatting changed (e.g., bold added, heading level changed) without text change: noted as a badge "Formatting changed" with a tooltip describing the change. The text itself is shown without diff highlighting. |

### Expansion and Collapse Behavior

| Property | Specification |
|----------|---------------|
| Default state | All entries are collapsed (showing only the list row). |
| Expand | Click the chevron or the row itself to expand. Only one entry expanded at a time (accordion behavior) unless the user holds Shift while clicking (opens multiple). |
| "View changes" button | Inside the expanded area, if the diff data needs to be fetched (not preloaded): "View changes" button triggers the fetch. Skeleton shown while loading. This avoids loading diffs for entries the user never inspects. |
| Transition | Expand/collapse uses `--transition-normal` (200ms ease-in-out) height animation. |
| Deep link | Each audit log entry has a permalink (URL hash `#audit-[id]`). Navigating to this URL auto-expands the entry and scrolls it into view. |

### Filtering and Search

The audit log list supports the standard filter pattern:

| Filter | Control |
|--------|---------|
| Date range | Date range picker with presets: Today, Last 7 days, Last 30 days, Custom range |
| User | Multi-select dropdown listing all users + "System" |
| Action type | Checkbox group: Created, Updated, Deleted, Security (login/logout), Administrative (role changes, subscription changes) |
| Entity type | Checkbox group: Opportunity, Proposal, Profile, Subscription, Settings, Compliance, User |
| Search | Free text search across entity names and change summaries |

Filters are URL-synced following the same pattern as opportunity list filters.agentId: aab4b513bb07647de (use SendMessage with to: 'aab4b513bb07647de' to continue this agent)
<usage>total_tokens: 34825
tool_uses: 0
duration_ms: 701286</usage>

---

## Updated Screen Inventory

The following screens are NEW additions to the base UX specification's Screen Inventory (Section 6).

### Client App — New Screens

| Screen | Purpose | Tier Gate | Source |
|--------|---------|-----------|--------|
| Task Board | Per-opportunity Kanban task management | Pro+ | Gap #2 |
| Content Library | Reusable content blocks CRUD + search | All (limited on Free) | Gap #7 |
| ESPD Builder | European Single Procurement Document wizard | Pro+ | Gap #9 |
| Consortium Finder | Partner search for EU grants | Pro+ | Gap #3 (6.4) |
| Active Grants | Post-award grant management hub | Pro+ | Gap #8 |
| Grant Reports | Periodic report generation + tracking | Pro+ | Gap #8 |
| Grant Budget Tracking | Actual vs planned expenditure | Pro+ | Gap #8 |
| Company Profile Vault | Company credentials, CVs, certs, projects, financials CRUD | All (limited on Free) | Gap #17 |

### Client App — New Tabs on Existing Screens

| Screen | New Tab/Panel | Purpose | Source |
|--------|--------------|---------|--------|
| Opportunity Detail | Risk Analysis tab | Clause risk flagging display | Gap #5 |
| Opportunity Detail | Requirements tab | Auto-generated trackable checklist | Gap #6 |
| Opportunity Detail | Documents tab | Document management (browse, delete, retention) | Gap #21 |
| Opportunity Detail | Timeline tab | Procurement lifecycle dates visualization | Gap #37 |
| Proposal Editor | Pricing Assistant (side panel) | Competitive pricing suggestions | Gap #4 |
| Proposal Editor | Approval Pipeline (top bar) | Configurable review stages | Gap #3 |
| Proposal Editor | Real-Time Collaboration | Presence, cursors, section locks, activity feed | Gap #24 |

### Analytics Hub — Sub-Views (replacing single "Analytics dashboard" screen)

| View | Purpose | Tier Gate |
|------|---------|-----------|
| Market Intelligence | Sector analytics, procurement volumes, trends | Pro+ |
| ROI Tracker | Bid investment vs contract value analysis | Pro+ |
| Team Performance | Per-team-member productivity metrics | Pro+ (bid managers+) |
| Competitor Intelligence | Competitor patterns, win rates, pricing | Pro+ / Enterprise |
| Pipeline Forecast | Predicted upcoming opportunities | Pro+ |
| Usage Analytics | Billing period usage vs tier limits | All |
| Custom Reports | Report builder with export and scheduling | Pro+ |

### Admin App — New Screens

| Screen | Purpose | Source |
|--------|---------|--------|
| Regulation Tracker | Regulatory change feed + monitored regulations | Gap #10 |

**Updated totals**: 5 public + 28 client app + 7 analytics views + 11 admin = **51 screens/views** (up from 35)

---

## Updated UX Requirements Registry

New UX requirements added by this supplement, cross-referenced to gap analysis.

### New Screen Requirements

| ID | Requirement | Screen | Gap # |
|----|-------------|--------|-------|
| UX-P09 | Task board with Kanban + list views, drag-and-drop, auto-generated task templates | Task Board | #2 |
| UX-P10 | Horizontal stepper approval pipeline with stage configuration and reviewer assignment | Approval Pipeline | #3 |
| UX-P11 | Pricing assistant side panel with market benchmarks, AI suggestion, and price positioning chart | Pricing Assistant | #4 |
| UX-P12 | Clause risk panel with severity badges, extracted text, AI explanation, and status tracking | Risk Analysis | #5 |
| UX-P13 | Requirements checklist with category tags, status tracking, assignee, and proposal section linking | Requirements Checklist | #6 |
| UX-P14 | Content blocks library with CRUD, categories, approval workflow, and proposal editor integration | Content Library | #7 |
| UX-P15 | ESPD wizard with auto-fill from company profile, field mapping, validation, and XML/PDF export | ESPD Builder | #9 |
| UX-P16 | Consortium finder with search, complementarity scoring, consortium workspace, and partner profiles | Consortium Finder | #3 (6.4) |
| UX-P17 | Post-award grant management with report generation, budget tracking, and deliverable tracking | Active Grants + Reports | #8 |
| UX-A07 | Regulation tracker feed with change diffs, severity badges, framework impact analysis, and alert creation | Regulation Tracker | #10 |
| UX-PV01–PV07 | Company Profile Vault with completeness indicator, team CVs with drag-reorder, cert expiry tracking, past project tagging, financial thresholds, auto-save, bid-outcome-to-project flow | Company Profile Vault | #17 |

### Component Addition Requirements

| ID | Requirement | Host Screen | Gap # |
|----|-------------|-------------|-------|
| UX-C01 | Relevance score badge (0-100) with color-coded ring and match factor tooltip | Opportunity List + Detail | #22 |
| UX-C02 | Template selector dialog with grid, sector filter, and past-proposal option | Proposal Generation flow | #13 |
| UX-C03 | Language selector in header (UI language) and AI operation dialogs (output language) | Global header + AI dialogs | #14 |
| UX-C04 | Document manager tab with file list, drag-drop upload, virus scan status, retention indicator | Opportunity Detail | #21 |
| UX-C05 | Per-tender permissions panel with role overrides and member access control | Opportunity Settings | #18 |
| UX-C06 | Usage meter widget in header bar (compact) and subscription settings (detailed) | Global header + Settings | #25 |
| UX-C07 | Self-service submission wizard with step-by-step portal instructions and confirmation | Post-export flow | #23 |
| UX-C08 | Per-bid add-on one-click purchase modal with Stripe integration | Feature gates | #20 |
| UX-C09 | Near-limit usage warnings at 70%, 90%, and 100% thresholds | All metered features | #25 |
| UX-C10 | Parsed document structured results (criteria, required docs, timeline) on opportunity detail | Opportunity Detail | #37 |
| UX-CO01–CO07 | Real-time collaborative editing with presence avatars, colored cursors, section advisory locks, role-based section access, offline resilience, mobile-adapted collaboration, activity feed | Proposal Editor | #24 |

### Analytics Requirements

| ID | Requirement | View | Gap # |
|----|-------------|------|-------|
| UX-AN01 | Analytics shell with tabbed navigation, global date range picker, and tier-gated access | Analytics Hub | #19 |
| UX-AN02 | Market intelligence view with sector volume, trend, and authority charts | Market Intelligence | #19 |
| UX-AN03 | ROI tracker with per-bid cost/value analysis and aggregate KPIs | ROI Tracker | #19 |
| UX-AN04 | Team performance table with win rates, preparation times, and drill-down | Team Performance | #19 |
| UX-AN05 | Competitor intelligence with search, cards, overlap analysis, and watchlist | Competitor Intelligence | #11 |
| UX-AN06 | Pipeline forecasting with Gantt-style timeline and confidence indicators | Pipeline Forecast | #12 |
| UX-AN07 | Usage analytics with progress bars, history charts, and projected limit dates | Usage Analytics | #25 |
| UX-AN08 | Custom report builder with template selection, metric configuration, and scheduled delivery | Custom Reports | #19 |

### Cross-Cutting Pattern Requirements

| ID | Requirement | Pattern | Gap # |
|----|-------------|---------|-------|
| UX-X01 | Empty states with illustration, headline, description, and primary CTA for all major screens | Empty States | #27 |
| UX-X02 | Consistent AI operation states: idle → requesting → streaming/processing → complete → error | AI Operations | #26 |
| UX-X03 | In-app notification center with bell icon, unread count, typed notifications, and click-to-navigate | Notifications | #28 |
| UX-X04 | Global search (Cmd+K) with categories and opportunity filter panel with saved presets | Search & Filter | #31 |
| UX-X05 | Mobile-first responsive adaptations: bottom nav, card lists, full-screen sheets, simplified charts | Mobile | #29 |
| UX-X06 | Trial countdown banner, expiry interstitial, and post-trial read-only state | Trial Experience | #36 |
| UX-X07 | Downgrade confirmation with feature comparison, data impact summary, and 7-day grace period | Downgrade Experience | #36 |
| UX-X08 | Data source attribution badges (AOP/TED/EU Grant) on opportunity cards and detail | Source Attribution | #38 |
| UX-X09 | White-label DNS provisioning wizard with preview mode and verification status | White-Label Setup | #35 |
| UX-X10 | Audit log diff display with JSON diff and inline text diff for before/after values | Audit Log | #33 |
| UX-X11 | WCAG 2.1 AA accessibility: keyboard navigation, screen reader labels, focus management, contrast | Accessibility | #30 |

---

*Document version: 1.1 | Date: 2026-04-05 | Status: Draft*
*v1.1: Added Screen 11 (Company Profile Vault) and Section 3.11 (Real-Time Collaborative Editing)*
*Companion to: EU_Solicit_User_Journeys_and_Workflows_v1.md*
