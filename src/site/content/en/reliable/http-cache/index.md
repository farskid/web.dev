---
layout: post
title: 'The HTTP Cache: A widely supported technique for avoiding unnecessary network requests'
authors:
  - jeffposnick
  - ilyagrigorik
date: 2018-11-05
description: |
  The browser's HTTP Cache is your first line of defense. It's not necessarily
  the most powerful or flexible approach, and you have limited control over the
  lifetime of cached responses. But there are several rules of thumb that give
  you a sensible caching implementation without much work, so you should always
  try to follow them.
codelabs:
  - codelab-http-cache
---

Fetching resources over the network is both slow and expensive: 

* Large responses require many roundtrips between the browser and the server.
* Your page won't load until all of its [critical resources][crp] have downloaded completely.
* If a person is accessing your site with a limited data plan, every unnecessary 
  network request is a waste of their money.

How can you avoid unnecessary network requests?The browser's HTTP Cache is your
first line of defense. It's not necessarily the most powerful or flexible
approach, and you have limited control over the lifetime of cached responses,
but it's effective, it's supported in all browsers, and it doesn't require much
work.

This guide shows you how to create an effective HTTP caching implementation.
If you're already familiar with the HTTP Cache and just need the recommendations,

## Browser compatibility {: #browser-compatibility }

There isn't actually a single API called the HTTP Cache. It's the general name
for a collection of web platform APIs. These APIs are supported in all browsers:

