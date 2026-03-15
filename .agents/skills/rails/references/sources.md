# Reference Sources

The patterns in this skill are distilled from three open-source Rails applications by Basecamp/37signals. The full source code lives in `../skip-samples/` (sibling to the project root) for deeper research when the reference docs here aren't sufficient.

## Sample Applications

### Writebook
- **Path:** `../skip-samples/writebook`
- **What it is:** A self-hosted publishing tool for books and manuals
- **Key patterns sourced:** Concern-based model organization, singleton resource controllers for state changes (closures, pins), delegated types, Current attributes, Turbo Stream broadcasts, ActionText with rich content

### Once Campfire
- **Source:** https://github.com/basecamp/once-campfire
- **Path:** `../skip-samples/once-campfire`
- **What it is:** A real-time group chat application (Once product line)
- **Key patterns sourced:** ActionCable presence tracking, Turbo Streams for real-time chat, notification bundling, join-model authorization, Solid Cable configuration, web push notifications

### Fizzy
- **Source:** https://github.com/basecamp/fizzy
- **Path:** `../skip-samples/fizzy`
- **What it is:** A full-featured project management and communication tool
- **Key patterns sourced:** Multi-concern model architecture, role-based authorization with creator fallback, @mention system, recurring Solid Queue jobs, FTS5 full-text search, filter query objects, complex Turbo Frame navigation

## How to Use

When the reference docs in this directory don't cover a pattern you need, read the sample app source directly:

```
# Example: see how Campfire handles presence
Read ../skip-samples/once-campfire/app/models/user/presence.rb

# Example: see how Writebook organizes model concerns
Glob ../skip-samples/writebook/app/models/**/*.rb
```

The reference docs should be the first stop. Only consult the full source when you need implementation details beyond what's documented here.
