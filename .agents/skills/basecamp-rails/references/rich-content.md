# Rich Content & ActionText

Read this when adding rich text editing, implementing @mentions, or choosing between ActionText and a Markdown-based approach.

## Table of Contents
- [ActionText Basics](#actiontext-basics)
- [Mentions System](#mentions-system)
- [Markdown Alternative](#markdown-alternative)

## ActionText Basics

Use `has_rich_text` for rich content:

```ruby
class Card < ApplicationRecord
  has_rich_text :description
end

class Comment < ApplicationRecord
  has_rich_text :body
end
```

In forms:

```erb
<%= form.rich_text_area :description %>
```

## Mentions System

Embed domain models (users) as ActionText attachments for @mentions.

### Make Users Attachable

```ruby
# app/models/user/mentionable.rb
module User::Mentionable
  include ActionText::Attachable

  def to_attachable_partial_path
    "users/mention"
  end

  def to_trix_content_attachment_partial_path
    "users/mention"
  end

  def attachable_plain_text_representation(caption)
    "@#{name}"
  end
end
```

### Extract Mentions from Rich Text

```ruby
# app/models/concerns/mentions.rb
module Mentions
  extend ActiveSupport::Concern

  included do
    has_many :mentions, as: :source, dependent: :destroy
    after_save_commit :create_mentions_later, if: :mentionable_content_changed?
  end

  def create_mentions(mentioner: Current.user)
    mentionees_from_attachments.each do |user|
      user.mentioned_by(mentioner, at: self)
    end
  end

  private
    def mentionees_from_attachments
      body.body&.attachables&.grep(User)&.uniq || []
    end
end
```

The key technique: `body.body.attachables.grep(User)` — first `.body` is the `has_rich_text` proxy, second `.body` is the `ActionText::Content`, `.attachables` returns embedded objects, `.grep(User)` filters to users.

### Mention Model

```ruby
class Mention < ApplicationRecord
  belongs_to :source, polymorphic: true
  belongs_to :mentioner, class_name: "User"
  belongs_to :mentionee, class_name: "User"

  after_create_commit :notify_mentionee
  after_create_commit :watch_source_by_mentionee

  private
    def watch_source_by_mentionee
      source.watch_by(mentionee)  # auto-subscribe to future activity
    end
end
```

### Guard by Lifecycle

Only process mentions in published content:

```ruby
module Card::Mentions
  extend ActiveSupport::Concern
  include ::Mentions

  def mentionable? = published?

  def should_check_mentions?
    was_just_published?  # scan on publish even if content didn't change
  end
end
```

## Markdown Alternative

For content that's better suited to Markdown (documentation, books), build a parallel `has_markdown` DSL:

```ruby
# lib/rails_ext/action_text_has_markdown.rb
module ActionText::HasMarkdown
  class_methods do
    def has_markdown(name)
      has_one :"markdown_#{name}", -> { where(name: name) },
        class_name: "ActionText::Markdown", as: :record,
        autosave: true, dependent: :destroy

      class_eval <<-RUBY
        def #{name}
          markdown_#{name} || build_markdown_#{name}
        end

        def #{name}=(content)
          self.#{name}.content = content
        end
      RUBY
    end
  end
end
```

Use Redcarpet for rendering with a custom renderer for syntax highlighting and anchor links:

```ruby
class MarkdownRenderer < Redcarpet::Render::HTML
  include Rouge::Plugins::Redcarpet

  def header(text, level)
    id = text.parameterize
    "<h#{level} id='#{id}'>#{text} <a href='##{id}'>#</a></h#{level}>"
  end
end
```

In forms, use a custom `<house-md>` web component or `textarea` instead of Trix:

```ruby
# Form helper
def markdown_area(record, name, **options)
  tag.textarea(record.send(name).content, name: "#{record.model_name.param_key}[#{name}]", **options)
end
```
