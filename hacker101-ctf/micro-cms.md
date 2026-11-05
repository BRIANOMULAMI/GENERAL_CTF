# Hacker101 CTF — Micro-CMS Write-up

## Challenge Overview

**Platform:** Hacker101 CTF
**Target:** Micro-CMS
**Challenge URL:** `https://39f0614aa4e690f07a9a8f0fa505c69d.ctf.hacker101.com/`

This challenge involved exploring a web-based Content Management System (CMS) to identify security weaknesses involving access control and stored Cross-Site Scripting (XSS).

The investigation focused on page enumeration, authorization checks, and testing the CMS's Title and Body input fields.

---

## 1. Broken Access Control — Private Page Disclosure

### Discovery

During initial enumeration, I identified several accessible CMS pages:

- `/page/1`
- `/page/2`
- `/page/9`

Further enumeration revealed that `/page/5` returned **403 Forbidden** when accessed directly.

However, testing the corresponding edit endpoint revealed an authorization weakness.

### Exploitation

Although the following endpoint denied access:

```text
/page/5
```

The edit endpoint was accessible:

```text
/page/edit/5
```

This exposed the contents of a page titled **Private Page**, including a CTF flag.

### Vulnerability

Broken Access Control / Insecure Direct Object Reference (IDOR).

The application enforced access restrictions inconsistently between the page-viewing and editing routes.

### Impact

An unauthorized visitor could retrieve content from a page that was supposed to be private.

**Flag:** `[REDACTED]`

---

## 2. Stored Cross-Site Scripting — Title Field

### Discovery

The CMS supported Markdown and allowed page creation through a form containing:

- Title
- Body

Testing revealed that the Title field did not properly escape injected HTML and JavaScript.

### Exploitation

A JavaScript payload was submitted through the Title input:

```html
<script>alert(1)</script>
```

After saving the page, the payload remained stored.

When returning to the homepage, where page titles were displayed, the injected JavaScript executed.

The challenge displayed a flag in the resulting popup.

### Vulnerability

Stored Cross-Site Scripting (Stored XSS).

The application failed to properly sanitize or encode user-controlled Title content before rendering it.

### Impact

An attacker could potentially execute JavaScript in the browsers of users visiting the affected page listing.

**Flag:** `[REDACTED]`

---

## 3. Stored Cross-Site Scripting — Body Field

### Discovery

The Body field behaved differently from the Title field.

Standard script tags were filtered, suggesting that the CMS applied additional filtering to Body content.

However, alternative HTML-based injection techniques could still be tested.

### Exploitation

An alternative payload using an image error event handler was tested:

```html
<img src=x onerror="alert('Body XSS confirmed')">
```

The payload was saved in the Body field and the resulting page was opened to test whether the event handler executed.

This demonstrated a potential filtering bypass through an HTML event handler rather than a standard script tag.

The same flag value was observed during the Title and Body XSS tests.

### Vulnerability

Stored Cross-Site Scripting (Stored XSS) through the Body field.

The filtering mechanism did not sufficiently prevent executable HTML event handlers.

### Impact

If an attacker can store executable HTML in page content, visitors viewing that content may execute attacker-controlled JavaScript.

**Flag:** `[REDACTED]`

---

## Lessons Learned

This challenge reinforced several important web application security concepts:

1. **Access control must be enforced consistently:** Protecting a viewing route does not help if another route exposes the same restricted information.
2. **Stored XSS can occur in multiple input fields:** Both page titles and page bodies can become injection points.
3. **Filtering script tags is insufficient:** HTML event handlers can provide alternative execution paths.
4. **Output encoding matters:** User-controlled content should be safely encoded according to its output context.
5. **Test authorization and input handling across all relevant endpoints:** A single successful or blocked request does not establish that the entire application is secure.

## Conclusion

The investigation identified a broken access control vulnerability and two Stored XSS injection points in the Micro-CMS application.

The exercise provided practical experience with endpoint enumeration, authorization testing, HTML injection, and identifying weaknesses in input filtering.

*All flag values have been redacted for responsible public documentation.*







3