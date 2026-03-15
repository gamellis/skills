# Tailwind CSS v4 & Frontend

Read this when setting up Tailwind in a Rails app, defining design tokens, creating component utilities, styling dark mode, or organizing CSS.

## Table of Contents
- [Setup](#setup)
- [CSS-First Configuration](#css-first-configuration)
- [@theme — Design Tokens](#theme--design-tokens)
- [@utility — Component Classes](#utility--component-classes)
- [Dark Mode](#dark-mode)
- [tailwind_merge and cn() Helper](#tailwind_merge-and-cn-helper)
- [Asset Pipeline Integration](#asset-pipeline-integration)
- [Turbo + Stimulus Compatibility](#turbo--stimulus-compatibility)

## Setup

Gemfile:

```ruby
gem "tailwindcss-rails", "~> 4.4"
gem "tailwind_merge"
```

Install and set up:

```bash
./bin/rails tailwindcss:install
```

Procfile.dev entry for development:

```
css: bin/rails tailwindcss:watch
```

Entry point: `app/assets/tailwind/application.css`

## CSS-First Configuration

Tailwind v4 moves all configuration into CSS — no `tailwind.config.js`, no JavaScript config file:

```css
@import "tailwindcss";
```

That single import activates automatic content scanning (no need to register template paths) and loads the full utility class library. All customization happens via CSS directives: `@theme` for design tokens, `@utility` for component classes, `@variant` for states.

Key difference from v3: there is no JavaScript config file. If you see `tailwind.config.js`, you're looking at v3 patterns.

## @theme — Design Tokens

Define design tokens with `@theme` — these become Tailwind utility classes automatically:

```css
@theme {
  --font-sans: InterVariable, sans-serif;
  --font-sans--font-feature-settings: 'cv02', 'cv03', 'cv04', 'cv11';
  --spacing: 0.25rem;
  --z-1: 1;
}
```

Theme variables follow Tailwind's naming convention — `--font-sans` creates `font-sans`, `--spacing` sets the base spacing scale. Any `--color-*` variable creates color utilities.

## @utility — Component Classes

Custom `@utility` classes are defined in `app/assets/tailwind/application.css` — check there for available classes like `btn-primary`, `text-field`, `checkbox`, etc. before creating new ones. Reusable HTML structures that involve multiple elements (e.g. checkbox with SVG overlay) live as partials in `app/views/shared/`.

Create reusable component classes with `@utility` — these are real utility classes that work with variants, `@apply`, and `tailwind_merge`:

```css
@utility btn {
  @apply inline-flex items-center gap-x-1 rounded-md px-3 py-2 text-sm
         font-medium shadow-sm transition-colors whitespace-nowrap;

  @variant disabled {
    @apply disabled:opacity-50 disabled:cursor-not-allowed;
  }

  @variant focus-visible {
    @apply outline-2 outline-offset-2;
  }
}

@utility btn-primary {
  @apply btn bg-blue-600 text-white;

  @variant hover {
    @apply bg-blue-500;
  }

  @variant focus-visible {
    @apply outline-blue-600;
  }

  @variant dark {
    @apply bg-blue-500;

    @variant hover {
      @apply bg-blue-400;
    }
  }
}

@utility btn-secondary {
  @apply btn bg-white text-gray-900 ring-1 ring-inset ring-gray-300;

  @variant hover {
    @apply bg-gray-50;
  }

  @variant dark {
    @apply bg-gray-700 text-white ring-gray-600;

    @variant hover {
      @apply bg-gray-600;
    }
  }
}
```

`@utility` creates classes that participate in the Tailwind ecosystem — they merge correctly with `tailwind_merge`, and `@variant` nests state/mode handling inside the definition.

### Common @utility Patterns

```css
/* Text inputs */
@utility text-field {
  @apply block w-full rounded-md bg-white px-3 py-1.5 text-base text-gray-900
         outline-1 -outline-offset-1 outline-gray-300 placeholder:text-gray-400
         focus:outline-2 focus:-outline-offset-2 focus:outline-blue-600 sm:text-sm/6;

  @variant dark {
    @apply bg-white/5 text-white outline-white/10 placeholder:text-gray-500
           focus:outline-blue-500;
  }
}

/* Badges with color variants */
@utility badge {
  @apply inline-flex items-center rounded-md px-2 py-1 text-xs font-medium;
}

@utility badge-primary {
  @apply badge bg-blue-100 text-blue-800;

  @variant dark {
    @apply bg-blue-900/40 text-blue-300;
  }
}

/* Breakout link — makes an entire card clickable */
@utility breakout-link {
  @apply static text-inherit no-underline;

  &::before {
    content: "";
    position: absolute;
    inset: 0;
  }

  &, &::before {
    cursor: pointer;
  }
}

/* Size variants */
@utility btn-sm { @apply px-2 py-1 text-xs; }
@utility btn-lg { @apply px-3.5 py-2.5 text-sm; }
@utility btn-icon { @apply p-2; }
```

## Spacing Convention

Prefer `mt-*` (margin-top) over `mb-*` (margin-bottom) for vertical spacing between elements. This keeps the spacing responsibility on the element that follows, making components more composable — a component doesn't need to know what comes after it.

## Dark Mode

Use the `dark:` variant on utility classes. Define dark variants inside `@utility` blocks with `@variant dark`:

```html
<body class="min-h-dvh bg-white dark:bg-gray-900 dark:text-white">
```

For `@utility` components, nest dark mode handling inside the definition (see btn-primary example above). This keeps light and dark styles co-located.

Theme toggling with a Stimulus controller on `<html data-controller="theme">`:

```html
<html data-controller="theme">
```

## tailwind_merge and cn() Helper

The `cn()` helper resolves Tailwind class conflicts — later classes win, so you can safely compose and override utility classes:

```ruby
# app/helpers/application_helper.rb
def cn(*classes)
  classes = classes.map { |c| c.is_a?(String) ? c : nil }
  TailwindMerge::Merger.new.merge(classes)
end
```

Usage in views:

```erb
<div class="<%= cn("p-4 text-gray-900", active ? "bg-blue-50" : "bg-white") %>">
```

Usage in helpers that accept class overrides:

```ruby
def sidebar_link_classes(active, collapsed: false)
  cn(
    "group flex rounded-md p-2 text-sm/6 font-semibold",
    collapsed ? "justify-center" : "gap-x-3",
    active ? "bg-gray-50 text-blue-600 dark:bg-white/5 dark:text-white"
           : "text-gray-700 hover:bg-gray-50 hover:text-blue-600"
  )
end
```

## Asset Pipeline Integration

Propshaft serves the compiled CSS. The Tailwind build writes to `app/assets/builds/tailwind.css`, which Propshaft picks up automatically.

```erb
<%# app/views/layouts/application.html.erb %>
<%= stylesheet_link_tag :app, "data-turbo-track": "reload" %>
<%= javascript_importmap_tags %>
```

The `tailwindcss-rails` gem registers `:app` as a named stylesheet bundle that includes the Tailwind build output. `data-turbo-track: "reload"` forces a full page reload when stylesheets change, ensuring users always see the latest styles after deployment.

importmap-rails handles JavaScript — no bundler needed:

```ruby
# config/importmap.rb
pin "application"
pin "@hotwired/turbo-rails", to: "turbo.min.js"
pin "@hotwired/stimulus", to: "stimulus.min.js"
pin "@hotwired/stimulus-loading", to: "stimulus-loading.js"
pin_all_from "app/javascript/controllers", under: "controllers"
```

## Turbo + Stimulus Compatibility

Tailwind classes work seamlessly with Turbo and Stimulus. Key fixes:

```css
/* Prevent turbo-frame from breaking flex/grid layouts */
turbo-frame { display: contents; }

/* Fix button_to forms with hidden inputs causing height issues */
form.button_to:has(input[type="hidden"]) {
  line-height: 0;
}
```

View Transitions for smooth Turbo page navigation:

```css
::view-transition-old(root) {
  animation: 100ms ease-out fade-out, 300ms ease slide-to-left;
}
::view-transition-new(root) {
  animation: 200ms ease-in 90ms fade-in, 300ms ease slide-from-right;
}
```
