# ORM Migration Patterns

## Prisma (TypeScript/Node.js)

### Workflow
```bash
# After editing schema.prisma:
npx prisma migrate dev --name descriptive_name   # dev
npx prisma migrate deploy                         # production
```

### Multi-step migration for column rename (data-safe)
Prisma doesn't support column renames natively without data loss.
Use a two-migration approach:

**Migration 1** — add new column and backfill:
```sql
-- In the generated SQL file, manually edit to add:
ALTER TABLE "User" ADD COLUMN "displayName" TEXT;
UPDATE "User" SET "displayName" = "username";
```

**Migration 2** — remove old column (after deploying migration 1 and updating app code):
```sql
ALTER TABLE "User" DROP COLUMN "username";
```

### Adding a relation (join table)
Prisma auto-generates the join table for `@relation` many-to-many.
For explicit join tables with extra fields, use an explicit model:

```prisma
model PostTag {
  postId    Int
  tagId     Int
  createdAt DateTime @default(now())

  post Post @relation(fields: [postId], references: [id], onDelete: Cascade)
  tag  Tag  @relation(fields: [tagId], references: [id], onDelete: Cascade)

  @@id([postId, tagId])
  @@index([tagId])
}
```

### Adding indexes in Prisma
```prisma
model Order {
  id         Int      @id @default(autoincrement())
  userId     Int
  status     String
  createdAt  DateTime @default(now())

  @@index([userId])
  @@index([status, createdAt])  // composite for filtering+sorting
}
```

---

## Alembic (Python / SQLAlchemy)

### Workflow
```bash
alembic revision --autogenerate -m "descriptive_name"
alembic upgrade head        # apply
alembic downgrade -1        # rollback one step
```

### Safe column rename pattern
```python
def upgrade():
    op.add_column('user', sa.Column('display_name', sa.String(255)))
    op.execute("UPDATE user SET display_name = username")
    # Deploy app code here before running next migration

def downgrade():
    op.drop_column('user', 'display_name')
```

```python
# Second migration
def upgrade():
    op.drop_column('user', 'username')

def downgrade():
    op.add_column('user', sa.Column('username', sa.String(255)))
    op.execute("UPDATE user SET username = display_name")
```

### Adding indexes concurrently (PostgreSQL only)
```python
def upgrade():
    op.create_index(
        'ix_order_user_id',
        'order',
        ['user_id'],
        postgresql_concurrently=True
    )

def downgrade():
    op.drop_index('ix_order_user_id', table_name='order')
```

### Batch operations (SQLite doesn't support ALTER)
```python
with op.batch_alter_table('user') as batch_op:
    batch_op.add_column(sa.Column('display_name', sa.String(255)))
    batch_op.drop_column('username')
```

---

## Drizzle (TypeScript/Node.js)

### Workflow
```bash
npx drizzle-kit generate   # generate migration SQL
npx drizzle-kit migrate    # apply
```

### Schema definition with indexes
```ts
import { pgTable, serial, text, integer, index, uniqueIndex } from 'drizzle-orm/pg-core';

export const orders = pgTable('orders', {
  id: serial('id').primaryKey(),
  userId: integer('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  status: text('status').notNull().default('pending'),
}, (table) => ({
  userIdIdx: index('orders_user_id_idx').on(table.userId),
  statusIdx: index('orders_status_idx').on(table.status),
}));
```

### Multi-step migration
Drizzle generates raw SQL — edit the generated file to split into steps:
```sql
-- Step 1 (safe)
ALTER TABLE "orders" ADD COLUMN "customer_name" text;
UPDATE "orders" SET "customer_name" = (SELECT name FROM users WHERE users.id = orders.user_id);

-- Step 2 (run after verifying)
-- ALTER TABLE "orders" DROP COLUMN "user_name_cache";
```

---

## ActiveRecord (Ruby on Rails)

### Workflow
```bash
rails generate migration DescriptiveName
rails db:migrate
rails db:rollback   # undo last
```

### Safe column rename
```ruby
class RenameUsernameToDisplayName < ActiveRecord::Migration[7.1]
  def up
    add_column :users, :display_name, :string
    User.find_each { |u| u.update_columns(display_name: u.username) }
    # Deploy app code before running the next migration
  end

  def down
    remove_column :users, :display_name
  end
end
```

### Adding indexes
```ruby
class AddIndexesToOrders < ActiveRecord::Migration[7.1]
  disable_ddl_transaction!   # required for concurrent indexes

  def change
    add_index :orders, :user_id, algorithm: :concurrently
    add_index :orders, [:status, :created_at], algorithm: :concurrently
  end
end
```