---
title: "[Dreamhack] image-storage"
date: 2026-03-21 15:00:00 +1100
categories: [CTF Writeup, Web]
tags: [web_security, file_upload, rce, web_shell, php]
---


## Overview

The application provides a file upload feature with no restriction on file type or extension. Since uploaded files are stored in a web-accessible directory, a PHP file can be uploaded and executed directly through the browser.

---

## Solution

### Step 1 — Identify the Vulnerability

The upload endpoint accepts any file without validating the extension or MIME type. This means a `.php` script will be stored on the server as-is, and if the upload directory is served by the web server, it will be executed when accessed via HTTP.

### Step 2 — Create the Payload

Created a minimal PHP file that reads the flag:

```php
<?php echo file_get_contents('/flag.txt'); ?>
```

Saved as `attack.php`.

### Step 3 — Upload and Locate the File

After uploading `attack.php`, the server responded with the storage path:

```
Stored in: ./uploads/attack.php
```

This confirms the file landed in the `/uploads/` directory, which is directly accessible from the web root.

### Step 4 — Execute the Payload

Navigated to the uploaded file:

```
http://host3.dreamhack.games:21296/uploads/attack.php
```

The server executed the PHP script and returned the contents of `/flag.txt`.

---

## Flag

```
DH{c29f44ea17b29d8b76001f32e8997bab}
```

---

## Key Takeaway

This challenge demonstrates the simplest form of an unrestricted file upload vulnerability. The two conditions that made it exploitable were:

1. **No file type validation** — the server accepted `.php` files without restriction
2. **Web-accessible upload directory** — files in `/uploads/` were served and executed by the PHP runtime

A real-world fix would be to either restrict uploads to safe extensions (allowlist), disable script execution in the upload directory, or store uploaded files outside the web root entirely.
