# Tailwind Plus Elements API Reference

Read this when working with `<el-dialog>`, `<el-disclosure>`, `<el-dropdown>`, `<el-select>`, `<el-autocomplete>`, `<el-command-palette>`, `<el-copyable>`, `<el-popover>`, or `<el-tab-group>` web components.

## Key Concepts

- All components are web components (`<el-*>` custom elements) — no framework needed
- Use the Invoker Commands API (`command`/`commandfor` attributes) to wire behavior declaratively
- Built-in commands: `show-modal`, `close`. Custom commands: `--toggle`, `--show`, `--hide`, `--copy`
- Transitions use data attributes: `data-closed`, `data-enter`, `data-leave`, `data-transition`
- Works with Turbo out of the box — custom elements survive navigation and morphing

## Dialog (`<el-dialog>`)

Wrapper around native `<dialog>` that adds scroll locking, click-outside-to-close, and smooth transitions.

### Structure

```html
<el-dialog>
  <dialog id="my-dialog" class="backdrop:bg-transparent">
    <el-dialog-backdrop class="fixed inset-0 bg-black/50 transition data-closed:opacity-0" />
    <el-dialog-panel class="bg-white rounded-lg transition data-closed:scale-95 data-closed:opacity-0">
      <!-- content -->
    </el-dialog-panel>
  </dialog>
</el-dialog>
```

### Events (on `<el-dialog>`, NOT on native `<dialog>`)

| Event    | Description |
|----------|-------------|
| `open`   | Dispatched when dialog is opened (not via `open` attribute) |
| `close`  | Dispatched when dialog is closed (not via `open` attribute) |
| `cancel` | Dispatched on ESC key or click-outside. `preventDefault()` prevents closing |

**IMPORTANT**: These events do NOT bubble (`bubbles: false`). Listen on the `<el-dialog>` element directly.

### Commands (on native `<dialog>`)

| Command      | Description |
|-------------|-------------|
| `show-modal` | Opens the dialog |
| `close`      | Closes the dialog |

### Methods (on `<el-dialog>`)

| Method   | Description |
|----------|-------------|
| `show()` | Opens the dialog |
| `hide()` | Closes the dialog. Optional `{ restoreFocus: false }` |

### Attributes (on `<el-dialog>`)

| Attribute | Description |
|-----------|-------------|
| `open`    | Boolean — set/remove to open/close the dialog |

### Opening

```html
<!-- Invoker command -->
<button command="show-modal" commandfor="my-dialog" type="button">Open</button>

<!-- Attribute -->
<el-dialog open><dialog>...</dialog></el-dialog>

<!-- JavaScript -->
document.getElementById('my-dialog-wrapper').show()
```

### Closing

```html
<!-- Invoker command -->
<button command="close" commandfor="my-dialog" type="button">Close</button>

<!-- JavaScript -->
document.getElementById('my-dialog-wrapper').hide()
```

Also closes on: ESC key, clicking outside `<el-dialog-panel>`.

## Disclosure (`<el-disclosure>`)

Toggle content visibility — accordion panels, expandable sections.

### Structure

```html
<button command="--toggle" commandfor="my-section" type="button">Toggle</button>
<el-disclosure id="my-section" hidden>
  <p>Content here</p>
</el-disclosure>
```

### Commands

`--toggle`, `--show`, `--hide`

### Methods

`show()`, `hide()`, `toggle()`

## Dropdown Menu (`<el-dropdown>`)

Keyboard-navigable dropdown with auto-anchoring.

### Structure

```html
<el-dropdown>
  <button type="button">Options</button>
  <el-menu anchor="bottom start" popover>
    <button type="button">Edit</button>
    <button type="button">Delete</button>
  </el-menu>
</el-dropdown>
```

All focusable children in `<el-menu>` are keyboard-navigable options.

### Anchor values

