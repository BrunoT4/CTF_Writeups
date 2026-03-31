# picoCTF 2026 — ORDER ORDER

**Category:** Web Exploitation

---

## The App

The challenge gives you a hosted expense tracker. You can sign up, add expenses (description, amount, date), and hit a "Generate Report" button that compiles your expenses into a CSV and drops it in your inbox.

Pretty simple surface area at first glance.

---

## Initial Recon

The first thing I saw upon navigating to the site was a login/signup page. Naturally I tried the usual: ' OR '1'='1, admin';. Nothing. Either it's parameterized or there's some sanitization happening up front, but either way the login form wasn't biting.
So I made an account and started poking around the actual app. I observed that expense submissions go out as a JSON payload over an API, all three fields get inserted as strings, and inserts are handled entirely server-side. No obvious injection surface there either.

What did catch my eye was the generated CSV filename. It was named dynamically after the logged-in user, in my case something like report_wasd_{random # string}.csv (not a very creative name, but it got the job done). That told me the username was flowing into backend processing somewhere downstream, and that's a different story from the insert. If the report query is pulling expenses by matching on the username string rather than a user ID, and that username isn't sanitized at query time, then the username I was using to register the account could very well be an attack surface. All of a sudden the problem title made a lot more sense. This must be a second order injection.

---

## Getting Past the Validator

There's a uniqueness check on signup that blocks exact duplicate usernames. From my initial recon, I knew there was no string filter for SQL metacharacters. You can register with `'`, `' OR '1'='1`, full injection strings, pretty much whatever you want. This is a comical amount of free reign for constructing my injection string.

I registered with `' OR '1'='1` as my username, generated a report, and the CSV came back with my previous account's expense data in it. So there is in fact an existing second-order injection.

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

It seems this table exclusively contains expense data, so the next place to check is whether this database contains any other tables.

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

The `reports` table stores `username TEXT` instead of a `user_id` foreign key. That's the root cause: the report query is string-matching on username rather than joining through the user's primary key. That's what makes the injection possible in the first place.

The other thing was that strange table name: `aDNyM19uMF9mMTRn`. That's clearly programmatically generated, which means it's probably hiding some sort of secret. Obviously this table seemed a little off, so I wanted to prioritize investigating it.

---

## Getting the Flag

I registered with:

```sql
' UNION SELECT group_concat(name||':'||value,'|'), NULL, NULL FROM aDNyM19uMF9mMTRn--
```

Upon registration I generated the report (no need to add expenses as the injection is already complete). The CSV came back with all the key-value pairs from that table dumped into the description column. Sure enough, one of the cells in that table contained the flag.

---

**Flag:** `picoCTF{s3c0nd_0rd3r_1t_1s_5c47e96a}`
