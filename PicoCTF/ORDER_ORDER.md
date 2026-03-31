# picoCTF 2026 — ORDER ORDER

**Category:** Web Exploitation

---

## The App

The challenge gives you a hosted expense tracker. You can sign up, add expenses (description, amount, date), and hit a "Generate Report" button that compiles your expenses into a CSV and drops it in your inbox.

Pretty simple surface area at first glance.

---

## Initial Recon

The first thing I noticed was that expense submissions go out as a JSON payload over an API. All three fields just get inserted as strings. The insert itself is handled entirely server-side so there's no obvious injection point there.

What *did* catch my eye was the generated CSV filename. It was named dynamically after the logged-in user. Something like `johndoe_report.csv`. That tells you the username is flowing into backend processing somewhere, which is worth keeping in mind.

The other thing I noticed was the app throws verbose errors when report generation fails. Not a vulnerability on its own but it will be useful later.

---

## The Hypothesis

My guess at the report pipeline is that the server runs a query along the lines of:

```sql
SELECT description, amount, date FROM expenses WHERE username = '<username>'
```

If that username is coming straight from wherever it's stored at registration (no parameterization) then the injection doesn't fire at insert time, it fires later when you generate a report. That's a second-order SQLi. All of a sudden the problem title makes sense. The attack surface isn't the expense fields, it's the **username** field at signup.

---

## Getting Past the Validator

There's a uniqueness check on signup that blocks exact duplicate usernames. I tested whether it'd block SQL metacharacters too. It didn't. You can register with `'`, `' OR '1'='1`, full injection strings, whatever.

I registered with `' OR '1'='1` as my username, generated a report, and the CSV came back with another user's expense data in it (an account I had made previously in the same session). So there is in fact an existing second-order injection.

---

## Figuring Out the Column Count

To do a UNION-based attack I needed the column count. I registered these usernames one by one and generated a report each time:

```
' UNION SELECT NULL--              → error
' UNION SELECT NULL,NULL--         → error
' UNION SELECT NULL,NULL,NULL--    → clean CSV ✓
' UNION SELECT NULL,NULL,NULL,NULL-- → error
```

The table has three columns: description, amount, date. Makes sense considering the expense adder has these three fields.

I also tried stacking queries with a semicolon, and got back:

> *"Report generation failed: You can only execute one statement at a time."*

So stacked queries are out. Single-statement UNION reads only, still more than enough.

---

## Dumping the Schema

SQLite doesn't have `information_schema` but it has `sqlite_master` which gives you the full schema. This time I registered with:

```sql
' UNION SELECT name, sql, NULL FROM sqlite_master--
```

When I generated the report, I got back the entire database structure:

| Table | Notable columns |
|---|---|
| `users` | id, username (UNIQUE), email, password |
| `expenses` | id, user_id (FK → users), description, amount, date |
| `reports` | id, **username TEXT**, status, created_at |
| `inbox` | id, username, description, report_path, status, created_at |
| `aDNyM19uMF9mMTRn` | name TEXT PRIMARY KEY, value TEXT |

A couple things jumped out immediately.

The `reports` table stores `username TEXT` instead of a `user_id` foreign key. That's the root cause: the report query is string-matching on username rather than joining through the users primary key. That's what makes the injection possible in the first place.

The other thing was that strange table name: `aDNyM19uMF9mMTRn`. That's clearly programmatically generated, which usually means it's hiding something like API keys, config values, or secrets.

---

## Getting the Flag

I registered with:

```sql
' UNION SELECT group_concat(name||':'||value,'|'), NULL, NULL FROM aDNyM19uMF9mMTRn--
```

I opted to check what was in the strangely named table first, since intuition told me it must contain something relevant. Upon registration I generated the report (no need to add expenses as the injection is already complete). The CSV came back with all the key-value pairs from that table dumped into the description column. To no surprise, one of the column names in that table was the flag.

---

**Flag:** `picoCTF{s3c0nd_0rd3r_1t_1s_5c47e96a}`
