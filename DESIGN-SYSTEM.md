# Design System — Passgrad Platform

Reference design system for all Passgrad apps. Source of truth is the existing WMS (`passgrad-workflow/src/index.css`). All new apps should adopt the same tokens and conventions.

---

## Font

- **Family**: Inter (Google Fonts)
- **Base tracking**: `letter-spacing: -0.011em`
- **OpenType features**: `"cv02", "cv03", "cv04", "cv11"` (Inter alternates — cleaner numerals and punctuation)
- **Anti-aliasing**: `antialiased`

---

## Color Tokens (OKLCH)

All colors defined as CSS custom properties. Tailwind v4 maps them via `@theme inline`.

### Light Mode (`:root`)

| Token                      | Value                    | Role                               |
| -------------------------- | ------------------------ | ---------------------------------- |
| `--background`             | `oklch(0.991 0 0)`       | Page background — near white       |
| `--foreground`             | `oklch(0.279 0.03 260)`  | Body text — blue-tinted dark       |
| `--card`                   | `oklch(1 0 0)`           | Card surface                       |
| `--card-foreground`        | `oklch(0.279 0.03 260)`  | Card text                          |
| `--popover`                | `oklch(1 0 0)`           | Popover/dropdown surface           |
| `--popover-foreground`     | `oklch(0.279 0.03 260)`  | Popover text                       |
| `--primary`                | `oklch(0.623 0.188 259)` | Brand blue — buttons, links, rings |
| `--primary-foreground`     | `oklch(1 0 0)`           | Text on primary                    |
| `--secondary`              | `oklch(0.969 0.003 260)` | Subtle background — ghost buttons  |
| `--secondary-foreground`   | `oklch(0.279 0.03 260)`  | Text on secondary                  |
| `--muted`                  | `oklch(0.969 0.003 260)` | Disabled / muted surface           |
| `--muted-foreground`       | `oklch(0.557 0.04 258)`  | Muted text — labels, placeholders  |
| `--accent`                 | `oklch(0.969 0.003 260)` | Hover highlight surface            |
| `--accent-foreground`      | `oklch(0.279 0.03 260)`  | Text on accent                     |
| `--destructive`            | `oklch(0.636 0.209 25)`  | Error / delete red                 |
| `--destructive-foreground` | `oklch(1 0 0)`           | Text on destructive                |
| `--border`                 | `oklch(0.926 0.014 254)` | Dividers, input borders            |
| `--input`                  | `oklch(0.926 0.014 254)` | Input field border                 |
| `--ring`                   | `oklch(0.623 0.188 259)` | Focus ring — matches primary       |
| `--radius`                 | `0.375rem`               | Base border radius                 |

#### Sidebar (light)

| Token                          | Value                                 |
| ------------------------------ | ------------------------------------- |
| `--sidebar`                    | `oklch(1 0 0)` — white                |
| `--sidebar-foreground`         | `oklch(0.557 0.04 258)` — muted       |
| `--sidebar-primary`            | `oklch(0.623 0.188 259)` — brand blue |
| `--sidebar-primary-foreground` | `oklch(1 0 0)`                        |
| `--sidebar-accent`             | `oklch(0.971 0.013 265)` — hover      |
| `--sidebar-accent-foreground`  | `oklch(0.623 0.188 259)`              |
| `--sidebar-border`             | `oklch(0.926 0.014 254)`              |
| `--sidebar-ring`               | `oklch(0.623 0.188 259)`              |

#### Semantic Colors (light, same in dark)

| Token                  | Value                        | Usage                     |
| ---------------------- | ---------------------------- | ------------------------- |
| `--success`            | `oklch(0.704 0.191 142.216)` | Green — confirmed, done   |
| `--success-foreground` | `oklch(0.985 0 0)`           |                           |
| `--warning`            | `oklch(0.95 0.18 80)`        | Yellow — pending, caution |
| `--warning-foreground` | `oklch(0.205 0 0)`           |                           |

#### Status Colors (light)

| Token                | Value                    | Usage  |
| -------------------- | ------------------------ | ------ |
| `--status-confirmed` | `oklch(0.588 0.156 152)` | Green  |
| `--status-draft`     | `oklch(0.623 0.188 259)` | Blue   |
| `--status-pending`   | `oklch(0.779 0.166 82)`  | Yellow |
| `--status-cancelled` | `oklch(0.636 0.209 25)`  | Red    |

#### Worktable Grid Tokens (light)

| Token                   | Value                    | Usage                    |
| ----------------------- | ------------------------ | ------------------------ |
| `--worktable-header`    | `oklch(1 0 0)`           | Column header background |
| `--worktable-row-hover` | `oklch(0.985 0.002 260)` | Row hover state          |
| `--worktable-selected`  | `oklch(0.971 0.013 265)` | Selected row             |
| `--worktable-grid-line` | `oklch(0.945 0.01 254)`  | Cell border lines        |

---

