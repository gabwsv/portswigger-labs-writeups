### Lab Description

This lab reflects your input in a JavaScript URL, but all is not as it seems. This initially seems like a trivial challenge; however, the application is blocking some characters in an attempt to prevent XSS attacks.

To solve the lab, perform a cross-site scripting attack that calls the `alert` function with the string `1337` contained somewhere in the `alert` message.

https://portswigger.net/web-security/cross-site-scripting/contexts/lab-javascript-url-some-characters-blocked

---
## Introduction

This lab features a reflected XSS vulnerability within a `javascript:` URL. To solve it, a sophisticated JavaScript payload is required due to heavy filtering. The technique is uncommon and advanced, but extremely functional.  

In this write-up, we will deconstruct the payload used to solve the lab piece by piece

---
## 1. Reconnaissance

The first step is to identify the injection vector. Since the vulnerability is a reflected XSS in the URL, the goal is to find where our input is reflected in the page's source code.

This application is a blog. After navigating to a post, the URL is:
`https://[...].web-security-academy.net/post?postId=3`

I searched the page source for the string "postId=3" or URL encoded "postId%3d3", which led me to an ``<a>`` tag containingh the "Back the blog" link. The `href` attribute immediately stood out:  

```html
<a href="javascript:fetch('/analytics', {method:'post',body:'/post%3fpostId%3d3'}).finally(_ => window.location = '/')">Back to blog</a>
```
The `href` attribute uses the `javascript:` URI scheme, and our input from the `postId` parameter is reflected inside the `body` property of the `fetch()` call. This is our injection point.

---

## 2. Deconstructing the Exploit and Bypass

The payload needs to escape multiple contexts (HTML attribute -> JavaScript URI -> JavaScript string) while bypassing character filters. The final payload is:

`&'},x=x=>{throw/**/onerror=alert,1337},toString=x,window+'',{x:'`

Let's break down how it's constructed.

1. **The Breakout (`&'},`):**
	- `&`: As discovered during testing, the injection must start with an ampersand to be treated as a new parameter within the `body` string's URL, preventing an immediate error.
	- `'`: This single quote closes the string literal for the `body` property.
	- `}`: This curly brace closes the configuration object for the ``fetch`` function.
	- `,`: This is the core of the bypass. Instead of trying to close the `fetch()` call (which is blocked by the WAF), we use a comma. This tells the JavaScript engine that we are providing another argument to the `fetch` function. The function itself will ingnore these extra arguments, but the engine will still evaluate them.
2. **The Malicious Function (`x=x=>{throw/**/onerror=alert,1337}`):**
	- This is the first "fake" argument. It's an arrow function assigned to the variable `x`.
	- Inside the function, it uses the lab's intended `throw/onerror` technique. The expression ``onerror=alert,1337`` uses the comma operator to first assign `alert` to the global onerror handler and then return the value `1337`, which is then thrown.
	- The `/**/` is a comment used as a space to bypass WAFs that might look for the exact string "throw onerror".
3. **The Hijack (`toString=x`):**
	- This is the second "fake" argument. It hijaks the global `toString` method and replaces it with our malicious function `x`. This is the most brillant part of the exploit. When the browser encouters a syntax error later, it will try to call `toString()` on our function to create an error message, which will inadvertently execute our code.
4. **Forcing the Error (`window+'',{x:'`):**
	- This is the final injected "argument". It is intentionally malformed JavaScript syntax. When the JavaScript engine tries to evaluate this, it fails and throws a `SyntaxError`.

Tip: In some cases, spaces or "throw event" is blocked the WAF, use the `/**/`, `/`, `%0a` or `%09` to bypass, in this lab, used the `/**/`

---
## Execution Flow

When the "Back to blog" link is clicked:
1. The browser starts executing the JavaScript in the `href`
2. It prepares the `fetch` call, evaluating the arguments one by one.
3. It evaluates our injected arguments, defining the function `x` and hijacking `toString`.
4. It hist the intentionally broken syntax and throws a `SyntaxError`.
5. To handle the error, the browser engine calls `toString()` on our function `x`.
6. Because we hijacked `toString`, this execute our function `x`.
7. Our function `x` runs, setting `onerror=alert` and throwing `1337`.
8. The ``onerror`` handler (`alert`) catches the value `1337`, and the `alert(1337)` box appears, solving the lab.

Click in "Back to Blog" to execute code.

<img width="505" height="217" alt="Pasted image 20250828203650" src="https://github.com/user-attachments/assets/65f1dca2-e8ac-4030-9897-d783ef4ecffa" />

---

## Conclusion

Solving this lab is extremely challenging and requires a deep understanding of JavaScript syntax, browser behavior, and common WAF bypass techniques. The key takeways are:
- **Abuse the Language:** The exploit works by abusing obscure JavaScript features like the comma operator and overriding the `toString` method.
- **Chain of Events:** The solution isn't a single payload but a chain reaction: a breakout causes a syntax error, which triggers a hijacked function, which finally executes the intended XSS.
- **Context is Everything:** Understanding that the injection point was inside a string, inside an object, inside a function call was crucial to building the correct breakout sequence.

I learned about an advanced payload that uses a combination of bypasses and javascript features like `,` for introduce a new variables into the code block and use the `throw` handler to call an `alert` event, as well as purposefully breaking the syntax to trigger the throw.


