---
author_name: Ryan Miller
author_twitter: andryanmiller
excerpt: CORS (Compartilhamento de Recursos de Origem Cruzada) é um assunto um tanto quanto obscuro para muitos desenvolvedores web. Como lendas de míticos monstros marinhos, todos desenvolvedor tem um história para contar sobre quando o CORS se apoderou de seus requests, levando-os para profundezas inexoráveis, e nunca mais foram vistos.
hero: /public/svg/sailor-outline.svg
language: pt
language_alternates: ['fr', 'en', 'ru']
og_image: /public/img/cors_og_image.png
published: true
slug: cors
title: CORS
translator_name: Heitor Figueiredo
translator_website: http://heitorfig.com
---
CORS (Compartilhamento de Recursos de Origem Cruzada) é um assunto um tanto quanto obscuro para muitos desenvolvedores web. Como lendas de míticos monstros marinhos, todos desenvolvedor tem um história para contar sobre quando o CORS se apoderou de seus requests, levando-os para profundezas inexoráveis, e nunca mais foram vistos.

<figure>
  <img src="/public/svg/kraken-outline.svg" width="100%" style="max-width: 50rem"/>
  <figcaption>"No 'Access-Control-Allow-Origin' header is present on the requested resource. Origin 'https://example.com' is therefore not allowed access."</figcaption>
</figure>

Seja buscando um pedaço de JSON, ou tentando configurar um CDN para seus assets, o CORS parece se tornar um fardo nos piores momentos. E assim, os desenvolvedores aprenderam a reprimir o CORS, permitindo que ele ganhasse uma reputação de incômodo que, de algum jeito, torna nossos usuários mais seguros.

Esse post busca desmistificar o CORS e mostrar seus pontos positivos, como uma especificação que não vem para atrapalhar a vida dos desenvolvedores, mas sim livrar-nos das garras do _same-origin policy_ (política de mesma origem). Vamos passar por cada cabeçalho necessário para safisfazer da forma certa as normas do CORS, e também discutir alguns lugares onde o CORS agora é relevante, mas que podem te surpreender.

### Uma Breve História do CORS

