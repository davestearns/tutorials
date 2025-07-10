# Cross Origin Resource Sharing (CORS)

> [!IMPORTANT]
> This tutorial assumes you have a good working knowledge of HTTP. If you don't read the [HTTP](http.md) tutorial first.

Back in the early 2000s, web browsers started allowing JavaScript to make HTTP requests while staying on the same page: a technique originally known as [AJAX](https://en.wikipedia.org/wiki/Ajax_(programming)), but today is known as the [`fetch()`](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) function. This was very exciting because it transformed the web browser from a mostly static information viewing tool into an _application platform_. We could now build rich interactive web applications like those we had on the desktop, but without requiring our customers to run an installer and keep the application up-to-date. The browser just automatically downloaded and ran the latest published version of our web application whenever our customers visited our site.

But the browser vendors faced a difficult question: should we allow JavaScript to make requests to a different [origin](http.md#origin) than the one the current page came from? In other words, should JavaScript loaded into a page served from `example.com` be able to make HTTP requests to `example.com:4000` or `api.example.com` or even `some-other-domain.com`?

On the one hand, browsers have always allowed page authors to include script files served from other origins: this was how one could include a JavaScript library file hosted on a [Content Delivery Network (CDN)](https://en.wikipedia.org/wiki/Content_delivery_network). Browsers also let HTML forms `POST` their fields to a different origin to enable "Contact Us" forms that posted directly to an automatic emailing service.

On the other hand, there were some very significant security concerns with allowing cross-origin requests initiated from JavaScript. Many sites use cookies to track [authenticated sessions](authentication.md), which are automatically sent with every request made to that same origin. If a user was signed-in to a sensitive site like their bank, and if that user was lured to a malicious page on `evil.com`, JavaScript within that page could easily make HTTP requests to the user's bank, and the browser would happily send along the authenticated session cookie. If the bank's site wasn't checking the `Origin` request header, the malicious page could conduct transactions on the user's behalf without the user even knowing that it's occurring.

Not surprisingly, the browser vendors decided to restrict cross-origin HTTP requests made from JavaScript. This was the right decision at the time, but it also posed issues for emerging web services like [Flickr](https://en.wikipedia.org/wiki/Flickr), [del.icio.us](https://en.wikipedia.org/wiki/Delicious_(website)), and [Google Maps](https://en.wikipedia.org/wiki/Google_Maps) that wanted to provide APIs callable from _any_ web application served from _any_ origin.

Several creative hacks were developed to make this possible, the most popular being the [JSONP technique](https://en.wikipedia.org/wiki/JSONP). But these were always acknowledged as short-term hacks that needed to be replaced by a long-term solution. The great minds of the Web got together to figure out how to enable cross-origin API servers without compromising security. The result was the [Cross-Origin Resource Sharing](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS) standard, more commonly referred to as CORS.

## How CORS Works

The CORS standard defines new HTTP headers and some rules concerning how browsers and servers should use those headers to negotiate a cross-origin HTTP request from JavaScript. The rules discuss two different scenarios: simple requests; and more dangerous requests that require a separate preflight authorization request.

### Simple Requests

[Simple cross-origin requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS#simple_requests) are defined as follows:

- The method is `GET`, `HEAD`, or `POST`
- The request may contain only "simple" headers, such as `Accept`, `Accept-Language`, `Content-Type`, and `Viewport-Width`.
- If a `Content-Type` header is included, it may only be one of the following:
	- `application/x-www-form-urlencoded` (format used when posting an HTML `<form>`)
	- `multipart/form-data` (format used when posting an HTML `<form>` with `<input type="file">` fields)
	- `text/plain` (just plain text)

If JavaScript in a page makes an HTTP request that meets these conditions, the browser will send the request to the server, adding an `Origin` header set to the current page's origin. The server may use this `Origin` request header to determine where the request came from, and decide if it should process the request.

If the server allows the request and responds with a 200 (OK) status code, it must also include a response header named `Access-Control-Allow-Origin` set to the value in the `Origin` request header, or `*`. This tells the browser that it's OK to let the client-side JavaScript see the response.

[![](https://mermaid.ink/img/pako:eNplkMFOwzAQRH_F2iNq0pA0TWOhSlVAgDgUKZxQLq6zbY0S29gObaj67zgp5YJP3p2n2dGcgKsagYLFzw4lx3vBdoa1lST-aWac4EIz6UhBmCVFI1C6_2I5iKvXZ1Ki-UJzAYpguSwpeXx4I1ODVnWG493GTJdPyjpKmBYhHlmrGwy5akdlbcROSEra_oAbpvUgXMzKwLsVlMRRRNYvI7ziHK0NCiWdUU2wahp1CK4ONyMyaD5w8NZrHC7qRnDmhJLTD6vkiIRh6MNpJS2Sjap7P5NKwgRaNC0Tte_mNESowO2xxQqo_9a4ZV3jKqjk2aOsc6rsJQfqTIcTMKrb7YFuWWP91OmauWuxf9udGbx_eZQ1mkJ10gGNs2wCvtV3pdor4EegJzgCvc3ScD6bZelikaX5Ipp7uPfrKAnjeZLkURqleZzPFucJfI8OUZilySzPkihOk_g2n59_AJSsn88?type=png)](https://mermaid.live/edit#pako:eNplkMFOwzAQRH_F2iNq0pA0TWOhSlVAgDgUKZxQLq6zbY0S29gObaj67zgp5YJP3p2n2dGcgKsagYLFzw4lx3vBdoa1lST-aWac4EIz6UhBmCVFI1C6_2I5iKvXZ1Ki-UJzAYpguSwpeXx4I1ODVnWG493GTJdPyjpKmBYhHlmrGwy5akdlbcROSEra_oAbpvUgXMzKwLsVlMRRRNYvI7ziHK0NCiWdUU2wahp1CK4ONyMyaD5w8NZrHC7qRnDmhJLTD6vkiIRh6MNpJS2Sjap7P5NKwgRaNC0Tte_mNESowO2xxQqo_9a4ZV3jKqjk2aOsc6rsJQfqTIcTMKrb7YFuWWP91OmauWuxf9udGbx_eZQ1mkJ10gGNs2wCvtV3pdor4EegJzgCvc3ScD6bZelikaX5Ipp7uPfrKAnjeZLkURqleZzPFucJfI8OUZilySzPkihOk_g2n59_AJSsn88)

This `Access-Control-Allow-Origin` header protects older servers that were built before the CORS standard, and are therefore not expecting cross-origin requests to be allowed. Since this header was defined with the CORS standard, older servers will not include it in their responses, so the browser will block the client-side JavaScript from seeing those responses.

This made sense for `GET` and `HEAD` requests since they only return information and shouldn't cause any changes on the server. The inclusion of `POST` was a bit problematic---it was added to ensure that existing HTML "Contact Us" forms that posted cross-origin would continue to work. This is why the `Content-Type` is also restricted to those used by HTML forms, and doesn't include `application/json`, which is used when posting JSON to more modern APIs.

Supporting simple cross-origin requests on the server-side is therefore as easy as adding one header to your response: `Access-Control-Allow-Origin: *`. If you want to restrict access to only a registered set of origins, you can compare the `Origin` request header against that set and respond accordingly.

### Preflight Requests

If the client-side JavaScript makes a cross-origin request that doesn't conform to the restrictive "simple request" criteria, the browser does some extra work to determine if the request should be sent to the server. The browser sends what's known as a "preflight request," which is a separate HTTP request for the same resource path, but using the `OPTIONS` HTTP method instead of the actual request method. 

[![](https://mermaid.ink/img/pako:eNqlU02PmzAQ_SuWT60EhIQAwaoiRexho6pLtORQVbk4ZkLcgk1t0900yn-vnS91tR9dqb7gmXm8eeOn2WMmK8AEa_jZg2Bww2mtaLsSyJ6OKsMZ76gwKEdUo7zhIMzzYumKs8UclaB-gToB7qQBJG2Icq8kaKFg0_B6a9C966XPNLk_ndpqsVjOi7sSDShjshdGf1qrwfRWakMQ7XgAj7TtGgiYbI-VQvGaC4La3QOsadddCzPGQGs_l8Io2fjnXv4XMFtZWRVFuXwLdwu0AqUJ-npNzW88FATBSW7pW705QaMwRMXnl5hmTSMf_PfqO6H_pe6EyhVU9vk5baw-o3p4HXodI5fyBwfv-TjoRZNmzPS0uTiEPvANoo4Qqo9P7HJC_98rp9kO5C93Hbhfu4YzargUg-9aijPE6SdvGvAOmr_HP5Ids_arQHdSaEBrWe1sjD3cgmopr-xS7F3LFTZbaGGFib1WsKF9Y1Z4JQ4WSnsjy51gmDg7PKxkX28x2ViHbNR3FTWXjbpma-W4z3gQ1qbcPSImyWToYbtO36RsLwAbYrLHj5gM0zhIxuM0nkzSOJuESerhnU2HUTBKoigL4zDORtl4cvDw7yNDGKRxNM7SKBzF0WiYJYc_g4dJPw?type=png)](https://mermaid.live/edit#pako:eNqlU02PmzAQ_SuWT60EhIQAwaoiRexho6pLtORQVbk4ZkLcgk1t0900yn-vnS91tR9dqb7gmXm8eeOn2WMmK8AEa_jZg2Bww2mtaLsSyJ6OKsMZ76gwKEdUo7zhIMzzYumKs8UclaB-gToB7qQBJG2Icq8kaKFg0_B6a9C966XPNLk_ndpqsVjOi7sSDShjshdGf1qrwfRWakMQ7XgAj7TtGgiYbI-VQvGaC4La3QOsadddCzPGQGs_l8Io2fjnXv4XMFtZWRVFuXwLdwu0AqUJ-npNzW88FATBSW7pW705QaMwRMXnl5hmTSMf_PfqO6H_pe6EyhVU9vk5baw-o3p4HXodI5fyBwfv-TjoRZNmzPS0uTiEPvANoo4Qqo9P7HJC_98rp9kO5C93Hbhfu4YzargUg-9aijPE6SdvGvAOmr_HP5Ids_arQHdSaEBrWe1sjD3cgmopr-xS7F3LFTZbaGGFib1WsKF9Y1Z4JQ4WSnsjy51gmDg7PKxkX28x2ViHbNR3FTWXjbpma-W4z3gQ1qbcPSImyWToYbtO36RsLwAbYrLHj5gM0zhIxuM0nkzSOJuESerhnU2HUTBKoigL4zDORtl4cvDw7yNDGKRxNM7SKBzF0WiYJYc_g4dJPw)

The browser also adds the following headers to the preflight request:

- `Origin` set to the origin of the current page.
- `Access-Control-Request-Method` set to the method the JavaScript is attempting to use in the actual request.
- `Access-Control-Request-Headers` set to a comma-delimited list of non-simple headers the JavaScript is attempting to include in the actual request.

When the server receives the preflight request, it can examine these headers to determine if the actual request should be allowed. If so, the server should respond with a 200 (OK) status code, and include the following response headers:

- `Access-Control-Allow-Origin` set to the value of the `Origin` request header. You can also set it to `*` but this will block the browser from sending cookies in the actual request, so don't do this if you are using [authenticated session](authentication.md) cookies.
- `Access-Control-Allow-Credentials` set to `true` if the server will allow the browser to send cookies during the actual request. If omitted or set to `false`, the browser will not include cookies in the actual request. When set to true, set `Access-Control-Allow-Origin` to the specific origin making the request, not `*`.
- `Access-Control-Allow-Methods` set to a comma-delimited list of HTTP methods the server will allow on the requested resource, or the specific method requested in the `Access-Control-Request-Method` header.
- `Access-Control-Allow-Headers` set to a comma-delimited list of non-simple headers the server will allow in a request for the resource, or the specific ones mentioned in the `Access-Control-Request-Headers`.
- `Access-Control-Expose-Headers` set to a comma-delimited list of **response** headers the browser should expose to the JavaScript if the actual request is sent. If you want the JavaScript to access one of your non-simple response headers (e.g., `Authorization` or `X-Request-ID`), you must include that header name in this list. Otherwise the header simply won't be visible to the client-side JavaScript.
- `Access-Control-Max-Age` set to the maximum number of seconds the browser is allowed to cache and reuse this preflight response if the JavaScript makes additional non-simple requests for the same resource. This cuts down on the amount of preflight requests, especially for client applications that make repeated requests to the same resources.

All of the following must be true for the browser to then send the actual request to the server:

- The `Access-Control-Allow-Origin` response header matches `*` or the value in the `Origin` request header.
- The actual request method is found in the `Access-Control-Allow-Methods` response header.
- The non-simple request headers are all found in the `Access-Control-Allow-Headers` response header.

If any of these are not true, the browser doesn't send the actual request and instead throws an error to the client JavaScript.

## CORS and CSRF Attacks

CORS enabled cross-origin APIs, but it also introduced a new security vulnerability: Cross-Site Request Forgery (CSRF). This is the scenario discussed earlier:

1. A customer is signed into a CORS-enabled API, which uses cookies for authenticated session tokens.
1. The customer is lured to a page on `evil.com`
1. That page contains JavaScript that makes `fetch()` requests to the CORS-enabled API.
1. The browser automatically sends the authentication session token cookie with the request.
1. The CORS-enabled API verifies the session cookie, treats the request as authenticated, and performs a potentially damaging operation.

As discussed in the [Authenticated Sessions](authentication.md#session-token-transmission) tutorial, the only way a CORS-enabled API can defend against such an attack is to use the `Origin` request header. This header is set automatically by the browser to the origin from which the HTML page came. JavaScript in that page can neither change this header nor suppress it, so the server can use it as a security input.

CORS-enabled APIs can use the `Origin` request header in a few ways to protect against CSRF attacks:

- Compare the `Origin` to a set of allowed origins and reject requests with origins not in the set.
- Prefix the session cookie names with a value that is derived from the `Origin`: either a hash of the origin value, or an ID associated with the origin in your database. When reading the session cookie from the request, use the `Origin` to read the particular cookie corresponding to the request origin. This keeps sessions established by different origins separate from each other.

These options are not exclusive---if you track a set of allowed origins, and you want to keep sessions separate per-origin, you can do both. But you should do at least one of these to defend against CSRF attacks.

## CORS Middleware

If you want to enable CORS for your API, most web frameworks offer this as a pre-packaged middleware you can simply add to your application with a bit of configuration. For example, in the Python FastAPI framework, it's as simple as this:

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# List of allowed request origins.
origins = [
    "http://localhost:8080",
    "https://client.one.com",
    "https://client.two.com",
    "https://client.three.com",
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,
    allow_credentials=True,
    allow_methods=["GET", "POST", "DELETE"],
    allow_headers=["Cookie"],
)
```

In the Rust Axum/Tower framework, it looks like this:

```rust
use tower::{ServiceBuilder, ServiceExt, Service};
use tower_http::cors::{Any, CorsLayer};

// Allows everything, including credentials
let cors = CorsLayer::very_permissive();

let mut service = ServiceBuilder::new()
    .layer(cors)
    .service_fn(handle);
```

The process is similarly easy in all web frameworks. Ask your favorite API tool how to enable CORS with your particular web framework.

