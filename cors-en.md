---
author_name: Ryan Miller
author_twitter: andryanmiller
excerpt: CORS (Cross-Origin Resource Sharing) is subject tinged with dread for many web developers. Like tales of a mythical sea beast, every developer has a story to tell about the day CORS seized upon one of their web requests, dragging it down into the inexorable depths, never to be seen again. 
hero: /public/svg/sailor-outline.svg
language: en
language_alternates: ['pt', 'fr', 'ru']
og_image: /public/img/cors_og_image.png
published: true
slug: cors
title: CORS
---
CORS (Cross-Origin Resource Sharing) is subject tinged with dread for many web developers. Like tales of a mythical sea beast, every developer has a story to tell about the day CORS seized upon one of their web requests, dragging it down into the inexorable depths, never to be seen again. 

<figure>
  <img src="/public/svg/kraken-outline.svg" width="100%" style="max-width: 50rem"/>
  <figcaption>"No 'Access-Control-Allow-Origin' header is present on the requested resource. Origin 'https://example.com' is therefore not allowed access."</figcaption>
</figure>

Whether it's fetching a bit of JSON, or attempting to configure a CDN for your media assets, CORS seems to make itself a bother at all the wrong times. And so developers have learned to placate CORS, allowing it to garner a reputation as a nuisance that, somehow, makes our users more secure.

This post aims to demystify CORS and show its lighter side–as a specification that didn't set out to hamper the aspirations of web developers everywhere, but instead to loose us from the grip of the _same-origin policy_. We'll go through each of the headers necessary to properly satisfy CORS constraints, and also discuss a couple places where CORS is now relevant but which may surprise you.

### A Brief History of CORS