`top`, `bottom`, `left`, `right` — combine with `start`/`end` (e.g. `bottom end`).
Gap: `--anchor-gap` CSS variable. Offset: `--anchor-offset`.

## Tabs (`<el-tab-group>`)

Accessible tab interface with keyboard navigation.

### Structure

```html
<el-tab-group>
  <el-tab-list>
    <button type="button">Tab 1</button>
    <button type="button">Tab 2</button>
  </el-tab-list>
  <el-tab-panels>
    <div>Panel 1</div>
    <div hidden>Panel 2</div>
  </el-tab-panels>
</el-tab-group>
```

Active tab is determined by which panel lacks `hidden` — works with server-side rendering.

### Methods

`setActiveTab(index)` on `<el-tab-group>`.

## Select (`<el-select>`)

Styled replacement for native `<select>`.

### Structure

```html
<el-select name="status" value="active">
  <button type="button">
    <el-selectedcontent>Active</el-selectedcontent>
  </button>
  <el-options popover anchor="bottom start">
    <el-option value="active">Active</el-option>
    <el-option value="inactive">Inactive</el-option>
  </el-options>
</el-select>
```

`<el-selectedcontent>` auto-displays selected option. `w-(--button-width)` matches dropdown to button width.

### Events

`input`, `change` on `<el-select>`.

## Autocomplete (`<el-autocomplete>`)

Text input with filtered suggestions — like `<datalist>` with full styling control.

### Structure

```html
<el-autocomplete>
  <input name="user" />
  <button type="button"><svg>...</svg></button>
  <el-options popover anchor="bottom start">
    <el-option value="Wade">Wade Cooper</el-option>
    <el-option value="Tom">Tom Cooper</el-option>
  </el-options>
</el-autocomplete>
```

## Command Palette (`<el-command-palette>`)

Keyboard-friendly search/select, typically inside a dialog for `Cmd+K`.

### Structure

```html
<el-dialog>
  <dialog>
    <el-command-palette>
      <input autofocus placeholder="Search..." />
      <el-command-list>
        <button hidden type="button">Option 1</button>
        <button hidden type="button">Option 2</button>
      </el-command-list>
      <el-no-results hidden>No results found.</el-no-results>
    </el-command-palette>
  </dialog>
</el-dialog>
```

### Methods

`setFilterCallback(cb)` — custom filter. Callback receives `{ query, node, content }`, returns boolean.
`reset()` — resets to initial state.

## Copy Button (`<el-copyable>`)

### Structure

```html
<el-copyable id="snippet">npm install @tailwindplus/elements</el-copyable>
<button command="--copy" commandfor="snippet">
  <span class="in-data-copied:hidden">Copy</span>
  <span class="not-in-data-copied:hidden">Copied!</span>
</button>
```

`data-copied` present for 2s after successful copy. `data-error` on failure.

## Popover (`<el-popover>`)

Floating panels for nav menus and flyouts.

### Structure

```html
<button popovertarget="my-popover" type="button">Menu</button>
<el-popover id="my-popover" anchor="bottom start" popover>
  Content here
</el-popover>
```

### Grouping

```html
<el-popover-group>
  <button popovertarget="a">A</button>
  <el-popover id="a" popover>...</el-popover>
  <button popovertarget="b">B</button>
  <el-popover id="b" popover>...</el-popover>
</el-popover-group>
```

## Detecting Ready State

```js
if (customElements.get('el-autocomplete')) {
  myFunction()
} else {
  window.addEventListener('elements:ready', myFunction)
}
```

## Shared Patterns

### Anchor positioning (options, menus, popovers)

```html
<el-options popover anchor="bottom start" class="[--anchor-gap:4px]">
```

### Transitions

```html
<el-options class="transition transition-discrete data-closed:opacity-0
  data-enter:duration-75 data-enter:ease-out data-leave:duration-100 data-leave:ease-in">
```

### Width matching

`w-(--button-width)` or `w-(--input-width)` on dropdowns to match trigger width.
