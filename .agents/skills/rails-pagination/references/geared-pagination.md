# Geared Pagination — Variable-Speed Infinite Scroll

Read this when implementing infinite scroll, "load more" buttons, or any list that doesn't need numbered page links. This is the default pagination choice for most Rails apps.

## Table of Contents
- [Setup](#setup)
- [Controller Usage](#controller-usage)
- [Page Object API](#page-object-api)
- [Variable Page Sizes](#variable-page-sizes)
- [View Integration with Turbo](#view-integration-with-turbo)
- [Pagination Helper](#pagination-helper)
- [Stimulus Controller](#stimulus-controller)
- [Day/Timeline Pagination](#daytimeline-pagination)

## Setup

```ruby
# Gemfile
gem "geared_pagination", "~> 1.2"
```

Include in `ApplicationController`:

```ruby
class ApplicationController < ActionController::Base
  include GearedPagination::Controller
end
```

## Controller Usage

One method call replaces Kaminari's `.page.per` chain:

```ruby
class BoardsController < ApplicationController
  def index
    set_page_and_extract_portion_from Current.user.boards
  end

  def show
    set_page_and_extract_portion_from @board.cards.awaiting_triage.latest.preloaded
  end
end
```

Works with any ActiveRecord relation — chain scopes before passing:

```ruby
set_page_and_extract_portion_from @card.comments.chronologically
set_page_and_extract_portion_from Current.account.tags.alphabetically
set_page_and_extract_portion_from @filter.cards
set_page_and_extract_portion_from Current.user.search(params[:q])
```

## Page Object API

`set_page_and_extract_portion_from` sets `@page` with:

- `@page.records` — the current page's records
- `@page.number` — current page number (1-indexed)
- `@page.before_last?` — `true` if more pages exist after this one
- No total count, no total pages — by design

## Variable Page Sizes

Geared pagination uses accelerating page sizes:

| Page | Records |
|------|---------|
| 1 | 15 |
| 2 | 30 |
| 3 | 50 |
| 4+ | 100 |

This loads the first page fast (15 records), then batches more aggressively as users scroll deeper. No `SELECT COUNT(*)` query is ever issued.

## View Integration with Turbo

### Automatic Pagination (Infinite Scroll)

The page loads the next batch automatically when the user scrolls to the bottom:

```erb
<%# app/views/boards/columns/show.html.erb %>
<%= with_automatic_pagination dom_id(@column, :cards), @page do %>
  <%= render partial: "cards/display/previews", locals: { cards: @page.records } %>
<% end %>
```

This generates:
1. A `turbo-frame` wrapping the records
2. An invisible pagination link observed by `IntersectionObserver`
3. When the link scrolls into view, the next page loads automatically

### Manual Pagination ("Load More" Button)

Shows a visible "Load more..." button:

```erb
<%= with_manual_pagination :comments, @page do %>
  <%= render @page.records %>
<% end %>
```

## Pagination Helper

Build a pagination helper for Turbo Frame integration:

```ruby
# app/helpers/pagination_helper.rb
module PaginationHelper
  def with_automatic_pagination(name, page, **properties)
    pagination_list name, paginate_on_scroll: true, **properties do
      concat(pagination_frame_tag(name, page) do
        yield
        concat link_to_next_page(name, page, activate_when_observed: true)
      end)
    end
  end

  def with_manual_pagination(name, page, **properties)
    pagination_list name, **properties do
      concat(pagination_frame_tag(name, page) do
        yield
        concat link_to_next_page(name, page)
      end)
    end
  end

  def pagination_frame_tag(namespace, page, data: {}, **attributes, &)
    turbo_frame_tag "#{namespace}-pagination-contents-#{page.number}",
      data: { timeline_target: "frame", **data }, role: "presentation", **attributes, &
  end

  def link_to_next_page(namespace, page, activate_when_observed: false, label: "Load more\u2026", data: {}, **attributes)
    if page.before_last? && !params[:previous]
      link_to label, url_for(params.permit!.to_h.merge(page: page.number + 1)),
        "aria-label": "Load page #{page.number + 1}",
        id: "#{namespace}-pagination-link-#{page.number + 1}",
        class: class_names("pagination-link",
          "pagination-link--active-when-observed": activate_when_observed,
          "btn txt-small center-block center": !activate_when_observed),
        data: {
          frame: "#{namespace}-pagination-contents-#{page.number + 1}",
          pagination_target: "paginationLink",
          action: ("click->pagination#loadPage:prevent" unless activate_when_observed),
          **data
        },
        **attributes
    end
  end

  private
    def pagination_list(name, tag_element: :div, paginate_on_scroll: false, **properties, &block)
      properties[:id] ||= "#{name}-pagination-list"
      tag.public_send tag_element,
        class: token_list(name, "display-contents"),
        data: { controller: "pagination",
                pagination_paginate_on_intersection_value: paginate_on_scroll },
        **properties, &block
    end
end
```

## Stimulus Controller

Handle both IntersectionObserver (automatic) and click (manual) pagination:

```javascript
// app/javascript/controllers/pagination_controller.js
import { Controller } from "@hotwired/stimulus"

const DELAY_BEFORE_OBSERVING = 400

export default class extends Controller {
  static targets = ["paginationLink"]
  static values = {
    paginateOnIntersection: { type: Boolean, default: false }
  }

  initialize() {
    this.activate()
  }

  disconnect() {
    this.observer?.disconnect()
  }

  async activate() {
    await new Promise(resolve => setTimeout(resolve, DELAY_BEFORE_OBSERVING))

    if (this.paginateOnIntersectionValue) {
      this.observer = new IntersectionObserver(
        ([entry]) => {
          if (entry?.isIntersecting && entry.intersectionRatio === 1) {
            this.#loadPaginationLink(entry.target)
          }
        },
        { rootMargin: "300px", threshold: 1 }
      )
    }
  }

  async paginationLinkTargetConnected(linkElement) {
    if (this.paginateOnIntersectionValue) {
      await new Promise(resolve => setTimeout(resolve, DELAY_BEFORE_OBSERVING))
      this.observer?.observe(linkElement)
    }
  }

  loadPage({ target }) {
    this.#loadPaginationLink(target)
  }

  #loadPaginationLink(linkElement) {
    this.observer?.unobserve(linkElement)
    linkElement.setAttribute("aria-busy", "true")

    const turboFrame = document.createElement("turbo-frame")
    turboFrame.id = linkElement.dataset.frame
    turboFrame.src = linkElement.href
    turboFrame.setAttribute("refresh", "morph")
    turboFrame.setAttribute("target", "_top")

    linkElement.parentNode.parentNode.append(turboFrame)
  }
}
```

### CSS for Invisible Scroll Triggers

```css
.pagination-link--active-when-observed {
  height: 0;
  overflow: hidden;
  visibility: hidden;
  display: block;
}
```

## Day/Timeline Pagination

For date-based feeds (activity logs), paginate by day instead of page number:

```ruby
# Helper
def day_timeline_pagination_link(day_timeline, filter)
  if day_timeline.next_day
    link_to "Load more\u2026",
      events_days_path(day: day_timeline.next_day.strftime("%Y-%m-%d"), **filter.as_params),
      class: "day-timeline-pagination-link",
      data: {
        frame: "day-timeline-pagination-contents-#{day_timeline.next_day.strftime("%Y-%m-%d")}",
        pagination_target: "paginationLink"
      }
  end
end
```

URL pattern: `?day=2026-02-05` instead of `?page=2`.
