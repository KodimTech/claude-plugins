# GraphQL Patterns

## Mutation module structure

Mutations are **module mixins**, not standalone classes.  
Location: `app/graphql/types/mutations/<domain>_mutation_type.rb`

```ruby
module Types
  module Mutations
    module InvoiceMutationType
      include Base

      included do
        field :create_invoice, Types::InvoiceType, null: false do
          argument :amount,      Float,   required: true
          argument :description, String,  required: false
        end

        field :delete_invoice, Types::InvoiceType, null: true do
          argument :id, ID, required: true
        end
      end

      def create_invoice(amount:, description: nil)
        return GraphQL::ExecutionError.new('Feature not enabled') unless current_firm.can_use_feature?(:billing)

        result = Invoices::CreateInvoice.call(firm: current_firm, params: { amount:, description: })
        return GraphQL::ExecutionError.new(result.error) if result.failure?

        result.invoice
      rescue StandardError => e
        GraphQL::ExecutionError.new("Failed: #{e.message}")
      end

      def delete_invoice(id:)
        invoice = current_firm.invoices.find_by(id:)
        return GraphQL::ExecutionError.new('Invoice not found') unless invoice

        invoice.destroy ? invoice : ValidationError.new('Destroy failed', invoice)
      end
    end
  end
end
```

**Register** in `app/graphql/types/mutations/mutation_type.rb`:
```ruby
include InvoiceMutationType
```

---

## Error handling rules

| Situation | Pattern |
|-----------|---------|
| Auth / feature check | `return GraphQL::ExecutionError.new('...')` |
| Record not found | `return GraphQL::ExecutionError.new('... not found')` |
| Interactor failure | `return GraphQL::ExecutionError.new(result.error)` |
| Model save/destroy failure | `ValidationError.new('message', model)` |
| Unexpected exception | `rescue StandardError => e; GraphQL::ExecutionError.new(e.message)` |

---

## Query / Type patterns

- Queries: `app/graphql/queries/<domain>/`
- Types: `app/graphql/types/`
- Helpers available in resolvers: `current_firm`, `current_user`, `context[:current_firm]`
- Use `argument :id, ID, as: :record_id` to rename args internally

---

## CI check — GraphQL schema (runs on PRs touching `app/graphql/**`)

After any change to types, mutations, or queries, the schema dump must be updated or CI fails:

```bash
bundle exec rake graphql:dump
```

Commit the updated schema file. The check runs `rake graphql:check` and diffs against the committed version.

---

## Argument tips

```ruby
argument :status, LeadScoreProfileStatusEnum, required: false
argument :firm_user_id, ID, required: true, as: :firm_user  # rename
argument :options, JsonType, required: false                  # free-form JSON
```
