# Architecture Patterns

## Interactors (`app/interactors/<domain>/`)

Primary pattern for business logic. One interactor = one operation.

```ruby
class Invoices::CreateInvoice
  include Interactor

  delegate :firm, :params, to: :context   # declare inputs

  def call
    create_invoice
    notify_user
  end

  private

  def create_invoice
    invoice = firm.invoices.build(params)
    context.fail!(error: "Save error: #{invoice.formatted_errors}") unless invoice.save
    context.invoice = invoice              # declare outputs
  end
end
```

**Organizer** (pipeline of interactors):
```ruby
class MarketingCampaigns::SendMarketingCampaign
  include Interactor::Organizer
  organize Send::PrepareAttributes, Send::PerformCampaign
end
```

**Calling an interactor:**
```ruby
result = SomeInteractor.call(param1: value, param2: value)
result.success?   # true/false
result.invoice    # output set by the interactor
result.error      # set on context.fail!
```

**With transaction:**
```ruby
def call
  ActiveRecord::Base.transaction do
    step_one
    step_two
  end
rescue SomeError => e
  context.fail!(error: e.message)
end
```

Rules:
- No `before`/`after` hooks — not used in this codebase
- `context.fail!` stops execution and rolls back organizer
- Inputs via `delegate :x, to: :context`; outputs via `context.x = value`

---

## Service Objects (`app/services/<domain>/`)

Use for: external API wrappers, complex utilities shared across domains.

```ruby
class LeadScoring::LeadScoreProfileService
  def self.duplicate_lead_score_profile(original, user)
    # ...
  end
end
```

---

## Concerns (`app/models/concerns/`)

Use when 2+ models share identical behavior.

```ruby
module Taggable
  extend ActiveSupport::Concern
  included do
    has_many :taggings, as: :taggable
    scope :tagged_with, ->(tag) { joins(:taggings).where(...) }
  end
end
```

---

## Model conventions

```ruby
class Matter < ApplicationRecord
  acts_as_paranoid                           # soft delete
  belongs_to :firm
  scope :for_firm, ->(firm) { where(firm:) } # always scope by firm
  include Taggable
  include TimelineActivityTrackable
end
```

- Base: `ApplicationRecord`
- Soft delete: `acts_as_paranoid` for user-facing entities
- Multi-tenancy: always `belongs_to :firm` + `firm_id` on every table
- Validators: `app/models/validators/`
