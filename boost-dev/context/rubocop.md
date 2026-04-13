# Rubocop Rules (boost-api)

Config files: `.rubocop.yml` (base) + `.rubocop_custom.yml` (overrides) + `.rubocop_todo.yml` (ignored).
Target: Ruby 3.3, Rails 7.1.

## Key rules for the executor

### What's DISABLED (don't worry about these)
- `Style/Documentation` — no class/module docstrings needed
- `Style/StringLiterals` — single or double quotes, either is fine
- `Style/NumericPredicate` — `.zero?` and `== 0` both OK
- `Rails/SkipsModelValidations` — `update_column`, `update_all` allowed
- `RSpec/MultipleExpectations` — multiple `expect` per example is fine
- `RSpec/MultipleMemoizedHelpers` — many `let` blocks per spec is fine
- `RSpec/BeforeAfterAll` — `before(:all)` allowed

### What IS enforced
- `RSpec/ExampleLength: Max: 20` — keep examples under 20 lines (arrays/hashes/heredocs count as 1)
- `Layout/FirstArrayElementIndentation: consistent` — first element aligns with `[`
- `Layout/FirstHashElementIndentation: consistent` — first key aligns with `{`
- `Layout/MultilineMethodCallIndentation: indented` — chained calls indented 2 spaces

### Hash syntax
`Style/HashSyntax: EnforcedShorthandSyntax: either` — both forms accepted:
```ruby
{ key: value }      # OK
{ key => value }    # OK
foo(key:)           # shorthand OK
foo(key: key)       # explicit OK
```

### Running rubocop
```bash
# Check specific files
bundle exec rubocop app/path/to/file.rb spec/path/to/spec.rb --format simple

# Auto-fix safe offenses
bundle exec rubocop -a app/path/to/file.rb

# Auto-fix all (including unsafe)
bundle exec rubocop -A app/path/to/file.rb
```

### When rubocop fails
1. Run `-a` first (safe autocorrect)
2. If offenses remain, fix manually
3. If a cop is genuinely not applicable, add inline disable with comment explaining why:
   ```ruby
   rubocop:disable Metrics/MethodLength # complex migration requires this
   ```
4. Never add to `.rubocop_todo.yml` — that file is for legacy issues only