* [`Cache-Control`](https://developer.mozilla.org/docs/Web/HTTP/Headers/Cache-Control#Browser_compatibility)
* [`ETag`](https://developer.mozilla.org/docs/Web/HTTP/Headers/ETag#Browser_compatibility)
* [`Last-Modified`](https://developer.mozilla.org/docs/Web/HTTP/Headers/Last-Modified#Browser_compatibility)

## How the HTTP Cache works {: #overview }

All HTTP requests that the browser makes are first routed to the browser cache
to check whether there is a valid cached response that can be used to fulfill
the request. If there's a match, the response is read from the cache, which
eliminates both the network latency and the data costs that the transfer incurs.

The HTTP Cache's behavior is controlled by a combination of
[request](https://developer.mozilla.org/en-US/docs/Glossary/Request_header) and
[response](https://developer.mozilla.org/en-US/docs/Glossary/Response_header)
headers. In an ideal scenario, you'll have control over both the code for your
web application (which will determine the request headers) and your web server's
configuration (which will determine the response headers).

## Request headers: stick with the defaults (usually) {: #request-headers }

While there are a number of important headers that should be included in your
web app's outgoing requests, the browser almost always takes care of setting
them on your behalf when it makes requests. Request headers that affect checking
for freshness, like [`If-None-Match`][If-None-Match] and
[`If-Modified-Since`][If-Modified-Since] just appear based on the browser's
understanding of the current values in the HTTP Cache.

This is good news—it means that you can continue including tags like `<img
src="my-image.png">` in your HTML, and the browser  automatically takes care of
HTTP caching for you, without extra effort.

{% Aside %}
Developers who do need more control over the HTTP Cache in their web application
have an alternative—you can "drop down" a level, and manually use the [Fetch
API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API), passing it
[`Request`](https://developer.mozilla.org/en-US/docs/Web/API/Request) objects
with specific
[`cache`](https://developer.mozilla.org/en-US/docs/Web/API/Request/cache)
overrides set. That's beyond the scope of this guide, though!
{% endAside %}

## Response headers: configure your web server {: #response-headers }

The part of the HTTP caching setup that matters the most is the headers that
your web server adds to each outgoing response. The following headers all factor
into effective caching behavior:

* [`Cache-Control`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control).
  The server can return a `Cache-Control` directive to specify how, and for how
  long, the browser and other intermediate caches should cache the individual
  response.
* [`ETag`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/ETag). When
  the browser finds an expired cached response, it can send a small token 
  (usually a hash of the file's contents) to the server to check if the file has
  changed. If the server returns the same token, then file is the same, and there's
  no need to re-download it.
* [`Last-Modified`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Last-Modified).
  This header serves the same purpose as `ETag`, but is less accurate.

Some web servers have built-in support for setting those headers by default,
while others leave the headers out entirely unless you explicitly configure
them. The specific details of _how_ to configure headers varies greatly
depending on which web server you use, and you should consult your server's
documentation to get the most accurate details.

To save you some searching, here are instructions on configuring a few popular
web servers:

+  [Express](https://expressjs.com/en/api.html#express.static)
+  [Apache](https://httpd.apache.org/docs/2.4/caching.html)
+  [nginx](http://nginx.org/en/docs/http/ngx_http_headers_module.html)
+  [Firebase Hosting](https://firebase.google.com/docs/hosting/full-config)
+  [Netlify](https://www.netlify.com/blog/2017/02/23/better-living-through-caching/)

Leaving out the `Cache-Control` response header does not disable HTTP caching!
Instead, browsers [effectively
guess](https://www.mnot.net/blog/2017/03/16/browser-caching#heuristic-freshness)
what type of caching behavior makes the most sense for a given type of content.
Chances are you want more control than that offers, so take the time to
configure your response headers.

## Which response header values should you use? {: #response-header-strategies }

There are two important scenarios that you should cover when configuring your
web server's response headers.

### Long-lived caching for versioned URLs {: #versioned-urls }

{% Details %}
  {% DetailsSummary 'h4' %}
    How versioned URLs can help your caching strategy
    Versioned URLs are a good practice because they make it easier to invalidate
    cached responses. 
  {% endDetailsSummary %}
  Suppose your server instructs browsers to cache a CSS file
  for 1 year (<code>Cache-Control: max-age=31536000</code>) but your designer just made an
  emergency update that you need to roll out immediately. How do you notify browsers
  to update the "stale" cached copy of the file?
  You can't, at least not without changing the URL of the resource. After the
  browser caches the response, the cached version is used until it's no longer
  fresh, as determined by <code>max-age</code> or <code>expires</code>, or until it is evicted from the cache
  for some other reason— for example, the user clearing their browser cache. As a
  result, different users might end up using different versions of the file when
  the page is constructed: users who just fetched the resource use the new
  version, while users who cached an earlier (but still valid) copy use an older
  version of its response. How do you get the best of both worlds: client-side
  caching and quick updates? You change the URL of the resource and force the user
  to download the new response whenever its content changes. Typically, you do
  this by embedding a fingerprint of the file, or a version number, in its
  filename—for example, <code>style.x234dff.css</code>.
{% endDetails %}

When responding to requests for URLs that contain
"[fingerprint](https://en.wikipedia.org/wiki/Fingerprint_(computing))" or
versioning information, and whose contents are never meant to change, add
`Cache-Control: max-age=31536000` to your responses.

Setting this value tells the browser that when it needs to load the same URL
anytime over the next one year (31,536,000 seconds; the maximum supported
value), it can immediately use the value in the local HTTP Cache, without having
to make a network request to your web server at all. That's great—you've
immediately gained the reliability and speed that comes from avoiding the
network!

Build tools like webpack can
[automate the process](https://webpack.js.org/guides/caching/#output-filenames)
of assigning hash fingerprints to your asset URLs.

{% Aside %}
You can also add the [`immutable`
property](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control#Revalidation_and_reloading)
to your `Cache-Control` header as a further optimization, though it [will be
ignored](https://www.keycdn.com/blog/cache-control-immutable#browser-support) in
some browsers.
{% endAside %}


### Server revalidation for unversioned URLs {: #unversioned-urls }

Unfortunately, not all of the URLs you load are versioned. Maybe you're not able
to include a build step prior to deploying your web app, so you can't add hashes
to your asset URLs. And every web application needs HTML files—those files are
(almost!) never going to include versioning information, since no one will
bother using your web app if they need to remember that the URL to visit is
`https://example.com/index.34def12.html`. So what can you do for those URLs?

This is one scenario in which you need to admit defeat. HTTP caching alone isn't
powerful enough to avoid the network completely. (Don't worry—you'll soon learn
about service workers, which will provide the support we need to swing the
battle back in your favor.) But there are a few steps you can take to make sure
that network requests are as quick and efficient as possible.

The following `Cache-Control` values can help you fine-tune whether and how unversioned URLs
are cached:

* `no-cache`. This instructs the browser that it must revalidate with the
  server before using a cached version of the URL.
* `no-store`. This instructs the browser and other intermediate caches (like CDNs) to never
  store any version of the file. `no-store` is often used for files that have sensitive or
  personal data.
* `private`. Browsers can cache the file but intermediate caches cannot.
  For example, `private` would make sense for server-side rendered HTML file
  that contains private user information.

Along with that, setting one of two additional response headers is recommended:

* [`ETag`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/ETag). When
  the browser finds an expired cached response, it can send a small token 
  (usually a hash of the file's contents) to the server to check if the file has
  changed. If the server returns the same token, then file is the same, and there's
  no need to re-download it.
* [`Last-Modified`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Last-Modified).
  This header serves the same purpose as `ETag`, but is less accurate.


either
[`Last-Modified`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Last-Modified)
or [`ETag`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/ETag).

ETags are identifiers for specified resources. They allow caches to be more
efficient and are useful to help prevent simultaneous updates from overwriting
each other.  By setting one or the other of those headers, you'll end up making
the revalidation request much more efficient. They end up triggering the
[`If-Modified-Since`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/If-Modified-Since)
or
[`If-None-Match`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/If-None-Match)
request headers that we mentioned earlier.

{% Details %}
  {% DetailsSummary 'h4' %}
    `ETag` example
  {% endDetailsSummary %}
  Assume that 120 seconds have passed since the initial fetch and the browser
  has initiated a new request for the same resource. First, the browser checks
  the local cache and finds the previous response. Unfortunately, the browser
  can't use the previous response because the response has now expired. At this
  point, the browser could dispatch a new request and fetch the new full
  response. However, that’s inefficient because if the resource hasn't changed,
  then there's no reason to download the same information that's already in the
  cache! That’s the problem that validation tokens, as specified in the <code>ETag</code>
  header, are designed to solve. The server generates and returns an arbitrary
  token, which is typically a hash or some other fingerprint of the contents of
  the file. The browser doesn't need to know how the fingerprint is generated; it
  only needs to send it to the server on the next request. If the fingerprint is
  still the same, then the resource hasn't changed and the browser can skip the
  download. 
{% endDetails %}

When a properly configured web server sees those incoming request headers, it
can confirm whether the version of the resource that the browser already has in
its HTTP Cache matches the latest version on the web server. If there's a match,
then the server can respond with an [`304 Not Modified`][304] HTTP response,
which is the equivalent of "Hey, keep using what you've already got!" There's
very little data to transfer when sending this type of response, so it's usually
much faster than having to actually send back a copy of the actual resource
being requested.

<figure class="w-figure">
  <img src="./http-cache.png" alt="A diagram of a client requesting a resource and the server responding with a 304 header.">
  <figcaption class="w-figcaption w-text--left">
    A request/response flow. The server uses a 304 Not Modifier header
    to tell the client to use its cached version of a resource.
  </figcaption>
</figure>

{% Aside 'codelab' %}
  [Control resource caching behavior using HTTP headers](/codelab-http-cache).
{% endAside %}

## Dig deeper {: #learn-more }

If you're looking to go beyond the basics of using the `Cache-Control` header,
the best guide out there is Jake Archibald's [Caching best practices & max-age
gotchas](https://jakearchibald.com/2016/caching-best-practices/).

For most developers, though, either `Cache-Control: no-cache` or `Cache-Control:
max-age=31536000` should be fine.

## Caching strategy checklist {: #checklist }

## Caching strategy flowchart {: #flowchart }

![Flowchart](flowchart.png)

[crp]: https://developers.google.com/web/fundamentals/performance/critical-rendering-path
[304]: https://developer.mozilla.org/docs/Web/HTTP/Status/304
[If-Modified-Since]: https://developer.mozilla.org/docs/Web/HTTP/Headers/If-Modified-Since
[If-None-Match]: https://developer.mozilla.org/docs/Web/HTTP/Headers/If-None-Match