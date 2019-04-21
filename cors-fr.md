---
author_name: Ryan Miller
author_twitter: andryanmiller
excerpt: CORS (Cross-Origin Resource Sharing) est un sujet qui inspire la crainte chez beaucoup de développeurs web. Comme les légendes des créatures marines mythiques, chaque développeur a une histoire à raconter sur le jour où CORS s’est emparé d’une de leurs requêtes web, l’emportant avec lui dans les abysses. Plus jamais on n’entendit parler d’elle.
hero: /public/svg/sailor-outline.svg
language: fr
language_alternates: ['en', 'pt', 'ru']
og_image: /public/img/cors_og_image.png
published: true
slug: cors
title: CORS
translator_name: Yanis Vieilly
translator_website: https://medium.com/@yanisvieilly
---
CORS (Cross-Origin Resource Sharing) est un sujet qui inspire la crainte chez beaucoup de développeurs web. Comme les légendes des créatures marines mythiques, chaque développeur a une histoire à raconter sur le jour où CORS s’est emparé d’une de leurs requêtes web, l’emportant avec lui dans les abysses. Plus jamais on n’entendit parler d’elle.

<figure>
  <img src="/public/svg/kraken-outline.svg" width="100%" style="max-width: 50rem"/>
  <figcaption>"No 'Access-Control-Allow-Origin' header is present on the requested resource. Origin 'https://example.com' is therefore not allowed access."</figcaption>
</figure>

Qu’il s’agisse de récupérer du JSON, ou de tenter de configurer un CDN pour des contenus média, CORS semble être toujours là au mauvais moment. C’est pourquoi les développeurs ont appris à apaiser CORS, lui valant une réputation d’obstacle qui, pour une raison quelconque, améliore la sécurité de nos utilisateurs.

Cet article cherche à démystifier CORS et montrer son bon côté : une spécification qui n’a pas pour but de freiner le progrès des développeurs web, mais plutôt de nous libérer de l’emprise de la _same-origin policy_. Nous allons parcourir chaque en-tête nécessaire au bon fonctionnement de CORS, puis parler de quelques endroits où CORS a son utilité et qui pourraient vous surprendre.

### La brève histoire de CORS

