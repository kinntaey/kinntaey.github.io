---
title: "[Study Note] File Upload Vulnerabilities"
date: 2026-03-20 10:27:00 +1100
categories: [Study Note, Web]
tags: [web_security, file_upload, rce, web_shell, bypass]
---

# File Upload Vulnerabilities

> **Purpose**: File upload functionality is one of the most common features in web applications — and one of the most dangerous when implemented carelessly. This note covers how these vulnerabilities work, how attackers exploit them, and how to defend against them.

---

## 1. What Are File Upload Vulnerabilities?

File upload vulnerabilities occur when a server accepts user-uploaded files without properly validating their **name**, **type**, **contents**, or **size**. Even a seemingly harmless image upload feature can become a critical attack vector if left unguarded.

An attack typically unfolds in two stages:

1. **Upload a malicious file** — get a foothold on the server
2. **Trigger its execution** — send a follow-up HTTP request to cause real damage

---

## 2. Impact

The severity depends on two key factors:

- **What the server fails to validate** — file type, name, contents, or size
- **What happens to the file after upload** — whether it ends up in an executable location

| Scenario | Impact |
|---|---|
| Server-side script execution enabled | Remote Code Execution (RCE), full server takeover |
| Filename not validated | Overwrite critical files (e.g. config files) |
| File size not validated | Disk exhaustion / Denial of Service (DoS) |
| Malicious HTML/SVG upload | Stored XSS |
| XML-based file parsing | XXE Injection |

---

## 3. Web Shells

> The primary weapon in file upload attacks.

A **web shell** is a malicious script that lets an attacker execute arbitrary commands on a remote server by simply sending HTTP requests.

**Basic web shell — read a file:**

```php
<?php echo file_get_contents('/path/to/target/file'); ?>
```

Once uploaded and requested, the server returns the target file's contents in the HTTP response.

**More versatile web shell — run any command:**

```php
<?php echo system($_GET['command']); ?>
```

This allows arbitrary system commands via a query parameter:

```
GET /uploads/shell.php?command=id HTTP/1.1
```

A successfully uploaded web shell gives the attacker the ability to read/write files, exfiltrate sensitive data, and pivot into internal infrastructure.

---

## 4. How Web Servers Handle Files

Understanding file execution is key to understanding these vulnerabilities.

When a request comes in, the server maps the file extension to a MIME type using a preconfigured table, then decides what to do:

- **Non-executable file** (image, static HTML): content is served directly to the client
- **Executable file + execution configured** (e.g. `.php`): the script runs on the server; output is returned
- **Executable file + execution NOT configured**: error response, or source code may leak as plain text

> The `Content-Type` response header can be a useful clue — if not explicitly set by the app, it reflects the server's file extension → MIME type mapping.

---

## 5. Bypassing File Upload Defenses

### 5-1. Content-Type Header Spoofing

If the server only checks the `Content-Type` header to determine file type, this is trivially bypassable — the client controls that value.

```
Content-Disposition: form-data; name="image"; filename="shell.php"
Content-Type: image/jpeg    ← change this to bypass the check
```

Using a proxy tool like Burp Suite, simply changing `Content-Type` to `image/jpeg` while keeping the `.php` payload will often slip past this validation.

---

### 5-2. Blacklist Bypass (Alternative Extensions)

Blocking only `.php` is inherently flawed — there are too many alternative extensions that may still be executable:

- `.php5`, `.php7`, `.phtml`, `.pht`
- `.shtml` (enables Server-Side Includes)
- `.asp`, `.cer`, `.asa` (IIS environments)

**Case manipulation:**
```
shell.pHp   →   passes case-sensitive blacklist, server executes it anyway
```

**Double extensions:**
- Apache: `file.php.jpg` may be processed as PHP depending on configuration
- IIS 6: `file.asp;.jpg` — the `;` causes the server to ignore the rest

---

### 5-3. Extension Obfuscation

The goal here is to pass validation while still being treated as an executable extension at the server level.

| Technique | Example |
|---|---|
| Trailing dot/space (Windows) | `shell.php.` |
| URL encoding | `shell%2Ephp` |
| Null byte injection | `shell.php%00.jpg` |
| Nested forbidden string | `shell.p.phphp` → strip `.php` → `shell.php` remains |
| Unicode conversion | `xC0 x2E` may convert to `.` during normalization |

> **Null byte trick**: C/C++ functions treat null bytes as string terminators. If a high-level language (PHP/Java) performs validation but a lower-level function handles the actual file save, the filename may be truncated at the null byte — effectively removing the fake safe extension.

---

### 5-4. Overriding Server Configuration

Apache servers load per-directory configuration from `.htaccess` files. If uploading one is allowed, an attacker can redefine how the server handles file types:

