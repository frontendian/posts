---
author_name: Ryan Miller
author_twitter: andryanmiller
excerpt: A good defensive strategy is multilayered. Whether it's the multifactor authentication system you use to log into GitHub, or the kill switch on Furiosa's war rig, having more than one safeguard against intrusion makes attacks substantially more difficult. The same is true for web security, and today's post is going to introduce you to a powerful tool you have to augment your website's security: content security policies, or CSPs.
hero: /public/svg/helmet.svg
language: en
og_image: /public/img/csp_og_image.png
published: true
published_date: 2018-07-20 08:00:00 -0700
slug: csp
title: "Content Security Policies"
---
A good defensive strategy is multilayered. Whether it's the multifactor authentication system you use to log into GitHub, or the [kill switch on Furiosa's war rig](http://madmax.wikia.com/wiki/Tatra_T815_%22The_War_Rig%22#Appearance_on_screen){:target="_blank"}, having more than one safeguard against intrusion makes attacks substantially more difficult. The same is true for web security, and today's post is going to introduce you to a powerful tool you have to augment your website's security: _content security policies_, or CSPs.

## Introduction

You can think of a content security policy as a bouncer standing in front of your webpage. It's the bouncer's role to check resources against a predetermined guest list and that each guest matches certain entrance criteria. Take, for example, a hypothetical content security policy for this very blog:

<pre><code class="http">Content-Security-Policy: default-src: 'self' cdn.frontendian.co;</code></pre>

I'll explain more about the syntax in a bit–this policy tells the browser to only accept resources that were loaded from the same origin as the page itself, namely `frontendian.co`, or the domain `cdn.frontendian.co`. But that's not all–just by specifying this policy, we prevent any inline (or evaluated) CSS or JavaScript from being loaded into the page. We could enable either via additional _keywords_ in our policy:

<pre><code class="http">Content-Security-Policy: default-src: 'self' 'unsafe-inline' 'unsafe-eval' cdn.frontendian.co;</code></pre>

By adding those two keywords, we now have a content security policy that communicates three things:

1. Only allow external resources from my page's origin.
2. Allow inline scripts and stylesheets.
3. Allow the use of JavaScript `eval`.

As we'll discuss in a moment, you can still be much more finegrained when it comes to how you want to load different kinds of resources. CSP _directives_, like `default-src`, exist for fonts, scripts, styles, and more, and allow to enforce stricter criteria for one resource type over another. 

## History

What motivated the creation of content security policies? In short: the rise of sites that allowed user-submitted scripts and styles, such as MySpace. Robert Hansen (perhaps better known as RSnake) has a nice writeup on [the origins of CSPs at Mozilla](https://web.archive.org/web/20150318224529/http://ha.ckers.org/blog/20090701/mozillas-content-security-policy/){:target="_blank"}. 

As was mentioned above, given the risk that malicious user-submitted content can pose it's best to have multiple layers of defense. Then and now, the primary means of disarming user content is via sanitization, but if that fails you're left without much recourse. CSPs ensure that, even if a user does somehow bypass your sanitization mechanisms, the browser will still prevent the malicious resources from executing.

## Crafting a Policy

CSPs are represented as a single line of text, and are communicated to the browser in one of two ways:

- Via a `<meta>` element with its `http-equiv` attribute set to `Content-Security-Policy`, e.g. `<meta http-equiv="Content-Security-Policy" content="...">`.
- Via a `Content-Security-Policy` header on the HTTP resource. For a handful of reasons which will be discussed, this is the preferred means of communicating a CSP to the browser.

The individual components of a CSP are called _directives_, and it is through these directives that you instruct the browser how it should treat various resources. Directives begin with the directive's name, such as `default-src` or `img-src` and are followed by one or more domains or keywords (like `'unsafe-inline'` or `'self'`) which is called the directive's _source list_.

## Fetch Directives

Directives that specify which resources may be loaded into a page are known as "fetch directives". What follows is a brief tour of the available fetch directives and their quirks.

### default-src

The `default-src` directive specifies the default source list for all other fetch directives, though it can be overriden by specifying another fetch directive in your policy.

<pre><code class="http">default-src: 'self' assets.frontendian.co;</code></pre>

Note that `default-src` may also list keywords that apply to a subset of fetch directives, such as `'unsafe-inline'`, given that directives which do not recongize those keywords will simply ignore them.

### child-src

The `child-src` directive is one of the more confusing directives in the CSP spec. Introduced in the second edition, it was intended to replace the `frame-src` directive and govern both "nested browsing contexts" (`<iframe>`s and `<frame>`s) as well as script resources loaded as `Worker`s. However, in the third edition of the spec the `worker-src` directive was introduced, and the `child-src` directive simply became shorthand for specifying both the `frame-src` and `worker-src` directives. Oy vey!

Given this, the `child-src` directive exists in a sort of limbo. It's unclear from the second edition of the CSP spec whether user agents should prefer a `frame-src` directive over the `child-src` directive with regard to nested browsing contexts, and so I recommend doing some thorough testing before rolling out the `child-src` directive.

<pre><code class="http">child-src: 'self' scripts.frontendian.co;</code></pre>

### connect-src

The `connect-src` directives governs which resources may be obtained via APIs like `XMLHttpPRequest` or `fetch`. That's not a complete list, and you're encouraged to check out [the full reference](https://www.w3.org/TR/CSP3/#directive-connect-src){:target="_blank"} on the W3C site.

<pre><code class="http">connect-src: 'self' api.frontendian.co;</code></pre>

### font-src 

The `font-src` directive gates all font resources your page might load, which more than likely originate from `@font-face` declarations. The `font-src` directive accepts no special keywords beyond `'self'` or `'none'`.

<pre><code class="http">font-src: 'self' fonts.frontendian.co;</code></pre>

### frame-src

While [deprecated](https://www.w3.org/TR/CSP2/#directive-frame-src){:target="_blank"} in the second edition of the CSP spec, the `frame-src` directive makes its triumphant return [in the third](https://www.w3.org/TR/CSP3/#directive-frame-src){:target="_blank"}. `frame-src` governs which URLs may be accessed via "nested browsing contexts", such as `<iframe>`s and `<frame>`s. 

It's recommended that you read up on the `child-src` directive if you plan on implementing `frame-src`, as it's one of the sharper corners of the CSP spec.

<pre><code class="http">frame-src: 'self' iframes.frontendian.co;</code></pre>

### img-src

The `img-src` directive gates image resources, and respects no special keywords beyond `'self'` or `'none'`. 

<pre><code class="http">img-src: 'self' img.frontendian.co;</code></pre>

### manifest-src 

The `manifest-src` directive, introduced in the third edition of the CSP spec, gates app manifests, which are one of the building blocks of Progressive Web Apps. 

<pre><code class="http">manifest-src: 'self' manifest.frontendian.co;</code></pre>

### media-src

The `media-src` directive gates audio and video resources, and respects no special keywords beyond `'self'`.

<pre><code class="http">media-src: 'self' media.frontendian.co;</code></pre>

### object-src

The `object-src` directives gates browsing contexts generated by elements such as `<object>` or `<embed>`.

<pre><code class="http">object-src: 'self' embed.frontendian.co;</code></pre>

### script-src

The `script-src` directive is arguably the most complex of all CSP directives, and for good reason. Malicious pose tremendous risk to webpages, and sites with a large number of script resources need sophisticated means of ensuring that every `<script>` is whitelisted.

To start with, the `script-src` directive accepts the standard `'self'` and domain elements for its source list:

<pre><code class="http">script-src: 'self' scripts.frontendian.co;</code></pre>

In addition to these, we also have the `'unsafe-inline'` and `'unsafe-eval'` keywords at our disposal. The former allows the usage of `<script>` tags that execute JavaScript inline, while the latter prevents the usage of `eval` to evaluate strings into executable JavaScript. __If at all possible you should avoid using these keywords__.

Yet as many of us who maintain legacy websites know, such a thing just might not be possible. So the `script-src` directives gives us a couple neat ways to ensure we only allow whitelisted inline JavaScript.

The first method, called `nonce-source`, allows you to specify a "nonce", or random key, that can be used to specify approved inline scripts. For example, consider the following:

<pre><code class="http">script-src: 'nonce-1a2b3c';</code></pre>
<pre><code class="html">&lt;script nonce="1a2b3c"&gt;function () { alert('Hello!'); }&lt;/script&gt;</code></pre>

Since our `script-src` directive specifies the `'nonce-1a2b3c'` `nonce-src`, we are able to execute any `<script>` that possesses the `nonce` attribute set to `1a2b3c`.

The second method, called `hash-source`, allows you to take a hash of the script you wish to execute and specify that in your `script-src` directive. 

<pre><code class="http">script-src: 'sha256-qznLcsROx4GACP2dm0UCKCzCG+HiZ1guq6ZZDob/Tng=';</code></pre>
<pre><code class="html">&lt;script&gt;alert('Hello, world.');&lt;/script&gt;</code></pre>

Given that the hash of the script (taken as `sha256`) matches the hash appended to `sha256-` in the `script-src` directive's source list, the script will be allowed to run. 

### style-src

The `style-src` directive gates CSS resources, and borrows a couple of the special keywords from the `script-src` directive (`'unsafe-inline'` and `'unsafe-eval'`).

The `'unsafe-inline'` keyword works as you might expect, and disallows the usage of `<style>` elements on the page, as well as `style` attributes on DOM nodes. The `'unsafe-eval'` keyword, however, is a bit less intuitive. It disallows interactions with the CSS Object Model, or CSSOM, which equates to being able to mutate a page's styles with JavaScript.

<pre><code class="http">style-src: 'self' styles.frontendian.co;</code></pre>

As an added perk, both the `nonce` and `hash` sources are available to `style-src`, and they are meant to function to apply to `<style>` elements in the same way that they would to `<script>` elements.

### worker-src

The `worker-src` directive was introduced in the third edition of the CSP spec, and governs which script resources may be used to create web workers. In the second edition of the CSP spec this responsibility was borne by the `child-src` directive, which also governed nested browsing contexts, but the decision was made to split the workload across two directives.

<pre><code class="http">worker-src: 'self' scripts.frontendian.co;</code></pre>

## Document Directives

Document directives provide more meta-level guidance about how the document should behave.

### base-uri

The `base-uri` directive gates which URLs may become the base URL of a page. This helps protect your page against [base tag hijacking](https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/base-tag-hijacking/){:target="_blank"}.

<pre><code class="http">base-uri: 'self';</code></pre>

### plugin-types

The `plugin-types` directives specifies the sorts of content that may be loaded into `<object>` or `<embed>` tags. You do this by passing a list of mimetypes as source list, like so:

<pre><code class="http">plugin-types: application/pdf;</code></pre>

## Navigation Directives

Navigation directives govern where your page may go after a user action (restricted to `<form>`s at the moment), or in which sorts of nested browsing contexts it may live. 

### form-action

The `form-action` directive specifies which URLs a form may send resources or user information. Be aware that, since this directive is considered a _navigation directive_, the `default-src` directive will not supply it with a default value. Make sure to set this directive independently if your site makes use of `<form>`s.

<pre><code class="http">form-action: 'self' api.frontendian.co;</code></pre>

### frame-ancestors

The `frame-ancestors` directive governs which resources may load the page into an `<iframe>` or other nested browsing context.

It is intended to obsolete the `X-Frame-Options` header.

<pre><code class="http">frame-ancestors: 'none';</code></pre>

## Monitoring Your Policy's Effectiveness

One of the biggest advantages of CSPs is that you can monitor attempts to breach your policy in real-time. This is done by specifying a `report-uri` (and/or `report-to`, per [the third edition](https://www.w3.org/TR/CSP3/#directive-report-uri){:target="_blank"}) directive in your policy.

<pre><code class="http">report-to: https://csp.frontendian.co;</code></pre>

Should a violation of your CSP occur in a user's session, the browser will POST to that URL with an object of the following structure:

<pre><code class="json">{
  "document-uri": "https://frontendian.co",
  "referrer": "",
  "blocked-uri": "http://haxors.com/malicious/script.js",
  "violated-directive": "script-src: 'self'",
  "original-policy": "script-src: 'self'; report-to: https://csp.frontendian.co;"
}</code></pre>

Be aware that reporting can easily swamp your site if you have a number of visitors and haven't yet properly vetted all of the possible resources that may violate your policy. Make sure to set up your reporting endpoint such that it won't impact your site's resources. Serverless offerings like [Zeit](https://zeit.co/){:target="_blank"} or Lambda would be great for this.

## Testing Your Policy

As you many have already imagined, a content security policy deployed prematurely can debilitate your site. Particularly if you use third-party scripts, you may be surprised (or perhaps horrified) to learn that those third-party scripts are themselves loading other scripts, and that your CSP is preventing crucial resources from loading. 

To account for this, the authors of the CSP spec made an affordance for those wishing to test out their CSPs before fully deploying them. This affordance is known as `report-only` mode, and allows you to deploy a content security policy without the browser actually enforcing the policy. Yet the browser will still send reports, as we discussed above, making it an effective way to ensure you've checked all of your boxes before activating your policy.

You can activate `report-only` mode by simply communicating your policy to the browser via the `Content-Security-Policy-Report-Only` header, instead of `Content-Security-Policy`. 

## Conclusion

CSPs are quickly becoming an essential part of every website's security toolkit. Hopefully this post has given you a better understanding of how you can tailor CSPs to your site's needs, and if you've spotted an error, be it factual or syntactical, let me know in the comments below!

## References & Resources

[MDN Guide to CSPs](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP){:target="_blank"}

[CSP Spec (Version 2)](https://www.w3.org/TR/CSP2){:target="_blank"}

[CSP Spec (Version 3)](https://www.w3.org/TR/CSP3){:target="_blank"}

[A Lovely Resource on CSPs (content-security-policy.com)](https://content-security-policy.com/){:target="_blank"}

[Awesome HTML5 Rocks Guide](https://www.html5rocks.com/en/tutorials/security/content-security-policy/){:target="_blank"}

[Twitter Case Study](https://blog.twitter.com/engineering/en_us/a/2011/improving-browser-security-with-csp.html){:target="_blank"}
