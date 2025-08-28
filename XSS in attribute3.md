
## **Reflected XSS in canonical link tag**

The description lab:

This lab reflects user input in a canonical link tag and escapes angle brackets.
To solve the lab, perform a cross-site scripting attack on the home page that injects an attribute that calls the `alert` function
To assist with your exploit, you can assume that the simulated user will press the following key combinations:
- ALT+SHIFT+X
- CTRL+ALT+X
- Alt+X

Solution this lab is only possible in Chrome.

https://portswigger.net/web-security/cross-site-scripting/contexts/lab-canonical-link-tag

---

## Introduction.

The lab consists of executing a XSS using the reflection in tag `<link rel"canonical">`, the angle breackets is escaped, but quotes not. The solutions use a creative tenchique using `accesskey` attribute to called payload.

More informations for technique in [XSS in Canonical tags](https://portswigger.net/research/xss-in-hidden-input-fields)

#### 1. Step: Reconnaissance and Analysis of the Reflection.

A first step, with never, is understend the application work. Using the devtools on the browser, I inspected the code page and I found the canonical tag

```html
	<link rel="canonical" href="https://id_lab.web-security-academy.net/">
```
![[Pasted image 20250827195951.png]]

The purpose of the "Canonical" tag is to indicate to search engines,  what URL is preferer to an page, eviting duplicate content problems.

example: 
In your website, the content in this page: www\.example\.com/produt?color=blue and page www\.example\.com/produt?color=red is the identic. Using the canonical link to refer a main page.

Test the Reflected, adding a any parameter in URL, with `?a`
![[Pasted image 20250827200605.png]]

The value of parameter sending, is reflected in the attribute ``href``. This is a promising vector for attribute injects in HTML

---

#### 2. Step. Try exploit and bypass filters

With the reflection point identified , the next step ir try to inject a new attribute that can execute JavaScript. The first attempt, is a manipulate event with ``onfocus``, combinated with the `accesskey`, which is a tip for description lab.

###### Payload 1 (failed):

	`" accesskey="X" onfocus="alert()`

The idea, is a closing the attribute `href` with the double quotes and the inject a new attributes. However, the application encode the double quotes to `%22`, making every payload belong to the `href` attribute , do no break context.

![[Pasted image 20250827201022.png]]

Note: the spaces also encoded.
###### Payload 2:

`?a'accesskey='X'onfocus='alert()`

Is the jump to cat. I changed double quotes to single quotes. The application wasn't expecting the single quotes, what allowed the closed `href` attribute and inject my attributes with success.

![[Pasted image 20250827201419.png]]

#### 3. Step: The Final Payload and Execution

Despite having managed to close the `href` attribute and inject success, the event `onfocus` not called with the accesskey. The attribute `accesskey="X"` allows an element to be focused or activated using an keywork combinations (with ALT+SHIFT+X)

The most direct alternatively to `accesskey` is `onclick` event. If `accesskey` "click" on element, the `onclick` so fired.

**Final Payload:**
``?a'accesskey='X'onclick='alert()'``

![[Pasted image 20250827201706.png]]

Press the ALT+SHIFT+X, the attribute `accesskey` is active, triggering the `onclick` event and execute `alert()` function. Resolved the lab. 

---
### Conclusion.

This lab demonstrate important concepts to a pentester:
- **Context is King**: Exploration was only possible why the reflection occurs in the attribute HTML.  Understanding the context is the first step to creating an effective payload
- **Filters are challenge, not barriers:** A filter blocked < and > or escapes double quotes, doesn't mean the application is secure. Many developers often forget to handle all cases, such as single quotes.
- **Unusual Attributes are Powerful Vectors:**  Attributes with `accesskey` are less common, but they can be the key to exploiting an XSS vulnerability when more obvious events ( like onload in an image tag) are not possible

I this lab, discovered the advanced techniques for explore the XSS in canonical tags using the accesskey. And note the some cases double quotes don't work, but single quotes do. Understanding the context is the first step to creating an effective payload.


