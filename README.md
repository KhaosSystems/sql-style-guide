
# Khaos Systems SQL Style Guide

This is a simple, opinionated SQL style guide that we follow for all internal SQL code, based on [Matt Mazur's SQL Style Guide](https://github.com/mattm/sql-style-guide), but with some adjustments to fit our specific needs and preferences. So thanks a ton to Matt and everyone involved!

The most notable deviation from Mazur is: 
 1. we use **singular** table and column names
 2. we use **_unit** suffixes for columns where relevant, e.g. `duration_seconds`, `file_size_kb`, etc.

## Example

Here's a non-trivial query to give you an idea of what this style guide looks like in the practice:

```sql
with hubspot_interest as (

    select
        email,
        timestamp_millis(property_beacon_interest) as expressed_interest_at
    from hubspot.contact
    where 
        property_beacon_interest is not null

), 

support_interest as (

    select 
        conversation.email,
        conversation.created_at as expressed_interest_at
    from helpscout.conversation
    inner join helpscout.conversation_tag on conversation.id = conversation_tag.conversation_id
    where 
        conversation_tag.tag = 'beacon-interest'

), 

combined_interest as (

    select * from hubspot_interest
    union all
    select * from support_interest

),

first_interest as (

    select 
        email,
        min(expressed_interest_at) as expressed_interest_at
    from combined_interest
    group by email

)

select * from first_interest
```
## Guidelines

### Use lowercase SQL

It's just as readable as uppercase SQL and you won't have to constantly be holding down a shift key.

```sql
-- Good
select * from user

-- Bad
SELECT * FROM user

-- Bad
Select * From user
```

### Put each selected column on its own line

When selecting columns, always put each column name on its own line and never on the same line as `select`. For multiple columns, it's easier to read when each column is on its own line. And for single columns, it's easier to add additional columns without any reformatting (which you would have to do if the single column name was on the same line as the `select`).

```sql
-- Good
select 
    id
from user 

-- Good
select 
    id,
    email
from user 

-- Bad
select id
from user 

-- Bad
select id, email
from user 
```

### select *

When selecting `*` it's fine to include the `*` next to the `select` and also fine to include the `from` on the same line, assuming no additional complexity like `where` conditions:

```sql
-- Good
select * from user 

-- Good too
select *
from user

-- Bad
select * from user where email = 'name@example.com'
```

### Indenting where conditions

Similarly, conditions should always be spread across multiple lines to maximize readability and make them easier to add to. Operators should be placed at the end of each line:

```sql
-- Good
select *
from user
where 
    email = 'example@domain.com'

-- Good
select *
from user
where 
    email like '%@domain.com' and 
    created_at >= '2021-10-08'

-- Bad
select *
from user
where email = 'example@domain.com'

-- Bad
select *
from user
where 
    email like '%@domain.com' and created_at >= '2021-10-08'

-- Bad
select *
from user
where 
    email like '%@domain.com' 
    and created_at >= '2021-10-08'
```

### Left align SQL keywords

Some IDEs have the ability to automatically format SQL so that the spaces after the SQL keywords are vertically aligned. This is cumbersome to do by hand (and in my opinion harder to read anyway) so I recommend just left aligning all of the keywords:

```sql
-- Good
select 
    id,
    email
from user
where 
    email like '%@gmail.com'

-- Bad
select id, email
  from user
 where email like '%@gmail.com'
```

### Use single quotes

Some SQL dialects like BigQuery support using double quotes, but for most dialects double quotes will wind up referring to column names. For that reason, single quotes are preferable:

```sql
-- Good
select *
from user
where 
    email = 'example@domain.com'

-- Bad
select *
from user
where 
    email = "example@domain.com"
```

If your SQL dialect supports double quoted strings and you prefer them, just make sure to be consistent and not switch between single and double quotes.

### Use `!=` over `<>`

Simply because `!=` reads like "not equal" which is closer to how we'd say it out loud.

```sql
-- Good
select 
    count(*) as paying_user_count
from user
where 
    plan_name != 'free'
```

### Commas should be at the the end of lines

```sql
-- Good
select
    id,
    email
from user

-- Bad
select
    id
    , email
from user
```

While the commas-first style does have some practical advantages (it's easier to spot missing commas and results in cleaner diffs), I'm just not a huge fan of how they look so prefer commas-last.

### Avoid spaces inside of parenthesis

```sql
-- Good
select *
from user
where 
    id in (1, 2)

-- Bad
select *
from user
where 
    id in ( 1, 2 )
```

### Break long lists of `in` values into multiple indented lines

```sql
-- Good
select *
from user
where 
    email in (
        'user-1@example.com',
        'user-2@example.com',
        'user-3@example.com',
        'user-4@example.com'
    )
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

### Column name conventions

* Boolean fields should be prefixed with `is_`, `has_`, or `does_`. For example, `is_customer`, `has_unsubscribed`, etc.
* Date-only fields should be suffixed with `_date`. For example, `report_date`.
* Date+time fields should be suffixed with `_at`. For example, `created_at`, `posted_at`, etc.
* When a unit is relevant, include it in the name. For example, `estimated_hours`, `file_size_kb`, `duration_seconds`, etc.

### Column order conventions

Put the primary key first, followed by foreign keys, then by all other columns. If the table has any system columns (`created_at`, `updated_at`, `is_deleted`, etc.), put those last.

```sql
-- Good
select
    id,
    name,
    created_at
from user

-- Bad
select
    created_at,
    name,
    id
from user
```

### Include `inner` for inner joins

Better to be explicit so that the join type is crystal clear:

```sql
-- Good
select
    user.email,
    sum(charge.amount) as total_revenue
from user
inner join charge on user.id = charge.user_id

-- Bad
select
    user.email,
    sum(charge.amount) as total_revenue
from user
join charge on user.id = charge.user_id
```

### For join conditions, put the table that was referenced first immediately after the `on`

By doing it this way it makes it easier to determine if your join is going to cause the results to fan out:

```sql
-- Good
select
    ...
from user
left join charge on user.id = charge.user_id
-- primary_key = foreign_key --> one-to-many --> fanout
  
select
    ...
from charge
left join user on charge.user_id = user.id
-- foreign_key = primary_key --> many-to-one --> no fanout

-- Bad
select
    ...
from user
left join charge on charge.user_id = user.id
```

### Single join conditions should be on the same line as the join

```sql
-- Good
select
    user.email,
    sum(charge.amount) as total_revenue
from user
inner join charge on user.id = charge.user_id
group by email

-- Bad
select
    user.email,
    sum(charge.amount) as total_revenue
from user
inner join charge
on user.id = charge.user_id
group by email
```

When you have mutliple join conditions, place each one on their own indented line:

```sql
-- Good
select
    user.email,
    sum(charge.amount) as total_revenue
from user
inner join charge on 
    user.id = charge.user_id and
    refunded = false
group by email
```

### Avoid aliasing table names most of the time

It can be tempting to abbreviate table names like `user` to `u` and `charge` to `c`, but it winds up making the SQL less readable:

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

Most of the time you'll want to type out the full table name.

There are two exceptions:

If you you need to join to a table more than once in the same query and need to distinguish each version of it, aliases are necessary.

Also, if you're working with long or ambiguous table names, it can be useful to alias them (but still use meaningful names):

```sql
-- Good: Meaningful table aliases
select
  company.com_name,
  beacon.created_at
from stg_mysql_helpscout__helpscout_company company
inner join stg_mysql_helpscout__helpscout_beacon_v2 beacon on company.com_id = beacon.com_id

-- OK: No table aliases
select
  stg_mysql_helpscout__helpscout_company.com_name,
  stg_mysql_helpscout__helpscout_beacon_v2.created_at
from stg_mysql_helpscout__helpscout_company
inner join stg_mysql_helpscout__helpscout_beacon_v2 on stg_mysql_helpscout__helpscout_company.com_id = stg_mysql_helpscout__helpscout_beacon_v2.com_id

-- Bad: Unclear table aliases
select
  c.com_name,
  b.created_at
from stg_mysql_helpscout__helpscout_company c
inner join stg_mysql_helpscout__helpscout_beacon_v2 b on c.com_id = b.com_id
```

### Include the table when there is a join, but omit it otherwise

When there are no join involved, there's no ambiguity around which table the columns came from so you can leave the table name out:

```sql
-- Good
select
    id,
    name
from company

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
where 
    property_beacon_interest is not null

-- Bad
select
    timestamp_millis(property_beacon_interest)
from hubspot.contact
where
    property_beacon_interest is not null
```

### Be explicit in boolean conditions

```sql
-- Good
select * 
from customer 
where 
    is_cancelled = true

-- Good
select * 
from customer 
where 
    is_cancelled = false

-- Bad
select * 
from customer 
where 
    is_cancelled

-- Bad
select * 
from customer 
where 
    not is_cancelled
```

### Use `as` to alias column names

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

I prefer grouping by name, but grouping by numbers is [also fine](https://blog.getdbt.com/write-better-sql-a-defense-of-group-by-1/).

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
  timestamp_trunc(com_created_at, year) as signup_year,
  count(*) as total_company_count
from company
group by signup_year

-- Bad
select
  timestamp_trunc(com_created_at, year) as signup_year,
  count(*) as total_company_count
from company
group by timestamp_trunc(com_created_at, year)
```

### Grouping columns should go first

```sql
-- Good
select
  timestamp_trunc(com_created_at, year) as signup_year,
  count(*) as total_company_count
from company
group by signup_year

-- Bad
select
  count(*) as total_company_count,
  timestamp_trunc(com_created_at, year) as signup_year
from mysql_helpscout.helpscout_company
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

Avoid subqueries; CTEs will make your queries easier to read and reason about.

When using CTEs, pad the query with new lines. 

If you use any CTEs, always `select *` from the last CTE at the end. That way you can quickly inspect the output of other CTEs used in the query to debug the results.

Closing CTE parentheses should use the same indentation level as `with` and the CTE names.

```sql
-- Good
with ordered_detail as (

    select
        user_id,
        name,
        row_number() over (partition by user_id order by date_updated desc) as detail_rank
    from billingdaddy.billing_stored_detail

),

first_update as (

    select 
        user_id, 
        name
    from ordered_detail
    where 
        detail_rank = 1

)

select * from first_update

-- Bad
select 
    user_id, 
    name
from (
    select
        user_id,
        name,
        row_number() over (partition by user_id order by date_updated desc) as detail_rank
    from billingdaddy.billing_stored_detail
) ranked
where 
    detail_rank = 1
```

### Use meaningful CTE names

```sql
-- Good
with ordered_detail as (

-- Bad
with d1 as (
```

### Window functions

Leave it all on its own line:

```sql
-- Good
select
    user_id,
    name,
    row_number() over (partition by user_id order by date_updated desc) as detail_rank
from billingdaddy.billing_stored_detail

-- Okay
select
    user_id,
    name,
    row_number() over (
        partition by user_id
        order by date_updated desc
    ) as detail_rank
from billingdaddy.billing_stored_detail
```
