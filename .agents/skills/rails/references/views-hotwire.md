# Views & Hotwire

Read this when building interactive UI, adding real-time updates to a page, writing Stimulus controllers, or choosing between Turbo Streams and Turbo Frames.

## Table of Contents
- [No JavaScript Framework](#no-javascript-framework)
- [Turbo Streams](#turbo-streams)
- [Custom Turbo Stream Actions](#custom-turbo-stream-actions)
- [Dialog Patterns](#dialog-patterns)
- [Stimulus Controllers](#stimulus-controllers)
- [Stimulus Patterns](#stimulus-patterns)
- [Fragment Caching](#fragment-caching)
- [JavaScript Organization](#javascript-organization)

## No JavaScript Framework

Build the entire frontend with Turbo + Stimulus + ActionCable. No React, Vue, or SPA frameworks.

**When choosing Turbo Streams vs Stimulus**: Use Turbo Streams when the server decides what changes (broadcasts, form responses). Use Stimulus when the client decides (toggling UI state, handling keyboard shortcuts, managing focus).

The server renders HTML. Turbo handles navigation, form submissions, and real-time updates. Stimulus adds lightweight JavaScript behavior.

## Turbo Streams

### Broadcasting from Models

```ruby
# app/models/card/broadcastable.rb
module Card::Broadcastable
  extend ActiveSupport::Concern

  included do
    broadcasts_refreshes  # auto Turbo Morph on model changes
  end
end
```

For explicit broadcasts:

```ruby
# Append/prepend/replace/remove
broadcast_append_to room, :messages, target: [room, :messages]
broadcast_remove_to room, :messages
broadcast_prepend_later_to user, :notifications, target: "notifications"  # _later = via job
```

### Turbo Stream Responses

```erb
<%# app/views/leafables/create.turbo_stream.erb %>
<%= turbo_stream.append @book, partial: "books/leaf", locals: { leaf: @leaf } %>
```

### Subscribing in Views

```erb
<%= turbo_stream_from Current.user, :notifications %>
<%= turbo_stream_from @board %>
```

## Custom Turbo Stream Actions

When built-in actions (append, replace, remove) aren't enough:

**Server helper:**
```ruby
# app/helpers/turbo_stream_actions_helper.rb
module TurboStreamActionsHelper
  def scroll_into_view(id, animation: nil)
    turbo_stream_action_tag :scroll_into_view, target: id, animation: animation
  end
end
Turbo::Streams::TagBuilder.prepend TurboStreamActionsHelper
```

**Client handler:**
```javascript
// app/javascript/actions/scroll_into_view.js
import { Turbo } from "@hotwired/turbo-rails"

Turbo.StreamActions.scroll_into_view = function() {
  const element = this.targetElements[0]
  element.scrollIntoView({ behavior: "smooth", block: "center" })
}
```

## Stimulus Controllers

Keep controllers small and single-purpose. Most are 20-50 lines.

### Micro-Controllers (< 10 lines)

```javascript
// auto_submit_controller.js — submit form on connect
export default class extends Controller {
  connect() { this.element.requestSubmit() }
}

// element_removal_controller.js
export default class extends Controller {
  remove() { this.element.remove() }
}

// toggle_class_controller.js
export default class extends Controller {
  static classes = ["toggle"]
  toggle() { this.element.classList.toggle(this.toggleClass) }
}
```

### Lazy Loading with IntersectionObserver

```javascript
// fetch_on_visible_controller.js
import { Controller } from "@hotwired/stimulus"
import { get } from "@rails/request.js"

export default class extends Controller {
  static values = { url: String }

  connect() {
    new IntersectionObserver((entries) => {
      if (entries.find(e => e.isIntersecting)) {
        get(this.urlValue, { responseKind: "turbo-stream" })
      }
    }).observe(this.element)
  }
}
```

### Autosave

```javascript
// autosave_controller.js
const AUTOSAVE_INTERVAL = 3000

export default class extends Controller {
  static classes = ["clean", "dirty", "saving"]
  #timer

  disconnect() { this.submit() }

  async submit() {
    if (this.#dirty) await this.#save()
  }

  change(event) {
    if (event.target.form === this.element && !this.#dirty) {
      this.#timer = setTimeout(() => this.submit(), AUTOSAVE_INTERVAL)
      this.#updateAppearance()
    }
  }

  get #dirty() { return !!this.#timer }
}
```

### Dialog with Native `<dialog>`

```javascript
export default class extends Controller {
  static targets = ["dialog"]
  static values = { modal: { type: Boolean, default: false } }

  open() {
    this.modalValue ? this.dialogTarget.showModal() : this.dialogTarget.show()
  }

  close() { this.dialogTarget.close() }
}
```

## Dialog Patterns

### Inline Dialogs (Default)

For any confirmation, info, or interactive dialog, build a small purpose-built partial. Use `<el-dialog>` directly with hardcoded styling — no helper, no config hash:

```erb
<%# app/views/projects/_delete.html.erb %>
<button type="button" command="show-modal" commandfor="delete-<%= project.id %>" class="btn-danger">
  Delete
</button>

<el-dialog>
  <dialog id="delete-<%= project.id %>" aria-labelledby="delete-<%= project.id %>-title"
          data-controller="focus-trap"
          class="fixed inset-0 size-auto max-h-none max-w-none overflow-y-auto bg-transparent backdrop:bg-transparent">
    <el-dialog-backdrop class="fixed inset-0 bg-gray-500/75 transition-opacity data-closed:opacity-0 data-enter:duration-300 data-enter:ease-out data-leave:duration-200 data-leave:ease-in dark:bg-gray-900/50"></el-dialog-backdrop>
    <div class="flex min-h-full items-end justify-center p-4 text-center sm:items-center sm:p-0">
      <el-dialog-panel class="relative transform overflow-hidden rounded-lg bg-white px-4 pt-5 pb-4 text-left shadow-xl transition-all data-closed:translate-y-4 data-closed:opacity-0 data-enter:duration-300 data-enter:ease-out data-leave:duration-200 data-leave:ease-in sm:my-8 sm:w-full sm:max-w-lg sm:p-6 data-closed:sm:translate-y-0 data-closed:sm:scale-95 dark:bg-gray-800 dark:outline dark:-outline-offset-1 dark:outline-white/10">
        <div class="sm:flex sm:items-start">
          <div class="mx-auto flex size-12 shrink-0 items-center justify-center rounded-full bg-red-100 dark:bg-red-500/10 sm:mx-0 sm:size-10">
            <%= heroicon "x-mark", variant: :outline, options: { class: "size-7 text-red-600 dark:text-red-400" } %>
          </div>
          <div class="mt-3 text-center sm:mt-0 sm:ml-4 sm:text-left">
            <h3 id="delete-<%= project.id %>-title" class="text-base font-semibold text-gray-900 dark:text-white">Delete project</h3>
            <div class="mt-2">
              <p class="text-sm text-gray-500 dark:text-gray-400">
                Are you sure you want to delete "<%= project.name %>"? This cannot be undone.
              </p>
            </div>
          </div>
        </div>
        <div class="mt-5 sm:mt-4 sm:flex sm:flex-row-reverse">
          <%= button_to "Delete", project_path(project), method: :delete,
                class: "btn-danger w-full justify-center sm:ml-3 sm:w-auto",
                form: { data: { turbo_confirm: nil } } %>
          <button type="button" autofocus command="close" commandfor="delete-<%= project.id %>"
                  class="btn-secondary mt-3 w-full justify-center font-semibold sm:mt-0 sm:w-auto">Cancel</button>
        </div>
      </el-dialog-panel>
    </div>
  </dialog>
</el-dialog>
```

This is the go-to pattern. Each dialog is self-contained — easy to understand, modify, and delete.

### Turbo Confirm (Convenience)

A global danger-styled confirm dialog exists for bulk-wiring destructive actions without building individual dialogs. Any link or form with `data-turbo-confirm` triggers it:

```erb
<%= link_to "Delete", project_path(@project),
      data: { turbo_method: :delete, turbo_confirm: "This project will be permanently deleted.",
              confirm_title: "Delete project", confirm_label: "Delete" } %>
```

Supported `data-confirm-*` attributes: `title`, `label`, `resource`. The dialog auto-detects the HTTP method and adjusts the confirm button label (e.g., "Delete" for DELETE requests).

Implementation: `app/helpers/alert_dialog_helper.rb`, `app/views/shared/_alert_dialog.html.erb`, `app/javascript/controllers/alert_dialog_controller.js`.

## Stimulus Patterns

### Use `targetConnected` for Progressive Enhancement

Format elements as they appear in the DOM, regardless of how they got there:

```javascript
messageTargetConnected(target) {
  this.#formatter.format(target)
}
```

### Delegate to JS Model Classes

Keep controllers thin. Extract complex logic into `app/javascript/models/`:

```javascript
import MessageFormatter from "models/message_formatter"
import ScrollManager from "models/scroll_manager"

export default class extends Controller {
  initialize() {
    this.#formatter = new MessageFormatter(Current.user.id, { ... })
  }
  connect() {
    this.#scrollManager = new ScrollManager(this.messagesTarget)
  }
}
```

### Use Outlets for Controller Communication

```javascript
// edit_mode_controller.js
export default class extends Controller {
  static outlets = ["autosave"]

  async change({ target: { checked } }) {
    if (!checked) {
      for (const autosave of this.autosaveOutlets) {
        await autosave.submit()
      }
    }
  }
}
```

### Private Fields and Modern JS

```javascript
#timer
#scrollManager

#sendBeacon() { ... }
get #dirty() { return !!this.#timer }

// Arrow fields for stable `this` in callbacks
#intersectionChange = (entries) => { ... }
```

### Shared Helper Modules

```javascript
// app/javascript/helpers/timing_helpers.js
export function throttle(fn, delay = 1000) { ... }
export function debounce(fn, delay = 1000) { ... }
export function nextFrame() { return new Promise(requestAnimationFrame) }
export function delay(ms) { return new Promise(resolve => setTimeout(resolve, ms)) }
```

### Prefer Form Helpers Over Raw HTML

**Default to Rails form helpers** (`f.check_box`, `f.radio_button`, `f.text_field`, etc.) instead of raw `<input>` tags. Form helpers automatically handle name/id generation, boolean attributes, hidden fields, and checked/selected state from the model — no manual wiring needed:

```erb
<%# BAD — raw HTML with manual name, hidden field, and conditional checked %>
<input type="hidden" name="user[remember_me]" value="0">
<input type="checkbox" name="user[remember_me]" value="1" <%= "checked" if checked %> class="checkbox">

<%# GOOD — form helper handles everything %>
<%= form.check_box :remember_me, class: "checkbox" %>
```

Boolean attributes like `disabled`, `autofocus`, and `readonly` are passed as keyword arguments:

```erb
<%= form.check_box :role, disabled: user.current? %>
<%= form.text_area :body, autofocus: card.title.blank? %>
<%= form.text_field :url, readonly: true %>
```

### Conditional Attributes

When you must use raw HTML (outside a form builder), **never duplicate an entire element just to toggle a boolean attribute.** Use an inline conditional:

```erb
<%# BAD — duplicates the whole element %>
<% if checked %>
  <input type="checkbox" name="<%= name %>" checked class="checkbox">
<% else %>
  <input type="checkbox" name="<%= name %>" class="checkbox">
<% end %>

<%# GOOD — inline conditional for the boolean attribute %>
<input type="checkbox" name="<%= name %>" <%= "checked" if checked %> class="checkbox">
```

This applies to all boolean HTML attributes: `checked`, `disabled`, `selected`, `autofocus`, `readonly`, `required`, `open`, etc.

For `data-*` attributes, use `tag.attributes` instead of duplicating the element in an `if/else`:

```erb
<el-dialog <%= tag.attributes(data: wired ? { controller: "my-dialog", action: "close->my-dialog#closed" } : {}) %>>
```

For standard HTML elements, use `tag.h3`, `tag.p`, etc. with a `data:` hash — Rails converts underscores to dashes automatically.

### Cookie-Based UI Preferences

```javascript
// app/javascript/helpers/cookie_helpers.js
export function setCookie(name, value = "", days = 365) { ... }
export function readCookie(name) { ... }
```

Use for sidebar state, edit mode, view preferences — things that don't need server round-trips.

## Fragment Caching

Cache partials aggressively:

```erb
<% cache message do %>
  <%# message rendering %>
<% end %>
```

Use `touch: true` on associations for cascade invalidation:

```ruby
belongs_to :room, touch: true
```

## JavaScript Organization

```
app/javascript/
├── application.js
├── controllers/        # Stimulus controllers (eager-loaded)
│   └── index.js
├── helpers/            # Shared utilities (timing, DOM, cookies)
├── models/             # JS model classes for complex logic
├── actions/            # Custom Turbo Stream actions
└── channels/           # ActionCable subscriptions (if needed)
```

All controllers are eager-loaded:

```javascript
// controllers/index.js
import { application } from "controllers/application"
import { eagerLoadControllersFrom } from "@hotwired/stimulus-loading"
eagerLoadControllersFrom("controllers", application)
```
