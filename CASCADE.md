# CASCADE

In PostgreSQL, **CASCADE** is an option you can use with foreign keys, **DROP**, and other 
commands to manage dependencies between tables and ensure data integrity. 
Here’s a breakdown of where and how **CASCADE** is used:

## Foreign Key Constraints with ON DELETE CASCADE / ON UPDATE CASCADE

When defining foreign key relationships, you can use `ON DELETE CASCADE` or `ON UPDATE CASCADE` 
to specify what should happen to dependent rows when a referenced row in the parent table is deleted or updated.

- **ON DELETE CASCADE:** If a row in the parent table is deleted, 
    all related rows in the child table(s) are automatically deleted.

Useful for maintaining referential integrity when deleting data from parent tables.

```sql
CREATE TABLE parent (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE child (
    id SERIAL PRIMARY KEY,
    parent_id INTEGER REFERENCES parent(id) ON DELETE CASCADE,
    description VARCHAR(100)
);
```

In this example, deleting a row from parent will automatically delete related rows in 
child where `parent_id` matches the deleted `id`.

- **ON UPDATE CASCADE:** If a row’s primary key in the parent table is updated, 
PostgreSQL automatically updates the foreign key values in related child rows.

Useful if primary keys might change and you want the related rows to stay in sync.

```sql
CREATE TABLE parent (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE child (
    id SERIAL PRIMARY KEY,
    parent_id INTEGER REFERENCES parent(id) ON UPDATE CASCADE,
    description VARCHAR(100)
);
```

Here, if the `id` of a row in parent is updated, the `parent_id` in child will automatically be updated to match the new `id`.

## ON DELETE SET NULL / ON DELETE SET DEFAULT

These are alternatives to `CASCADE` that also help maintain referential integrity:

- **ON DELETE SET NULL:** Sets the foreign key column in the child table to `NULL` when the parent row is deleted.

Use this when the relationship is optional, and the child row should remain without a parent if the parent is deleted.

```sql
CREATE TABLE child (
    id SERIAL PRIMARY KEY,
    parent_id INTEGER REFERENCES parent(id) ON DELETE SET NULL,
    description VARCHAR(100)
);
```

- **ON DELETE SET DEFAULT:** Sets the foreign key column in the child table to its default value when the parent row is deleted.

Requires the foreign key column to have a default value.

```sql
CREATE TABLE child (
    id SERIAL PRIMARY KEY,
    parent_id INTEGER DEFAULT 1 REFERENCES parent(id) ON DELETE SET DEFAULT,
    description VARCHAR(100)
);
```

## CASCADE with DROP Commands

When you drop a table or other database objects, `CASCADE` is used to drop all dependent objects automatically. For example:

- **DROP TABLE ... CASCADE:** Drops the table and automatically drops any dependent objects (like foreign keys in other tables) that reference it.

```sql
DROP TABLE parent CASCADE;
```

This command will drop the parent table and any tables that have foreign keys pointing to it, or at least those specific foreign key constraints.

- **DROP VIEW ... CASCADE:** Drops a view and any dependent views that rely on it.

```sql
DROP VIEW my_view CASCADE;
```

- **DROP FUNCTION ... CASCADE:** Drops a function and any other functions, views, or triggers that depend on it.

```sql
DROP FUNCTION my_function CASCADE;
```

## Summary of Cascade Options in PostgreSQL

|Usage                  |Description                                                                                        |
|:----------------------|:--------------------------------------------------------------------------------------------------|
|ON DELETE CASCADE      |Deletes related rows in the child table when a row in the parent table is deleted.                 |
|ON UPDATE CASCADE      |Updates foreign key values in the child table when a primary key in the parent table is updated.   |
|ON DELETE SET NULL     |Sets the foreign key in the child table to NULL when the parent row is deleted.                    |
|ON DELETE SET DEFAULT  |Sets the foreign key in the child table to its default value when the parent row is deleted.       |
|DROP ... CASCADE       |Deletes the object and all dependent objects, such as foreign keys, views, or triggers.            |


## Best Practices with CASCADE

- **Use Cascades Carefully:** Especially with `ON DELETE CASCADE`, as it can lead to unexpected data loss if not carefully managed.
- **Consider Referential Integrity Needs:** Think about whether child records should be deleted, updated, set to `NULL`, or left alone if the parent record changes.
- **Plan Dependencies:** When using `DROP ... CASCADE`, be aware of dependencies so you don’t inadvertently drop needed objects.

