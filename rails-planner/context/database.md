# Database Patterns

## Migration style (Rails 7.2)

```ruby
class CreateTeamsIntegrations < ActiveRecord::Migration[7.2]
  def change
    create_table :teams_integrations do |t|
      t.references :firm,      null: false, foreign_key: true, index: true
      t.references :firm_user, null: false, foreign_key: { on_delete: :cascade }, index: true
      t.string  :status,       null: false, default: 'pending'
      t.text    :access_token_ciphertext    # encrypted columns use *_ciphertext suffix
      t.boolean :enabled,      null: false, default: false
      t.timestamps
    end

    add_index :teams_integrations, [:firm_id, :status], name: 'idx_teams_integrations_firm_status'
  end
end
```

---

## Column conventions

| Need | Convention |
|------|-----------|
| Foreign key | `t.references :model, null: false, foreign_key: true, index: true` |
| Cascade delete | `foreign_key: { on_delete: :cascade }` |
| Encrypted field | `t.text :field_name_ciphertext` |
| Soft delete | `t.datetime :deleted_at, index: true` (added by `acts_as_paranoid`) |
| Enum | `t.string :status, null: false, default: 'active'` + enum in model |
| Multi-tenancy | Every table needs `t.references :firm, null: false, foreign_key: true` |

---

## Index conventions

```ruby
# Always name composite indexes explicitly
add_index :table, [:firm_id, :created_at], name: 'idx_table_firm_created_at'

# Unique constraint
add_index :table, [:firm_id, :email], unique: true, name: 'idx_table_firm_email_unique'
```

---

## Safety rules — strong_migrations (CI enforced)

The CI runs `rake strong_migrations:check_json` on every PR touching `db/**`. Unsafe operations **block the PR**.

| Unsafe operation | Safe alternative |
|-----------------|-----------------|
| Add NOT NULL column without default | Add nullable → backfill → add constraint in separate PRs |
| Add index (locks table) | `add_index ..., algorithm: :concurrently` + `disable_ddl_transaction!` |
| Add foreign key | `add_foreign_key ..., validate: false` then validate separately |
| Change column type | New column → backfill → swap |
| Rename table/column | New name → dual-write → remove old |

**`safety_assured { }`** bypasses the check but **triggers a human-review warning in CI**. Use only when you can prove it's safe (e.g., table has 0 rows, or operation is actually safe for your Postgres version).

---

## CI migration checks (all run on PRs touching `db/**` or `app/models/**`)

1. **strong_migrations** — static analysis of unsafe operations
2. **reversibility** — runs UP → DOWN → UP on a real Postgres instance. Migration **must be fully reversible**. Never `drop_table` without a `create_table` in `down`.
3. **annotation check** — after any migration, model annotations must be updated:
   ```bash
   bundle exec annotaterb annotate_models
   ```
   Commit the updated annotation comments or CI fails.

---

## Soft delete (paranoia)

Add to model: `acts_as_paranoid`  
Adds `deleted_at datetime` column — migration:

```ruby
add_column :table_name, :deleted_at, :datetime
add_index  :table_name, :deleted_at
```

Default scopes automatically exclude deleted records. Use `.with_deleted` to include them.