CORS, or the idea that was to become CORS, was born in [the Web 2.0 era](https://www.oreilly.com/pub/a/web2/archive/what-is-web-20.html?){:target="_blank"}, circa 2005. One of the premier Web 2.0 buzzwords was [AJAX](http://adaptivepath.org/ideas/ajax-new-approach-web-applications/){:target="_blank"}, or "Asynchronous JavaScript and XML", and it captured the idea that you could use the [XMLHttpRequest](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest){:target="_blank"} API to asynchronously update a webpage without a full refresh. 

Yet when XMLHttpRequest first arrived on the scene it had a significant limitation: you could only use its API to communicate with services which were on the same domain as the requesting site. That meant if your site lived on `https://iloveajax.com`, and you wanted to make a request to `https://externalresource.com` (or even `https://subdomain.iloverajax.com`) the browser would simply refuse to initiate the request. This is called the [same-origin policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy){:target="_blank"}.

As AJAX picked up steam it became apparent that something had to be done about XMLHttpRequest and its same-origin straightjacket. The web development community saw how opening up AJAX to other domains could give rise to new services and ways to use the web, which (spoiler alert) it did in the likes of Firebase, Mixpanel, New Relic, and more. At about this same time (2005) people began to cheat the system using something called [JSONP](https://en.wikipedia.org/wiki/JSONP){:target="_blank"}, which essentially hijacked the `<script>` tag (and its rather lax resource security policy) to query data from remote services. 

In 2005 [the first draft](https://www.w3.org/TR/2005/NOTE-access-control-20050613/){:target="_blank"} of what would become the CORS specification was published. Yet it wouldn't be until 2007 that major aspects of the specification began to take shape, such as the "preflight" mechanism and the usage of HTTP headers versus XML markup, and another seven years after that before it became a [W3C recommendation](https://www.w3.org/TR/2014/REC-cors-20140116/){:target="_blank"}. By that time, however, browsers had already begun implementing the more stable parts of the spec.

Writing specifications is no easy task, but you wouldn't be blamed for asking why this one took a decade. When you consider the security implications surrounding CORS, however, it makes a bit more sense. Of chief concern was the fact that most, if not all, web services expected non-GET requests to originate from specific domains (usually owned by the same folks that owned the service in question, given that the same-origin policy was still the law of the land). If CORS were implemented, however, and the same-origin policy for XMLHttpRequests relaxed, said services could now receive a deluge of DELETE, PUT, etc... requests from _any origin_, and it wasn't reasonable to expect every public-facing web service to adapt to CORS prior to its recommendation by the W3C. 

So the decision was made to make CORS opt-in, meaning that browsers would continue to enforce the same-origin policy unless given a specific series of signals by a web service that it was permissible to serve content to different origins. We'll discuss the specifics of this mechanism, called _preflighting_, in just a bit. By building this opt-in feature into the design of CORS it meant that web services wouldn't need to service a torrent of unexpected requests, and web developers could start building new breeds of services and tools.

You may never warm to CORS, but if there's one thing for which we should all have a little gratitude, it's the fact that CORS balances both backwards-compatibility and the opening of a huge swath of new functionality to web developers. Not an easy feat! And to better demonstrate the accomplishment that it is, let's dig into how CORS might affect your web requests and how you can avoid some of its subtler gotchas. 

<figure>
  <img src="/public/svg/seagull-outline.svg" width="100%" style="max-width: 30rem"/>
</figure>

### Preflighting

Probably the most baffling aspect of CORS is its usage of _preflight_ requests. Imagine you initiated the following cross-domain request to POST an update to a user's profile:

<pre><code>POST https://api.users.com/me HTTP/1.1
Host: example.com
Content-Type: application/json; charset=utf-8

{
  "name": "Demo User",
  "description": "I'm a demo user!"
}</code></pre>

If you initiated this request in a browser that implements CORS, you'd see the browser send the following request first:

<pre><code>OPTIONS https://api.users.com/me HTTP/1.1
Host: example.com
Access-Control-Request-Headers: content-type
Access-Control-Request-Method: POST
Origin: https://example.com</code></pre>

Let's pause here for a moment. What's going on? This first `OPTIONS` request is called a preflight request, and it's the concrete implementation of the CORS opt-in mechanism at work, as was just mentioned above. Beyond requests that a `<form>` element might make, which are called "simple requests" and will be discussed in the next section, the CORS specification requires that browsers check with servers before making cross-origin requests.

What does a response to a preflight request look like? If our endpoint isn't familiar with CORS, it might return a status code like `404` or `501`, which would result in the browser immediately stifling the request.

If the server does support CORS, but doesn't allow requests from our domain, we might see something like:

<pre><code>OPTIONS https://api.users.com/me HTTP/1.1
Status: 200
Access-Control-Allow-Origin: https://notyourdomain.com
Access-Control-Allow-Method: POST</code></pre> 

This response tells the browser that this endpoint is only to be accessed by `https://notyourdomain.com`, and that it doesn't want to allow any other domain to interact with it. The browser would obey and terminate your request.

If the server does support CORS and doesn't care who interacts with the endpoint in question, we'd likely see:

<pre><code>OPTIONS https://api.users.com HTTP/1.1
Status: 200
Access-Control-Allow-Origin: *
Access-Control-Allow-Method: POST</code></pre> 

The asterisk (`*`) character means that the endpoint wants to allow any domain to access the endpoint, and that the browser should allow the actual request, namely our request to update the user profile, to proceed.

There are a few additional nuances to preflight requests, and to better understand them, let's take a moment to understand simple requests and why they aren't subject to preflighting.

<figure>
  <img src="/public/svg/whale-outline.svg" width="100%" style="max-width: 50rem"/>
</figure>

### Understanding Simple Requests

If there's one thing I'd wish I'd known about CORS sooner, it's how it handles _simple requests_. Think of simple requests as any request a `<form>` element might be able to initiate. Why is this distinction important? Well, prior to CORS, the only requests a webpage could send originated from `<form>` elements. Thus, since such requests have been permissible pre-CORS, the specification doesn't require that the browser perform preflight requests for them.

Technically defined, however, simple requests are the combination of a [_simple method_](https://www.w3.org/TR/cors/#simple-method){:target="_blank"} with a [_simple header_](https://www.w3.org/TR/cors/#simple-header){:target="_blank"}.

The __simple methods__ are `GET`, `HEAD`, and `POST`. Easy enough to remember.

The __simple headers__ are `Accept`, `Accept-Language`, `Content-Language`, or (and this is important) `Content-Type` __if__ `Content-Type` possesses any of three values: `application/x-www-form-urlencoded`, `multipart/form-data`, or `text/plain`.

Why would the usage of those three magical `Content-Type` values make the header a simple header? The answer has to do with HTML `<form>` elements and the three types of content encodings (MIME types) they are allowed to submit. Check out [this article](https://developer.mozilla.org/en-US/docs/Web/API/HTMLFormElement/enctype) on MDN to learn more. The CORS writers felt it wasn't necessary to gate these requests since forms had already been in existence for several years, and servers would likely be aware such client-side requests were possible.

To help crystallize this distinction, here are a couple simple requests as described above:

<pre><code class="http">GET https://api.users.com/user/1 HTTP/1.1</code></pre>

<pre><code>POST https://api.users.com/user/1 HTTP/1.1
Content-Type: application/x-www-form-urlencoded

name=Demo%20User&description=I%27m%20a%20demo%20user%21
</code></pre>

And here are the same requests, but tweaked slightly so as to result in their being preflighted:

<pre><code class="http">GET https://api.users.com/user/1 HTTP/1.1
X-Random-Header: 42</code></pre>

<pre><code>POST https://api.users.com/user/1 HTTP/1.1
Content-Type: application/json

{
  "name": "Demo User",
  "description": "I'm a demo user!"
}</code></pre>

In both of these cases, though we're using simple methods, the addition of headers that fall outside the definition of "simple headers" result in a preflight request being issued. These requests can only be sent if the preflight response contains an `Access-Control-Allow-Headers` header that cites the non-simple header as allowed, e.g.:

<pre><code>OPTIONS https://api.users.com HTTP/1.1
Status: 200
Access-Control-Allow-Headers: Content-Type
Access-Control-Allow-Origin: *
Access-Control-Allow-Method: POST</code></pre> 

Understanding simple requests will hopefully shed some light on why certain requests seem to pass by CORS strictures unharmed while others are blocked. The addition of a single header, or usage of an alternative method, is enough to cause CORS to engage and scuttle your request.

A final note: just because a request is a simple request does not mean it has wholly escaped CORS. It only means that the browser may initiate the actual request straight away versus performing a preflight. If the response to a simple request contains an `Access-Control-Allow-Origin` that does not include the domain that made the request, or supplies false for `Access-Control-Allow-Credentials` when credentials were in fact used, the response can still be stifled, even though it completed. The result of the response is discarded and is never made visible to the requesting JavaScript.

<figure>
  <img src="/public/svg/tail-outline.svg" width="100%" style="max-width: 30rem"/>
</figure>

### Portable CORS

With our quick surveys of preflight and simple requests complete, it's useful to know where else you might find CORS beyond the `XMLHttpRequest` or `fetch` APIs. There are two additional specifications which oblige certain requests to implement CORS procedeures:

- CSS3 requires all font requests [to ensure CORS is enforced](https://www.w3.org/TR/css-fonts-3/#same-origin-restriction){:target="_blank"}.
- WebGL requires all texture requests [to ensure CORS is enforced](https://www.khronos.org/registry/webgl/specs/latest/1.0/#4.2){:target="_blank"}

<figure>
  <img src="/public/svg/lighthouse-outline.svg" width="100%" style="max-height: 30rem"/>
</figure>

### Conclusion

We covered a great deal of territory quickly, and I hope this post has given you a better perspective on the motivations behind the CORS spec. There are still a few topics I didn't get to here, such as how you can cache preflight responses with the `Access-Control-Max-Age` header. In the meanwhile, I've included a short list of links that were useful to me as I put together this post, and if you've spotted an error, be it factual or syntactical, let me know in the comments below!

<!-- ### CORS Headers: A Bestiary

`Access-Control-Allow-Origin` is a _response_ header, and designates which domain a response is deliverable to (but only one). Note that supplying a wildcard (`*`) value here will allow any domain to receive the response. This header is checked both during "preflighting" and after a successful response is received.

`Origin` is a _request_ header, and is sent by the requesting browser to signifying the domain from which the request originates. The domain is specified as an FQDN, or full-qualified domain name.

`Access-Control-Allow-Headers` is a _response header_, and specifies which headers (beyond the "simple headers", to be discussed shortly) a request is allowed to include. This header is only applicable during "preflighting".

`Access-Control-Allow-Methods` is a _response header_, and specifies the methods the request is allowed to use. This header is applicable during "preflighting".

`Access-Control-Allow-Credentials` is a _response header_, and specifies whether a request is allowed to include a user's credentials (cookies, HTTP authentication, or ["client-side SSL certificates"](https://www.w3.org/TR/cors/#user-credentials)). This header is checked both during "preflighting" and after a successful response is received.

`Access-Control-Max-Age` is a _response header_, and lets the browser know that requests which match the request for which the response was supplied won't need preflights for a certain period of seconds.

`Access-Control-Expose-Headers` is a _response header_, and instructs the browser which headers should be inaccessible to JavaScript on the response as delivered. Can accompany otherwise successful responses. -->

### References & Links

[MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS){:target="_blank"} 

[W3C CORS Specification](https://www.w3.org/TR/cors/){:target="_blank"}

[W3C Fetch Specification - CORS Section](https://fetch.spec.whatwg.org/#http-cors-protocol){:target="_blank"}

[Wonderful Stack Overflow Thread](https://stackoverflow.com/questions/15381105/cors-what-is-the-motivation-behind-introducing-preflight-requests){:target="_blank"}
