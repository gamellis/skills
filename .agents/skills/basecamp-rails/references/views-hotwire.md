# Views & Hotwire

Read this when building interactive UI, adding real-time updates to a page, writing Stimulus controllers, or choosing between Turbo Streams and Turbo Frames.

## Table of Contents
- [No JavaScript Framework](#no-javascript-framework)
- [Turbo Streams](#turbo-streams)
- [Custom Turbo Stream Actions](#custom-turbo-stream-actions)
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
