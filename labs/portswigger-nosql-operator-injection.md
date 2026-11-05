# PortSwigger: Exploiting NoSQL Operator Injection to Bypass Authentication

**Platform:** PortSwigger Web Security Academy
**Category:** Web exploitation, NoSQL injection
**Tools:** Burp Suite (Repeater)

> Performed on PortSwigger's intentionally vulnerable lab for educational purposes.

## Goal
Log in as the administrator without knowing the password, on a login form backed by a NoSQL database (MongoDB-style JSON queries).

## Walkthrough

### 1. Guessing passwords (didn't work)
I first tried common passwords against the `administrator` username, such as `password` and `admin123`, without Burp Suite. Nothing worked.

### 2. Looking at the request in Burp Repeater
After several failed attempts I sent the login request to **Burp Suite Repeater** to see what the server was receiving. The POST body was **JSON**, which is a hint that NoSQL operators might be accepted.

### 3. First operator attempt (didn't work)
I tried `admin` and `administrator` as the username together with:

```json
"password": { "$ne": null }
```

None of these worked, and I started to think password injection wasn't going to work.

### 4. Matching every record
I tried to return all records by using `$ne` (not equal) on both fields:

```json
{
  "username": { "$ne": null },
  "password": { "$ne": null }
}
```

The server returned **HTTP 500**, an internal server error saying too many records were being returned. This showed the query operators *were* being evaluated.

### 5. Confirming NoSQL injection
I went back to a known account. I changed the username to the given test user, `wiener`, and kept the password as:

```json
"password": { "$ne": null }
```

It **worked** and I was logged in as `wiener`, which confirmed the login is vulnerable to operator injection.

### 6. Targeting the admin with `$regex`
After thinking for a while, I remembered the `$regex` operator, which lets me match usernames **containing** `admin`:

```json
{
  "username": { "$regex": "admin" },
  "password": { "$ne": null }
}
```

The server replied **302 Found**, redirecting to the account page, and set a session cookie.

### 7. Solving the lab
Right-clicking the response and choosing **Open in browser** loaded the site logged in as the admin, and the lab showed **"LAB SOLVED"**.

## Open question I noted
What if there are **several** usernames that contain `admin`? The regex returns whichever account the database matches first, so I would need a tighter pattern (for example an anchored regex like `^admin`) to target a specific account.

## Key takeaways
- When a login sends JSON, test whether the server accepts **query operators** such as `$ne` and `$regex`, not just strings.
- A **500 error** from an operator payload can still be a useful signal: it proves the input reaches the database query.
- Confirm the technique on a known account (`wiener`) before going after the admin.

## How this should be fixed (general guidance)
- Don't pass request JSON straight into the database query. Cast fields to **strings** so objects like `{ "$ne": null }` are rejected.
- Use an allow-list of accepted fields and types, and strip or block `$`-prefixed operators from user input.
- Add rate limiting and lockouts on login endpoints.