```apache
AddType application/x-httpd-php .jpg
```

After uploading this, all `.jpg` files in that directory will be executed as PHP.

IIS servers have an equivalent mechanism via `web.config`.

---

### 5-5. Bypassing Magic Bytes / Content Validation

More robust servers check the actual file content (magic bytes / file signature) rather than just the extension. For example, JPEG files always begin with `FF D8 FF`.

However, tools like **ExifTool** make it easy to create **polyglot files** — valid images that also contain embedded malicious code in their metadata:

```bash
# Embed PHP code inside a GIF comment using gifsicle
gifsicle < original.gif --comment "<?php system($_GET['cmd']); ?>" > shell.php.gif
```

Functions like PHP's `getimagesize()` only verify the image dimensions and header — they don't inspect comments or metadata sections, making this bypass viable.

---

### 5-6. Race Condition Attacks

Some servers upload a file to a temporary location first, validate it, and then delete or move it. The attack targets the narrow window between upload and deletion.

```
Upload file → Temporarily stored (a few ms) → Validation fails → Deleted
                        ↑
           Send execution request during this window
```

For URL-based uploads, if the temporary directory name is generated with a predictable function like PHP's `uniqid()`, the path can potentially be brute-forced.

Uploading a **larger file** can extend this window by forcing chunk-by-chunk processing, giving more time to trigger execution.

---

### 5-7. Uploading via PUT Method

Some servers support `PUT` requests, which can allow file uploads entirely outside of the web UI:

```http
PUT /uploads/shell.php HTTP/1.1
Host: target.com
Content-Type: application/x-httpd-php

<?php echo file_get_contents('/etc/passwd'); ?>
```

Send an `OPTIONS` request to check if the target server advertises support for `PUT`.

---

### 5-8. Attacks Without RCE

Even when server-side execution isn't possible, file upload vulnerabilities can still be exploited:

- **Stored XSS**: upload an HTML or SVG file containing `<script>` tags — executes in other users' browsers
- **XXE Injection**: upload `.docx` or `.xls` files to exploit XML parsing vulnerabilities
- **Cross-domain hijacking**: upload `crossdomain.xml` or `clientaccesspolicy.xml` to manipulate Flash/Silverlight access policies
- **Malware distribution**: upload trojaned executables or virus-infected files for other users to download

---

## 6. Defense

### ✅ File Validation

- Use an **allowlist** for file extensions — don't rely on blocklists
- Validate the actual file content (magic bytes), not just the `Content-Type` header
- Strip directory traversal sequences (`../`), null bytes, and special characters from filenames
- Limit filename length (< 255 characters on NTFS)
- Enforce both minimum and maximum file size limits

### ✅ File Storage

- Never grant **execute permissions** to upload directories
- Rename uploaded files server-side (e.g. hash-based names) — never use the user-supplied filename directly
- Store files in a **temporary sandboxed directory** and only move them to their final location after passing all validation
- Serve uploaded files from a **separate domain or server** if possible
- Consider storing files in a database rather than the filesystem

### ✅ Server Configuration

- Prevent `.htaccess` and `web.config` from being uploaded; configure servers to ignore them in upload directories
- Ensure double-extension files (e.g. `file.php.jpg`) cannot be executed
- Add `Content-Disposition: Attachment` and `X-Content-Type-Options: nosniff` headers to static file responses

### ✅ Access Control & Operations

- Restrict file upload functionality to authenticated and authorized users
- Implement CSRF protection on upload endpoints
- Run a virus scanner on the server
- Log all upload activity
- Use a well-tested framework for file handling rather than writing custom validation from scratch

---

## 7. Upload Request Validation Flow

```
Incoming upload request
        │
        ├─ Filename check     → Strip special chars, traversal sequences; regenerate server-side
        │
        ├─ Extension check    → Compare against allowlist (case-insensitive)
        │
        ├─ Content-Type check → Use as a hint only; never trust alone
        │
        ├─ Magic bytes check  → Verify actual file header bytes
        │
        ├─ File size check    → Enforce min/max limits
        │
        └─ Temporary storage → Move to final path only after all checks pass
                               (final directory has no execute permissions)
```

---

## 8. Related Vulnerabilities

- [Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal)
- [Stored XSS](https://owasp.org/www-community/attacks/xss/)
- [XXE Injection](https://portswigger.net/web-security/xxe)
- [Race Conditions](https://portswigger.net/web-security/race-conditions)

---

## References

- OWASP – Unrestricted File Upload: https://owasp.org/www-community/vulnerabilities/Unrestricted_File_Upload
- PortSwigger Web Security Academy – File Upload Vulnerabilities: https://portswigger.net/web-security/file-upload