CORS, ou a idéia que se tornaria o CORS, nasceu na [era da Web 2.0](https://www.oreilly.com/pub/a/web2/archive/what-is-web-20.html?){:target="_blank"}, em meados de 2005. Uma das principais *buzzwords* da Web 2.0 foi [AJAX](http://adaptivepath.org/ideas/ajax-new-approach-web-applications/){:target="_blank"}, ou "Asynchronous JavaScript and XML" (Javascript e XML Assícronos), e ela trazia a idéia que era possível usar a API do [XMLHttpRequest](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest){:target="_blank"} para, de forma assíncrona, atualizar uma página sem recarregá-la de forma completa.

Porém, quando o XMLHttpRequest apareceu ele possuia uma limitação significante: você apenas podia usar sua API para comunicar com serviços que estivessem no mesmo domínio que o site que os estava chamando. Isso significa que se seu site estivesse em `https://euadoroajax.com.br/`, e você quisesse fazer uma requisição para `https://resourceexterno.com.br/` (ou até `https://subdominio.euadoroajax.com.br/`) o navegador simplesmente se recusaria a iniciar a requisição. Isso é chamado de [*same-origin policy*](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy){:target="_blank"}.

Conforme o AJAX foi ganhando força ficou aparente que algo deveria ser feito a respeito do XMLHttpRequest e sua camisa de força, chamada *same-origin policy*. A comunidade de desenvolvimento web viu como liberar o AJAX para outros domínios poderia favorecer novos serviços e formas de usar a web, o que (alerta de spoiler) aconteceu, exemplos são: Firebase, Mixpanel, New Relic e mais. Nessa mesma época (2005) pessoas começaram a trapacear o sistema usando algo chamado [JSONP](https://en.wikipedia.org/wiki/JSONP){:target="_blank"}, o que basicamente "sequestrava" a tag `<script>` (e sua política de segurança de recursos extremante fraca) para obter dados de serviços remotos.

Em 2005 [o primeiro rascunho](https://www.w3.org/TR/2005/NOTE-access-control-20050613/){:target="_blank"} do que seria a especificação do CORS foi publicado. Mas somente em 2007 que aspectos significantes da especificação começariam a tomar forma, como o mecanismo de "preflight" e o uso de cabeçalhos do HTTP vs. markup do XML, e outros sete anos depois se tornaria uma [recomendação da W3C](https://www.w3.org/TR/2014/REC-cors-20140116/){:target="_blank"}. Enquanto isso, o navegadores já haviam começado a implementar partes mais estáveis da especificação.

Escrever especificações não é uma tarefa fácil, mas não se sinta culpado em indagar o porque isso levou uma década. Quando você considera as implicações sobre segurança que envolvem o CORS, no entanto, isso faz um pouco mais de sentido. A principal preocupação era o fato de que a maioria, senão todos, *web services* esperavam que requisições *non-GET* fossem originadas de domínios específicos (normalmente pertencendo aos mesmos proprietários do serviço em questão, dado que a *same-origin policy* ainda é a lei que regia o reino). No entanto, se o CORS fosse implementado, e a *same-origin policy* fosse suprimida, esses serviços poderiam então receber várias um dilúvio de requisições de DELETE, PUT, etc... de qualquer origem, e não fazia sentido esperar que todos os *web services* voltados ao público se adaptassem ao CORS antes de sua recomendação pelo W3C.

Então a decisão foi fazer o CORS *opt-in* (opcional de utilização), o que significa que os navegadores continuariam a utilizar a *same-origin policy* a não ser que uma série de sinais fossem enviados por um *web service* dizendo que era permitido fornecer o conteúdo para origens distintas. Nós vamos discutir as especificidades desse mecanismo, chamado *preflighting*, logo mais. Montando essa funcionalidade *opt-in* no design do CORS fez com que os *web services* não precisariam estar expostos a uma torrente de requisições inesperadas, e os desenvolvedores pudessem começar a desenvolver novos tipos de serviços e ferramentas.

Você pode nunca se entusiasmar com o CORS, mas se há uma coisa pela qual todos devemos ter um pouco de gratidão, é o fato de que o CORS equilibra tanto a compatibilidade com versões anteriores quanto a abertura de uma enorme faixa de novas funcionalidades para desenvolvedores. Não é um feito fácil! E para demonstrar melhor a realização que é isso, vamos ver como o CORS pode afetar suas requisições e como você pode evitar alguns [*gotchas*](https://en.wikipedia.org/wiki/Gotcha_(programming)){:target="_blank"} mais sutis.

<figure>
  <img src="/public/svg/seagull-outline.svg" width="100%" style="max-width: 30rem"/>
</figure>

### Preflighting

Provavelmente o aspecto mais desconcertante do CORS é o uso de requisições de *preflight*. Imagine que você iniciou a seguinte requisição de *cross-domain* como *POST* para atualizar o perfil de um usuário:

<pre><code>POST https://api.users.com/me HTTP/1.1
Host: example.com
Content-Type: application/json; charset=utf-8

{
  "name": "Burt Macklin",
  "description": "Parece um trabalho para Burt Macklin!"
}</code></pre>

Se você fez essa requisição em um navegador que implementa CORS, você verá que o navegador envia primeiro a seguinte requisição:

<pre><code>OPTIONS https://api.users.com/me HTTP/1.1
Host: example.com
Access-Control-Request-Headers: content-type
Access-Control-Request-Method: POST
Origin: https://example.com</code></pre>

Espera um pouco. O que está acontecento? Essa primeira requisição `OPTIONS` é chamada de requisição *preflight*, e é a implementação concreta do mecanismo *opt-in* do CORS em funcionamento, como mencionado acima. Entre as requisições que um `<form>` pode fazer, que são chamadas de "requisições simples" e serão discutidas mais a frente, a especificação do CORS requer que o navegador verifique com os servidores antes de fazer uma requisição *cross-origin*.

Como é a resposta de uma requisição *preflight*? Se o nosso *endpoint* não está familiarizado com o CORS, pode ser que ele retorne um *status code* como `404` ou `501`, o que faz com que o navegador imediatamente rejeite a requisição.

Se o servidor não suporta CORS, mas não permite requisições de outros domínios, nós veremos algo como:

<pre><code>OPTIONS https://api.users.com/me HTTP/1.1
Status: 200
Access-Control-Allow-Origin: https://notyourdomain.com
Access-Control-Allow-Method: POST</code></pre>

Essa resposta diz ao navegador que esse *endpoint* só pode ser acessado por `https://notyourdomain.com`, e que ele não quer deixar nenhum outro domínio interagir com ele. O navegador irá obedecer e terminar a sua requisição.

Se o servidor suporta CORS e não se importa com quem interage com o *endpoint* em questão, nós veremos isso:

<pre><code>OPTIONS https://api.users.com HTTP/1.1
Status: 200
Access-Control-Allow-Origin: *
Access-Control-Allow-Method: POST</code></pre>

O asterisco (`*`) significa que o *endpoint* permite que qualquer domínio acesse-o, e que o navegador deve permitir a requisição, ou seja, permite a requisição de atualizar do usuário a prosseguir.

Existem algumas nuances nas requisições *preflight*, e para melhor entendê-las, vamos tirar um tempo para entender o que são requisições simples e porque elas não estão sujeitas ao *preflight*.

<figure>
  <img src="/public/svg/whale-outline.svg" width="100%" style="max-width: 50rem"/>
</figure>

### Entendendo Requisições Simples (Simple Requests)

Se tem uma coisa sobre CORS que eu queria ter descoberta mais cedo, é como ele lida com requisições simples. Pense em requisições como qualquer requisição que um `<form>` pode fazer. Porque essa distinção é importante? Bem, antes do CORS, os únicos pedidos que uma página poderia enviar eram originários de `<form>`s. Assim, como essas requisições eram permitidas antes do CORS, a especificação não exige que o navegador execute requisições *preflight* para elas.

Definindo tecnicamente, no entanto, requisições simples são a combinação de um [_método simples_](https://www.w3.org/TR/cors/#simple-method){:target="_blank"} com um [_cabeçalho simples_](https://www.w3.org/TR/cors/#simple-header){:target="_blank"}.

Os __métodos simples__ são `GET`, `HEAD`, e `POST`. Fáceis de lembrar.

Os __cabeçalhos simples__ são `Accept`, `Accept-Language`, `Content-Language`, ou (e isso é importante) `Content-Type` __se__ `Content-Type` possuir qualquer um desses três valores: `application/x-www-form-urlencoded`, `multipart/form-data`, or `text/plain`.

Por que o uso desses três valores mágicos do `Content-Type` torna o cabeçalho um cabeçalho simples? A resposta tem a ver com elementos `<form>` e os três tipos de codificação de conteúdo (tipos MIME) que eles podem enviar. Confira [este artigo](https://developer.mozilla.org/en-US/docs/Web/API/HTMLFormElement/enctype){:target="_blank"} no MDN para saber mais. Os redatores do CORS acharam que não era necessário bloquear esses pedidos, uma vez que os formulários já existiam há vários anos, e os servidores provavelmente perceberiam que tais requisições no *client-side* eram possíveis.

Para ajudar a fortalecer essa distinção, aqui estão algumas requisições simples, conforme descrito acima:

<pre><code class="http">GET https://api.users.com/user/1 HTTP/1.1</code></pre>

<pre><code>POST https://api.users.com/user/1 HTTP/1.1
Content-Type: application/x-www-form-urlencoded

name=Demo%20User&description=I%27m%20a%20demo%20user%21
</code></pre>

E aqui estão as mesmas requisições, mas com alguns ajustes para que elas passem pelo *preflight*:

<pre><code class="http">GET https://api.users.com/user/1 HTTP/1.1
X-Random-Header: 42</code></pre>

<pre><code>POST https://api.users.com/user/1 HTTP/1.1
Content-Type: application/json

{
  "name": "Burt Macklin",
  "description": "Parece um trabalho para Burt Macklin!"
}</code></pre>

Em ambos os casos, embora estamos usando métodos simples, a adição de cabeçalhos que não se enquadram na definição de "cabeçalhos simples" resulta na emissão de uma requisição *preflight*. Essas requisições só podem ser enviadas se a resposta do *preflight* contiver um cabeçalho "Access-Control-Allow-Headers" que cita o cabeçalho não simples como permitido, por exemplo:

<pre><code>OPTIONS https://api.users.com HTTP/1.1
Status: 200
Access-Control-Allow-Headers: Content-Type
Access-Control-Allow-Origin: *
Access-Control-Allow-Method: POST</code></pre>

Entendendo requisições simples vai, eu espero, trazer uma luz no porquê algumas requisições parecem passar ilesos pelas restrições do CORS enquanto outras são bloqueadas. A adição de um simples cabeçalho, ou o uso de um outro método, é o suficiente para o CORS chegar e abandonar sua requisição.

Nota final: só porque uma requisição é simples não significa que ela escapou completamente do CORS. Só significa que o navegador a iniciou de uma vez, sem fazer um *preflight*. Se a resposta para uma requisição simples contém um `Access-Control-Allow-Origin` que não inclui o domínio que fez a requisição, ou fornece false para `Access-Control-Allow-Credentials` quando as credencias foram de fato usadas, a resposta pode ser sufocada, ainda que tenha sido completada. O resultado da resposta é descartado e nunca fica visível para o JavaScript que a solicitou.

<figure>
  <img src="/public/svg/tail-outline.svg" width="100%" style="max-width: 30rem"/>
</figure>

### CORS Portátil

Com nossos pontos sobre *preflight* e requisições simples completos, é útil saber onde mais você pode encontrar o CORS além do `XMLHttpRequest` ou `fetch` de APIs. Existem duas especificações adicionais que solicitam certas requisições a implementar procedimentos de CORS:

- CSS3 requer que todas requisições de fontes [garantam a necessidade de CORS](https://www.w3.org/TR/css-fonts-3/#same-origin-restriction){:target="_blank"}.
- WebGL requer que todas requisições de textures [garantam a necessidade de CORS](https://www.khronos.org/registry/webgl/specs/latest/1.0/#4.2){:target="_blank"}

<figure>
  <img src="/public/svg/lighthouse-outline.svg" width="100%" style="max-height: 30rem"/>
</figure>

### Conclusão

Nós cobrimos grande parte do território de forma rápida, e espero que este post tenha te dado uma melhor perspectiva das motivações por trás do CORS. Existem vários pontos que não consegui abranger aqui, como: como você pode *cachear* respostas de *preflight* com o cabeçalho `Access-Control-Max-Age`. Enquanto isso, eu incluí uma lista de links que foram úteis para mim enquanto montava este post, e se você encontrou algum erro, seja de informacional ou sintático, deixe nos comentários!

### Links e Referências

[MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS){:target="_blank"}

[Espedificação do CORS da W3C](https://www.w3.org/TR/cors/){:target="_blank"}

[Especificação de Fetch da W3C - Seção sobre CORS](https://fetch.spec.whatwg.org/#http-cors-protocol){:target="_blank"}

[Thread Maravilhosa do Stack Overflow](https://stackoverflow.com/questions/15381105/cors-what-is-the-motivation-behind-introducing-preflight-requests){:target="_blank"}
