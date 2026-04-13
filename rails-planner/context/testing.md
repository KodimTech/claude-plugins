# Testing Patterns (RSpec + FactoryBot)

## File locations

| What | Where |
|------|-------|
| Interactor | `spec/interactors/<domain>/<name>_spec.rb` |
| GraphQL mutation | `spec/mutations/<domain>/<name>_spec.rb` |
| GraphQL query | `spec/queries/<domain>/<name>_spec.rb` |
| Model | `spec/models/<name>_spec.rb` |
| Factory | `spec/factories/<name>.rb` |

---

## Interactor spec

```ruby
RSpec.describe Invoices::CreateInvoice do
  let_it_be(:firm) { SpecContext.firm }           # shared across all examples (test-prof)
  let_it_be(:user) { create(:firm_user, firm:) }

  let(:params) { { amount: 100.0, description: 'Test' } }

  subject(:result) { described_class.call(firm:, params:) }

  it 'creates the invoice' do
    expect(result).to be_success
    expect(result.invoice).to be_persisted
    expect(result.invoice.amount).to eq(100.0)
  end

  context 'when params are invalid' do
    let(:params) { { amount: nil } }

    it 'fails with an error' do
      expect(result).to be_failure
      expect(result.error).to include('Save error')
    end
  end
end
```

---

## Mutation spec

```ruby
RSpec.describe 'Mutation: createInvoice' do
  let(:variables) { { amount: 100.0 } }

  subject(:response) { create_invoice(variables) }   # helper defined in spec support

  it 'returns the created invoice' do
    expect(response.dig('data', 'createInvoice', 'id')).to be_present
  end

  context 'when feature is disabled' do
    before { current_firm.update!(feature_flags: {}) }

    it 'returns an error' do
      expect(response['errors'].first['message']).to include('not enabled')
    end
  end
end
```

---

## Factory conventions

```ruby
factory :invoice do
  firm         { SpecContext.firm }          # use SpecContext.firm for multi-tenant factories
  amount       { 100.0 }
  description  { 'Test invoice' }
  association  :created_by, factory: :firm_user

  trait :paid do
    status { :paid }
  end

  trait :with_line_items do
    after(:create) { |inv| create_list(:line_item, 2, invoice: inv) }
  end
end
```

---

## Key conventions

- `let_it_be` (TestProf) for data shared across all examples — avoids redundant DB writes
- `let!` for eager creation needed within a single example group
- `let` for lazy data (computed only when referenced)
- Always use `described_class` for the class under test
- Check `result.success?` / `result.failure?` for interactors — never inspect internals directly
- Use `SpecContext.firm` in factories to stay in the test tenant
