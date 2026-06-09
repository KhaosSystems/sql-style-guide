# Khaos Systems SQL Style Guide

This is a simple, opinionated SQL style guide that we follow for all internal SQL code, based on [Matt Mazur's SQL Style Guide](https://github.com/mattm/sql-style-guide), but with some adjustments to fit our specific needs and preferences.

The most notable deviation from Mazur is that we use **singular** table and column names (Mazur uses plural). We might divert more in the future, but for now Matt's style is pretty solid for our needs.

## Example
```sql
create table project (
    id serial primary key,
    name text not null,
    engine text not null, -- Unity, Unreal, Godot
    created_at timestamptz not null default now()
);

-- Non-trivial query
select
    project.name,
    count(*) as open_task_count,
    sum(task.estimated_hours) as remaining_hours
from project
inner join task on project.id = task.project_id
where task.status != 'Done'
group by project.id, project.name
having count(*) >= 3
order by remaining_hours desc;
```

## Guidelines

### Use lowercase SQL
It's just as readable as uppercase SQL and you won't have to constantly be holding down a shift key.
```sql
-- Good
select * from user_account;

-- Bad
SELECT * FROM user_account;

-- Bad
Select * From user_account;
```

### Put each selected column on its own line
```sql
-- Good
select
    id,
    email
from user;

-- Bad
select id, email from user;
```

### `select *`
When selecting everything, it's fine to keep `select *` on a single line:
```sql
-- Good
select * from user;
```

### Indenting where conditions
When there's only one condition, leave it on the same line as `where`. With multiple conditions, put `where` on its own line and indent each condition one level deeper, with the logical operator (`and`/`or`) at the start of each subsequent line.
```sql
-- Good
select *
from user
where is_cancelled = false

-- Good
select *
from user
where
    is_cancelled = false
    and created_at > '2026-01-01'
```

### Left align SQL keywords
Keep `select`, `from`, `where`, etc. left-aligned at the same indentation level.
```sql
-- Good
select
    id,
    email
from user
where email like '%@example.com'

-- Bad
select
    id,
    email
  from user
 where email like '%@example.com'
```

### Use single quotes
Some SQL dialects like BigQuery support using double quotes, but for most dialects double quotes will wind up referring to column names. For that reason, single quotes are preferable:
```sql
-- Good
select * from user_account where name = 'Alice';

-- Bad
select * from user_account where name = "Alice";
```

### Use `!=` over `<>`
Because nobody cares that you used to program Pascal/BASIC/Ada, and `!=` is the standard in every modern programming language <3
```sql
-- Good
select * from user_account where name != 'Alice';

-- Bad
select * from user_account where name <> 'Alice';
```

### Commas should be at the end of lines
Some people argue that leading commas are easier to spot missing, but it looks messy and we therefore prefer trailing commas. It aligns better with how we write code in general.
```sql
-- Good
select
    name,
    email,
    created_at
from user_account;

-- Bad
select
    name
    , email
    , created_at
from user_account;
```

### No spaces inside parentheses
```sql
-- Good
select (1 + 2) * (3 + 4);

-- Bad
select ( 1 + 2 ) * ( 3 + 4 );
```

### Break long lists of `in` values into multiple indented lines
```sql
-- Good
select *
from user
where email in (
    'user-1@example.com',
    'user-2@example.com',
    'user-3@example.com',
    'user-4@example.com'
);
```

### Use snake_case for table and column names
Snake case is more readable than camelCase or PascalCase for long explicit names, which is sometimes necessary- especially for domain-heavy software like ours.
```sql
-- Good
select * from long_and_explicit_table_name_for_a_good_reason;

-- Bad
select * from longAndExplicitTableNameForAGoodReason;

-- Bad
select * from LongAndExplicitTableNameForAGoodReason;
```

### Use singular names for both tables and columns
 1. Singular is consistent. English is full of irregular plurals, or words with no plural form at all, but will almost always have a singular form. E.g.
    * There are no plurals for "information", "equipment", "furniture", etc.
    * Some plurals are irregular, e.g. "person" -> "people", "child" -> "children", "mouse" -> "mice", etc.
        This is especially problematic when using codegen, since the codegen won't know how to pluralize correctly, and thus requires manual overrides for pluralization rules, which is extra work and a potential source of issues depending on codegen strategy.
 2. Ordering, specifically in master-detail relationships, it aligns better by name (master first, then detail):
    * `order`, `order_item` (instead of `orders`, `order_items`)

> Note: this is a deliberate deviation from Mazur, who uses plural table names.

```sql
-- Good
create table user ( ... )
create table user_account ( ... )
create table user_profile ( ... )

select * from user inner join user_account on user.id = user_account.user_id

-- Bad
create table users ( ... )
create table user_accounts ( ... )
create table user_profiles ( ... )

select * from users inner join user_accounts on users.id = user_accounts.user_id
```

### Column naming conventions
 - Boolean fields should be prefixed with `is_`, `has_`, or `does_`. E.g., `is_customer`, `has_unsubscribed`, `does_have_permission`.
 - Id fields should be suffixed with `_id`. E.g., `user_id`, `project_id`.
 - Timestamps should be suffixed with `_at`. E.g., `created_at`, `updated_at`.
 - Date-only fields should be suffixed with `_date`. E.g. `start_date`.
 - When a unit is relevant, include it in the name. E.g. `estimated_hours`, `file_size_kb`, `duration_seconds`, etc.

### Column order conventions
When defining a table, order columns as: primary key first, then foreign keys, then the other columns, then system/timestamp columns (`created_at`, `updated_at`, etc.) last.
```sql
-- Good
create table task (
    id serial primary key,
    project_id integer not null references project(id),
    name text not null,
    estimated_hours numeric,
    created_at timestamptz not null default now(),
    updated_at timestamptz not null default now()
);
```

### Include `inner` for inner joins
Be explicit; `join` on its own is ambiguous to read.
```sql
-- Good
select
    user.email,
    charge.amount
from user
inner join charge on user.id = charge.user_id

-- Bad
select
    user.email,
    charge.amount
from user
join charge on user.id = charge.user_id
```

### Put the table referenced first immediately after the `on`
This makes it easier to determine whether your join is going to cause fan-out.
```sql
-- Good
from user
inner join charge on user.id = charge.user_id

-- Bad
from user
inner join charge on charge.user_id = user.id
```

### Single join conditions on the same line as the join
With multiple conditions, put each on its own indented line.
```sql
-- Good
from user
inner join charge on user.id = charge.user_id

-- Good
from user
inner join charge
    on user.id = charge.user_id
    and charge.is_refunded = false
```

### Avoid aliasing table names most of the time
Prefer the full (singular) table name. Only alias when the name is very long, or when you join the same table more than once.
```sql
-- Good
select
    user.email,
    sum(charge.amount) as total_revenue
from user
inner join charge on user.id = charge.user_id

-- Bad
select
    u.email,
    sum(c.amount) as total_revenue
from user u
inner join charge c on u.id = c.user_id
```

### Include the table when there is a join, but omit it otherwise
When there are no joins involved, there's no ambiguity around which table the columns came from so you can leave the table name out:
```sql
-- Good
select
    id,
    email
from user;

-- Bad
select
    company.id,
    company.name
from company
```
But when there are joins involved, it's better to be explicit so it's clear where the columns originated:
```sql
-- Good
select
    user.email,
    sum(charge.amount) as total_revenue
from user
inner join charge on user.id = charge.user_id

-- Bad
select
    email,
    sum(amount) as total_revenue
from user
inner join charge on user.id = charge.user_id
```

### Always rename aggregates and function-wrapped arguments
```sql
-- Good
select
    count(*) as total_user_count
from user

-- Bad
select
    count(*)
from user

-- Good
select
    timestamp_millis(property_beacon_interest) as expressed_interest_at
from hubspot.contact
where property_beacon_interest is not null

-- Bad
select
    timestamp_millis(property_beacon_interest)
from hubspot.contact
where property_beacon_interest is not null
```

### Be explicit in boolean conditions
```sql
-- Good
select *
from customer
where is_cancelled = true

-- Good
select *
from customer
where is_cancelled = false

-- Bad
select *
from customer
where is_cancelled

-- Bad
select *
from customer
where not is_cancelled
```

### Use `as` to alias tables and columns
```sql
-- Good
select
    id,
    email,
    timestamp_trunc(created_at, month) as signup_month
from user

-- Bad
select
    id,
    email,
    timestamp_trunc(created_at, month) signup_month
from user
```

### Group using column names or numbers, but not both
I prefer grouping by name, but grouping by numbers is [also fine](https://www.getdbt.com/blog/write-better-sql-a-defense-of-group-by-1).
```sql
-- Good
select
    user_id,
    count(*) as total_charge_count
from charge
group by user_id

-- Good
select
    user_id,
    count(*) as total_charge_count
from charge
group by 1

-- Bad
select
    timestamp_trunc(created_at, month) as signup_month,
    vertical,
    count(*) as user_count
from user
group by 1, vertical
```

### Take advantage of lateral column aliasing when grouping by name
```sql
-- Good
select
    timestamp_trunc(created_at, year) as signup_year,
    count(*) as total_company_count
from company
group by signup_year

-- Bad
select
    timestamp_trunc(created_at, year) as signup_year,
    count(*) as total_company_count
from company
group by timestamp_trunc(created_at, year)
```

### Grouping columns should go first
```sql
-- Good
select
    timestamp_trunc(created_at, year) as signup_year,
    count(*) as total_company_count
from company
group by signup_year

-- Bad
select
    count(*) as total_company_count,
    timestamp_trunc(created_at, year) as signup_year
from company
group by signup_year
```

### Aligning case/when statements
Each `when` should be on its own line (nothing on the `case` line) and should be indented one level deeper than the `case` line. The `then` can be on the same line or on its own line below it, just aim to be consistent.
```sql
-- Good
select
    case
        when event_name = 'viewed_homepage' then 'Homepage'
        when event_name = 'viewed_editor' then 'Editor'
        else 'Other'
    end as page_name
from event

-- Good too
select
    case
        when event_name = 'viewed_homepage'
            then 'Homepage'
        when event_name = 'viewed_editor'
            then 'Editor'
        else 'Other'
    end as page_name
from event

-- Bad
select
    case when event_name = 'viewed_homepage' then 'Homepage'
        when event_name = 'viewed_editor' then 'Editor'
        else 'Other'
    end as page_name
from event
```

### Use CTEs, not subqueries
CTEs make queries easier to read and let you name intermediate results. Prefer them over nested subqueries.
```sql
-- Good
with paying_user as (
    select
        user.id as user_id,
        sum(charge.amount) as total_revenue
    from user
    inner join charge on user.id = charge.user_id
    group by user.id
)

select * from paying_user

-- Bad
select *
from (
    select
        user.id as user_id,
        sum(charge.amount) as total_revenue
    from user
    inner join charge on user.id = charge.user_id
    group by user.id
) as paying_user
```

### Use meaningful CTE names
Name CTEs after what they contain, not `d1`, `t2`, etc.
```sql
-- Good
with active_subscription as ( ... )

-- Bad
with d1 as ( ... )
```

### Window functions
Keep a window function on one line if it fits. If it's long, break the `partition by` and `order by` onto their own indented lines.
```sql
-- Good (short, one line)
select
    user_id,
    row_number() over (partition by user_id order by created_at) as charge_rank
from charge

-- Good (long, broken up)
select
    user_id,
    row_number() over (
        partition by user_id
        order by created_at desc
    ) as charge_rank
from charge
```

## Credits
Based on [Matt Mazur's SQL Style Guide](https://github.com/mattm/sql-style-guide), adapted with our own conventions (notably singular naming).