### Dark Mode (`.dark`)

| Token                          | Value                           |
| ------------------------------ | ------------------------------- |
| `--background`                 | `oklch(0.145 0 0)`              |
| `--foreground`                 | `oklch(0.985 0 0)`              |
| `--card`                       | `oklch(0.205 0 0)`              |
| `--card-foreground`            | `oklch(0.985 0 0)`              |
| `--popover`                    | `oklch(0.205 0 0)`              |
| `--popover-foreground`         | `oklch(0.985 0 0)`              |
| `--primary`                    | `oklch(0.922 0 0)`              |
| `--primary-foreground`         | `oklch(0.205 0 0)`              |
| `--secondary`                  | `oklch(0.269 0 0)`              |
| `--secondary-foreground`       | `oklch(0.985 0 0)`              |
| `--muted`                      | `oklch(0.269 0 0)`              |
| `--muted-foreground`           | `oklch(0.708 0 0)`              |
| `--accent`                     | `oklch(0.269 0 0)`              |
| `--accent-foreground`          | `oklch(0.985 0 0)`              |
| `--destructive`                | `oklch(0.704 0.191 22.216)`     |
| `--destructive-foreground`     | `oklch(0.985 0 0)`              |
| `--border`                     | `oklch(1 0 0 / 10%)`            |
| `--input`                      | `oklch(1 0 0 / 15%)`            |
| `--ring`                       | `oklch(0.556 0 0)`              |
| `--sidebar`                    | `oklch(0.205 0 0)` — dark panel |
| `--sidebar-foreground`         | `oklch(0.985 0 0)`              |
| `--sidebar-primary`            | `oklch(0.488 0.243 264.376)`    |
| `--sidebar-primary-foreground` | `oklch(0.985 0 0)`              |
| `--sidebar-accent`             | `oklch(0.269 0 0)`              |
| `--sidebar-accent-foreground`  | `oklch(0.985 0 0)`              |
| `--sidebar-border`             | `oklch(1 0 0 / 10%)`            |
| `--sidebar-ring`               | `oklch(0.556 0 0)`              |

---

## Border Radius Scale

| Token             | Value                                    |
| ----------------- | ---------------------------------------- |
| `--radius` (base) | `0.375rem`                               |
| `--radius-sm`     | `calc(var(--radius) - 4px)` ≈ `0.125rem` |
| `--radius-md`     | `calc(var(--radius) - 2px)` ≈ `0.25rem`  |
| `--radius-lg`     | `var(--radius)` = `0.375rem`             |
| `--radius-xl`     | `calc(var(--radius) + 4px)` ≈ `0.625rem` |

---

## Misc Utility Colors

| Token             | Value     | Usage                  |
| ----------------- | --------- | ---------------------- |
| `--color-pblue`   | `#4abad3` | Passgrad cyan accent   |
| `--color-porange` | `#e69100` | Passgrad orange accent |

---

## Chart Colors

| Token       | Color                               |
| ----------- | ----------------------------------- |
| `--chart-1` | `oklch(0.488 0.243 264.376)` — blue |
| `--chart-2` | `oklch(0.696 0.17 162.48)` — teal   |
| `--chart-3` | `oklch(0.769 0.188 70.08)` — yellow |
| `--chart-4` | `oklch(0.627 0.265 303.9)` — purple |
| `--chart-5` | `oklch(0.645 0.246 16.439)` — red   |

---

## Animations

| Token                       | Definition                      |
| --------------------------- | ------------------------------- |
| `--animate-accordion-down`  | `accordion-down 0.2s ease-out`  |
| `--animate-accordion-up`    | `accordion-up 0.2s ease-out`    |
| `--animate-slide-in-right`  | `slide-in-right 0.2s ease-out`  |
| `--animate-slide-out-right` | `slide-out-right 0.2s ease-out` |
| `--animate-fade-in`         | `fade-in 0.2s ease-out`         |

---

## Scrollbar

Custom thin scrollbar utility class `.scrollbar-thin`:

- `scrollbar-width: thin`
- Thumb color: `var(--border)`
- Width/height: `6px`
- Track: transparent

---

## Stack

| Layer              | Choice                                            |
| ------------------ | ------------------------------------------------- |
| Component library  | shadcn/ui                                         |
| Styling            | Tailwind CSS v4 (`@tailwindcss/vite`)             |
| Color format       | OKLCH throughout                                  |
| Icons              | Lucide React                                      |
| Font               | Inter (Google Fonts)                              |
| Dark mode strategy | Class-based (`.dark` on `<html>`)                 |
| Utility helper     | `cn()` from `@/lib/utils` (clsx + tailwind-merge) |

---

## Conventions

- All colors **must** be OKLCH.
- No inline styles — Tailwind utility classes only.
- Custom CSS goes in `src/index.css` only.
- Use `cn()` for conditional class names.
- shadcn/ui primitives from `@/components/ui/` first before building custom components.