CORS, ou l’idée qui allait devenir CORS, est né dans [l’ère du Web 2.0](https://www.oreilly.com/pub/a/web2/archive/what-is-web-20.html?){:target="_blank"}, vers 2005. Un des premiers *buzzwords* du Web 2.0 fut [AJAX](http://adaptivepath.org/ideas/ajax-new-approach-web-applications/){:target="_blank"}, ou "Asynchronous JavaScript and XML" ("JavaScript et XML asynchrones", en français), dont l’idée est que l’on peut utiliser l’API [XMLHttpRequest](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest){:target="_blank"} pour mettre à jour une page de manière asynchrone, sans la rafraîchir complètement.

Mais quand XMLHttpRequest est arrivé, celui-ci avait une restriction de taille : il était seulement possible pour son API de communiquer avec les services présents sur le même domaine que le site qui effectuait la requête. En pratique, cela signifie que si un site était disponible sur `https://iloveajax.com`, et que vous vouliez faire une requête vers `https://externalresource.com` (ou même `https://subdomain.iloveajax.com`), le navigateur refuserait de démarrer cette requête. C’est le principe de la [_same-origin policy_](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy){:target="_blank"}.

AJAX prenant de l’ampleur, il devint clair qu’il fallait faire quelque chose à propos d’XMLHttpRequest et sa camisole _same-origin_. La communauté des développeurs web vit comment ouvrir AJAX à d’autres domaines pourrait mener à de nouveaux services et de nouvelles façons d’utiliser le web, et (attention, spoiler) c’est exactement ce qui se produisit quand on pense à Firebase, Mixpanel, New Relic, et autres. C’est vers cette époque (2005) que les gens commencèrent à ruser en utilisant quelque chose appelé [JSONP](https://en.wikipedia.org/wiki/JSONP){:target="_blank"}, qui détourne la balise `<script>` (et sa politique plutôt laxiste en matière de sécurité) pour effectuer des requêtes vers des services externes.

En 2005 [le premier brouillon](https://www.w3.org/TR/2005/NOTE-access-control-20050613/){:target="_blank"} de ce qui deviendrait la spécification CORS fut publié. Mais il faudra attendre 2007 pour que les concepts majeurs de la spécification commencent à prendre forme, tels que le mécanisme du *preflight* et l’utilisation des en-têtes HTTP (à la place des balises XML), et sept ans de plus pour qu’elle devienne [une recommandation du W3C](https://www.w3.org/TR/2014/REC-cors-20140116/){:target="_blank"}. Entre temps, cependant, les navigateurs avaient déjà commencé à implémenter les parties stables de la spécification.

Écrire des spécifications n’est pas chose aisée, mais vous êtes en droit de vous demander pourquoi celle-ci a pris une décennie à être rédigée. Quand vous pensez aux implications relatives à la sécurité autour de CORS, cependant, c’est déjà plus compréhensible. Le principal souci étant que la plupart des services web (si ce n’est tous) s’attendaient à ce que les requêtes non-GET qu’ils recevaient viennent de domaines spécifiques (généralement administrés par les mêmes personnes qui possédaient le service en question, étant donné que la _same-origin policy_ était en vigueur). Si CORS venait à être implémenté, cependant, et que la _same-origin policy_ était désactivée, ces mêmes services allaient pouvoir recevoir un déluge de requêtes DELETE, PUT, etc… depuis _n’importe quelle origine_, et il n’était pas envisageable de s’attendre à ce que chaque service web ouvert au public s’adapte à CORS avant sa recommandation par le W3C.

La décision fut donc prise de rendre CORS _opt-in_, ce qui signifie que les navigateurs continueraient à appliquer la _same-origin policy_, sauf si un service web déclarait grâce à une série de signaux qu’il autorisait le contenu vers d’autres origines. Nous allons parler des spécificités de ce mécanisme appelé _preflighting_ dans un instant. En intégrant ce côté _opt-in_ au design de CORS, cela signifiait que les services web n’auraient pas à répondre à une multitude de requêtes inattendues, et que les développeurs web allaient pouvoir commencer à construire de nouveaux types de services et d’outils.

Peut-être que vous n’apprécierez jamais vraiment CORS. Mais s’il y a une chose pour laquelle on devrait tous être un minimum reconnaissants, c’est le fait que CORS a trouvé le juste équilibre entre rétrocompatibilité et l’ouverture d’un large éventail de nouvelles fonctionnalités aux développeurs web. Ce n’est pas chose facile ! Et pour mieux démontrer cet exploit, voyons ensemble comment CORS peut affecter vos requêtes web et comment vous pouvez éviter certains de ses subterfuges.

<figure>
  <img src="/public/svg/seagull-outline.svg" width="100%" style="max-width: 30rem"/>
</figure>

### Preflighting

Un des aspects les plus déroutants de CORS est son utilisation des requêtes _preflight_. Imaginez que vous lanciez la requête _cross-domain_ POST suivante pour mettre à jour un profil utilisateur :

<pre><code>POST https://api.users.com/me HTTP/1.1
Host: example.com
Content-Type: application/json; charset=utf-8

{
  "name": "Demo User",
  "description": "I'm a demo user!"
}</code></pre>

Si vous lanciez cette requête dans un navigateur qui implémente CORS, vous verriez celui-ci envoyer la requête suivante en premier :

<pre><code>OPTIONS https://api.users.com/me HTTP/1.1
Host: example.com
Access-Control-Request-Headers: content-type
Access-Control-Request-Method: POST
Origin: https://example.com</code></pre>

Attendez un peu. Qu’est-ce qu’il vient de se passer ? La première requête `OPTIONS` est appelée requête _preflight_, et c’est l’implémentation concrète du côté _opt-in_ de CORS dont nous parlions précédemment. Au-delà des requêtes qu’un élément `<form>` pourrait produire, que l’on appelle “requêtes simples” et que l’on abordera dans la prochaine partie, la spécification CORS demande aux navigateurs de consulter les serveurs avant de faire des requêtes _cross-origin_.

À quoi ressemble une requête _preflight_ ? Si notre endpoint ne connaît pas CORS, il pourrait retourner un code de statut tel que `404` ou `501`, ce qui indiquerait au navigateur qu’il doit annuler la requête.

Si le serveur supporte CORS, mais n’autorise pas de requêtes depuis notre domaine, on pourrait voir quelque chose comme ça :

<pre><code>OPTIONS https://api.users.com/me HTTP/1.1
Status: 200
Access-Control-Allow-Origin: https://notyourdomain.com
Access-Control-Allow-Method: POST</code></pre> 

Cette réponse dit au navigateur que cet _endpoint_ peut seulement être accédé par `https://notyourdomain.com`, et qu’aucun autre domaine n’est autorisé à interagir avec lui. Le navigateur obéirait alors et mettrait fin à notre requête.

Si le serveur supporte CORS et n'impose pas de restrictions quant aux domaines pouvant interagir avec l’_endpoint_ en question, nous verrions sûrement :

<pre><code>OPTIONS https://api.users.com HTTP/1.1
Status: 200
Access-Control-Allow-Origin: *
Access-Control-Allow-Method: POST</code></pre> 

L’astérisque (`*`) signifie que l’_endpoint_ veut autoriser n’importe quel domaine à interagir avec lui, et que le navigateur doit autoriser la requête (dans cet exemple, la requête pour mettre à jour le profil utilisateur) à se poursuivre.

Il y a quelques autres subtilités concernant les requêtes _preflight_, et pour mieux les comprendre, prenons un moment pour comprendre les requêtes simples et pourquoi celles-ci ne sont pas concernées par le _preflighting_.

<figure>
  <img src="/public/svg/whale-outline.svg" width="100%" style="max-width: 50rem"/>
</figure>

### Comprendre les requêtes simples

S’il y a une chose que j’aurais aimé savoir plus tôt à propos de CORS, c’est comment celui-ci gère les _requêtes simples_. Pensez aux requêtes simples comme à n’importe quelle requête qu’un élément `<form>` pourrait initier. Pourquoi doit-on faire cette distinction ? Car avant CORS, les seules requêtes qu’une page web pouvait envoyer provenaient des éléments `<form>`. Ainsi, comme ces mêmes requêtes étaient déjà permises avant l’arrivée de CORS, la spécification ne requiert pas que le navigateur effectue des requêtes _preflight_ pour elles.

Pour rentrer dans les détails techniques, les requêtes simples sont une combinaison d’une [_méthode simple_](https://www.w3.org/TR/cors/#simple-method){:target="_blank"} avec un [_en-tête simple_](https://www.w3.org/TR/cors/#simple-header){:target="_blank"}.

Les __méthodes simples__ sont `GET`, `HEAD`, et `POST`. Il est assez facile de s’en souvenir.

Les __en-têtes simples__ sont `Accept`, `Accept-Language`, `Content-Language`, ou (et c’est important) `Content-Type` __si__ `Content-Type` prend une de ces trois valeurs : `application/x-www-form-urlencoded`, `multipart/form-data`, ou `text/plain`.

Pourquoi l’utilisation de ces trois valeurs magiques de `Content-Type` font de cet en-tête un en-tête simple ? La réponse est liée aux éléments `<form>` et aux trois types d’encodage de contenu (types MIME) qu’ils sont autorisés à envoyer. Consultez [cet article](https://developer.mozilla.org/en-US/docs/Web/API/HTMLFormElement/enctype) sur MDN pour en savoir plus. Les auteurs de CORS ont pensé qu’il n’était pas nécessaire de prévenir ces requêtes, étant donné que les formulaires existaient déjà depuis plusieurs années, et que les serveurs étaient sans doute déjà au courant que de telles requêtes étaient possibles côté client.

Pour aider à bien comprendre cette distinction, voici quelques requêtes simples telles que décrites précédemment :

<pre><code class="http">GET https://api.users.com/user/1 HTTP/1.1</code></pre>

<pre><code>POST https://api.users.com/user/1 HTTP/1.1
Content-Type: application/x-www-form-urlencoded

name=Demo%20User&description=I%27m%20a%20demo%20user%21
</code></pre>

Et voilà les mêmes requêtes, mais légèrement modifiées pour qu’elles doivent être précédées d'un _preflight_ :

<pre><code class="http">GET https://api.users.com/user/1 HTTP/1.1
X-Random-Header: 42</code></pre>

<pre><code>POST https://api.users.com/user/1 HTTP/1.1
Content-Type: application/json

{
  "name": "Demo User",
  "description": "I'm a demo user!"
}</code></pre>

Dans les deux cas, bien que nous utilisions des méthodes simples, l’ajout d’en-têtes qui ne font pas partie des “en-têtes simples” implique le lancement d’une requête de _preflight_. Ces requêtes peuvent seulement être envoyées si la réponse au _preflight_ contient un en-tête `Access-Control-Allow-Headers` qui indique que l’en-tête “non-simple” est autorisé, par exemple :

<pre><code>OPTIONS https://api.users.com HTTP/1.1
Status: 200
Access-Control-Allow-Headers: Content-Type
Access-Control-Allow-Origin: *
Access-Control-Allow-Method: POST</code></pre> 

Comprendre les requêtes simples devrait permettre d’expliquer le fait que certaines requêtes semblent passer les restrictions CORS sans problème, alors que d’autres sont bloquées. L’ajout d’un en-tête, ou l’utilisation d’une méthode alternative, sont autant d’éléments qui peuvent déclencher le blocage de votre requête par CORS.

Une dernière note : juste parce qu’une requête est une requête simple ne signifie pas qu’elle échappera complètement à CORS. Cela signifie simplement que le navigateur peut initier la requête immédiatement sans avoir besoin d’une requête _preflight_. Si la réponse à cette requête simple contient un `Access-Control-Allow-Origin` qui n’inclut pas le domaine qui a effectué la requête, ou un `Access-Control-Allow-Credentials` de valeur `false` alors que des identifiants ont bien été utilisés, celle-ci peut être annulée, alors même qu’elle avait été complétée. Le résultat de la réponse est rejeté et n’est jamais rendu visible pour le code JavaScript l’ayant requis.

<figure>
  <img src="/public/svg/tail-outline.svg" width="100%" style="max-width: 30rem"/>
</figure>

### CORS portable

Maintenant que notre brève étude des requêtes simples et de _preflight_ est terminée, il peut être utile de savoir où il est également possible de trouver CORS, en dehors d’`XMLHttpRequest` et des APIs `fetch`. Il y a deux spécifications supplémentaires qui obligent certaines requêtes à implémenter le processus CORS :

- CSS3 requiert que toutes les requêtes pour les polices [appliquent CORS](https://www.w3.org/TR/css-fonts-3/#same-origin-restriction){:target="_blank"}.
- WebGL requiert que toutes les requêtes pour les textures [appliquent CORS](https://www.khronos.org/registry/webgl/specs/latest/1.0/#4.2){:target="_blank"}

<figure>
  <img src="/public/svg/lighthouse-outline.svg" width="100%" style="max-height: 30rem"/>
</figure>

### Conclusion

Nous avons appris beaucoup de choses en peu de temps, et j’espère que ce post vous a permis de comprendre les motivations derrière la spécification CORS. Il reste quelques sujets que nous n’avons pas abordé, tels que la possibilité de mettre en cache les réponses preflight avec l’en-tête `Access-Control-Max-Age`. En attendant, j’ai inclus à cet article une petite liste de liens qui m’ont aidé à le rédiger, et si vous avez trouvé une erreur, que ce soit dans les points abordés ou une faute de grammaire, faites-le moi savoir dans les commentaires en bas de la page !

### References & Links

[MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS){:target="_blank"} 

[Spécification CORS du W3C](https://www.w3.org/TR/cors/){:target="_blank"}

[Spécification Fetch du W3C - Partie CORS](https://fetch.spec.whatwg.org/#http-cors-protocol){:target="_blank"}

[Un sujet merveilleux sur Stack Overflow](https://stackoverflow.com/questions/15381105/cors-what-is-the-motivation-behind-introducing-preflight-requests){:target="_blank"}
