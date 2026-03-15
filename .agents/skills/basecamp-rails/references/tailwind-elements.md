# Tailwind Plus Elements

Read this when adding interactive UI components (dialogs, dropdowns, tabs, disclosures, popovers, selects) without a JavaScript framework.

## Table of Contents
- [Setup with importmap](#setup-with-importmap)
- [Available Components](#available-components)
- [Command Pattern](#command-pattern)
- [Dialog](#dialog)
- [Disclosure](#disclosure)
- [Dropdown Menu](#dropdown-menu)
- [Tabs](#tabs)
- [Select](#select)
- [Turbo Compatibility](#turbo-compatibility)

## Setup with importmap

Pin the vendored JS in importmap — no npm or node required:

```ruby
# config/importmap.rb
pin "@tailwindplus/elements", to: "@tailwindplus--elements.js"
```

Import in your application entry point:

```js
// app/javascript/application.js
import "@tailwindplus/elements"
```

The vendored file lives at `vendor/javascript/@tailwindplus--elements.js`. importmap avoids npm/node — pin the vendored JS directly and it's available everywhere.

## Available Components

9 components: autocomplete, command-palette, copy-button, dialog, disclosure, dropdown-menu, popover, select, tabs.

All are web components (`<el-*>` custom elements) that work with any server-rendered HTML — no React, Vue, or build step needed.

## Command Pattern

All components use the Invoker Commands API — standard HTML attributes wire up behavior without JavaScript:

```html
<!-- Open a dialog -->
<button command="show-modal" commandfor="my-dialog">Open</button>

<!-- Close a dialog -->
<button command="close" commandfor="my-dialog">Close</button>

<!-- Toggle a disclosure -->
<button command="--toggle" commandfor="my-section">Toggle</button>
```

Built-in commands: `show-modal`, `close`. Custom commands (prefixed with `--`): `--toggle`, `--show`, `--hide`, `--copy`.

No JavaScript wiring needed — the `command`/`commandfor` attributes handle everything declaratively.

## Dialog

Wrapper around native `<dialog>` that adds scroll locking, click-outside-to-close, and smooth transitions:

```html
<button command="show-modal" commandfor="delete-confirm" type="button">
  Delete
</button>

<el-dialog>
  <dialog id="delete-confirm" class="backdrop:bg-transparent">
    <el-dialog-backdrop
      class="fixed inset-0 bg-gray-900/80 transition duration-200 data-closed:opacity-0"
    />
    <el-dialog-panel
      class="bg-white rounded-lg p-6 transition duration-200 data-closed:scale-95 data-closed:opacity-0"
    >
      <h3>Delete this item?</h3>
      <p>This action cannot be undone.</p>
      <div class="flex gap-4 mt-4">
        <button command="close" commandfor="delete-confirm" type="button"
                class="btn-secondary">Cancel</button>
        <button type="submit" class="btn-danger">Delete</button>
      </div>
    </el-dialog-panel>
  </dialog>
</el-dialog>
```

Open: `command="show-modal"`. Close: `command="close"`, Escape key, or click outside the panel.

Transitions use data attributes: `data-closed`, `data-enter`, `data-leave`, `data-transition`.

## Disclosure

Toggle content visibility — ideal for accordion panels and expandable sections:

```html
<button command="--toggle" commandfor="details-section" type="button">
  Show details
</button>

<el-disclosure id="details-section" hidden>
  <p>Hidden content appears here.</p>
</el-disclosure>
```

Commands: `--toggle`, `--show`, `--hide`.

Add transitions with data attributes:

```html
<el-disclosure hidden
  class="transition transition-discrete duration-300 data-closed:opacity-0">
  <!-- ... -->
</el-disclosure>
```

## Dropdown Menu

Keyboard-navigable dropdown with automatic anchoring:

```html
<el-dropdown>
  <button type="button" class="btn-secondary">Options</button>
  <el-menu anchor="bottom start" popover
    class="mt-1 rounded-md bg-white shadow-lg ring-1 ring-black/5
           transition data-closed:opacity-0 data-closed:scale-95">
    <button class="block w-full px-4 py-2 text-left text-sm hover:bg-gray-100"
            type="button">Edit</button>
    <button class="block w-full px-4 py-2 text-left text-sm hover:bg-gray-100"
            type="button">Duplicate</button>
    <hr role="none" class="my-1" />
    <button class="block w-full px-4 py-2 text-left text-sm text-red-600 hover:bg-gray-100"
            type="button">Delete</button>
  </el-menu>
</el-dropdown>
```

Anchoring values: `top`, `bottom`, `left`, `right`, combined with `start`/`end` (e.g., `bottom end`). Control gap with `--anchor-gap` CSS variable.

## Tabs

Accessible tab interface with keyboard navigation:

```html
<el-tab-group>
  <el-tab-list class="flex gap-2 border-b">
    <button type="button" class="px-3 py-2 text-sm font-medium">Overview</button>
    <button type="button" class="px-3 py-2 text-sm font-medium">Details</button>
    <button type="button" class="px-3 py-2 text-sm font-medium">History</button>
  </el-tab-list>
  <el-tab-panels class="pt-4">
    <div>Overview content</div>
    <div hidden>Details content</div>
    <div hidden>History content</div>
  </el-tab-panels>
</el-tab-group>
```

The initially active tab is determined by which panel lacks the `hidden` attribute — works correctly with server-side rendering.

## Select

Fully styled replacement for native `<select>`:

```html
<el-select name="status" value="active">
  <button type="button" class="btn-secondary w-full justify-between">
    <el-selectedcontent>Active</el-selectedcontent>
    <svg class="size-4 text-gray-400"><!-- chevron --></svg>
  </button>
  <el-options popover anchor="bottom start"
    class="w-(--button-width) mt-1 rounded-md bg-white shadow-lg ring-1 ring-black/5">
    <el-option value="active" class="px-3 py-2 text-sm hover:bg-gray-100">Active</el-option>
    <el-option value="paused" class="px-3 py-2 text-sm hover:bg-gray-100">Paused</el-option>
    <el-option value="archived" class="px-3 py-2 text-sm hover:bg-gray-100">Archived</el-option>
  </el-options>
</el-select>
```

`<el-selectedcontent>` automatically displays the selected option's content. `w-(--button-width)` matches dropdown width to the trigger button.

## Turbo Compatibility

Tailwind Plus Elements work with Turbo out of the box — web components survive Turbo Drive navigation and morphing. No special configuration needed.

Custom elements are registered once on page load and automatically activate on any new elements inserted by Turbo Streams or Turbo Frames. This means you can render `<el-dialog>`, `<el-disclosure>`, etc. inside Turbo Frame responses and they work immediately.
