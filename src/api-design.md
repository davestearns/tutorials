# API Design

In the previous tutorial on [API Servers](api-servers.md) we learned how to use a layered architecture to separate concerns, keeping the code easy to understand and extend. Now it's time talk about what the API actually looks like to callers. Since this is an HTTP server, the caller ultimately must send HTTP requests and handle HTTP responses, but we have a lot of options for what those requests and responses actually look like.

Over the years a few common API design patterns have emerged, and these days there are really only three that are typically used: REST, RPC, and GraphQL. Let's look at each in turn.

## REST

REST is an acronym that stands for **Re**presentational **S**tate **T**ransfer, which is the name of a design pattern first described by [Roy Fielding](https://en.wikipedia.org/wiki/Roy_Fielding) in his [PhD dissertation](https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm). The full details of this pattern are complex and elegant, and few systems follow it completely, but the basic ideas became a very popular choice for API servers over the last couple of decades.

In its most basic form, a REST API exposes a set of logical **resources**, which are the core objects your API server can manipulate. In a social media system these would be things like Accounts, Posts, DirectMessages, Feeds, Notifications, etc.

Callers refer to these resources using the [resource path](http.md#resource-path) in the HTTP request. This path can be hierarchical, which makes it easy to refer to the set of objects as a whole, or one specific object in the set, or even other objects related to a specific object. For example:

Resource Path | Meaning
--------------|--------
`/accounts` | All accounts in the system
`/accounts/123` | The specific account with the identifier `123`
`/sessions/me` | The currently authenticated account
`/accounts/123/friends` | All accounts that are friends of account `123`
`/accounts/123/posts` | All posts made by account `123`

The words used in resource paths should always be _nouns_ or unique identifiers. Resource paths with verbs, like `/signin` or `/send_email`, should be avoided. Use `POST /sessions` for signing in, and something like `POST /outbox` to send an email.

A REST API also exposes a set of basic **operations** that callers might be able to perform on these resources. The operation is specified as the [method](http.md#methods-and-resources) in the HTTP request. In _theory_ this method could be anything the client and server understand, but in practice the set of methods you can use is often dictated by proxy servers in-between. To be safe, REST APIs tend to use only the core HTTP methods that every server and proxy should support:

Method | Meaning
-------|--------
GET | return the current state of the resource
PUT | completely replace the current state of the resource with the state in the request body
PATCH | partially update the current state of the resource with the partial state in the request body
POST | add a new child resource with the state in the request body
DELETE | delete the resource
OPTIONS | list the methods the current user is allowed to use on the resource

When sending or returning resource state, it should be some text-based format that is easy to work with in the browser as well as other types of clients like native mobile apps. The default choice these days is to use [JSON](https://www.json.org/json-en.html). Because of this, we often refer to REST APIs as REST/JSON APIs.

Combining methods and resource paths, callers can do a wide variety of things:

Method & Path | Meaning
--------------|--------
`POST /accounts` | Create a new system account (sign up)
`POST /sessions` | Create a new authenticated session (sign in)
`GET /sessions/me` | Get the details for the currently authenticated account
`GET /accounts/123/posts` | Get all posts made by account `123`
`POST /posts` | Create a new general post from the currently authenticated account
`DELETE /posts/123` | Delete a previously-created post with the ID `123`
`POST /channels/abc/posts` | Create a new post that is only visible in channel `abc`
`DELETE /sessions/mine` | Delete the currently authenticated session (sign out)

`GET` requests against large resource collections will typically return only a page of resources at a time (e.g., 100 records at a time). Otherwise your API server and database will bog down trying to read and transmit far too much data during a single request. Clients can iteratively move through these pages by specifying a `?last_id=X` [query string parameter](http.md#query-string-parameters-and-values) containing the last ID they got in the previous page. It's common to restrict how many pages one can retrieve in total, so that bots can't simply scrape all of your data.

Large resource collections might also provide limited forms of filtering and sorting via other query string parameters. For example, one might be able to get all posts made during during January 2025 using a request like `GET /accounts/123/posts?between=2025-01-01_2025-02-01`.

The set of method and resource combinations you support becomes your API. Some combinations might be available only during an authenticated session, and some might be allowed only if the authenticated account has permissions to perform the operation (e.g., only the creator of a post can delete it).

When done well, REST APIs are simple, intuitive, and ergonomic. But the REST pattern has some drawbacks:

- It's difficult or clumsy to model more complex operations that don't neatly correspond to the basic HTTP verbs. For example, an API for controlling an audio system might want to expose operations for pausing and resuming the current playlist, but there are no standard HTTP methods for that. One could expose a `PATCH /playlists/current` API to which the client can send `{ "paused": true}`, but that is fairly inelegant and obscure. APIs that needs to do this should consider an [RPC](#rpc) style instead.
- As the size of a resource's state grows, more and more data is sent to clients, even if they only need a small fraction of it. This gets even worse when returning lists of those resources. One can support partial projections through a query string parameter, but this again becomes clumsy and inelegant. This scenario was one of the motivations for [GraphQL](#graphql).
- If a client needs several different resources all at once, it needs to make several HTTP requests to different resource paths. If the state of one resource determines the resource paths for others, the requests must be done sequentially, which slows down rendering, making the client feel sluggish. This scenario was also a key motivation for [GraphQL](#graphql).
- How does a client know which methods and resource paths are available? And how do they know what kind of data to send in a request, and what shape of data they will receive in the response? Some frameworks can generate this sort of documentation automatically from your code, but with others you have to write the documentation manually. [RPC](#rpc) and [GraphQL](#graphql) APIs are typically self-documenting.

## RPC

Long before the REST pattern became popular, various kinds of servers exposed APIs that looked more like a set of functions or procedures that clients could invoke remotely. These APIs were just like the ones exposed from internal services, but clients could now call them across a network. This pattern is known as **Remote Procedure Calls**, or RPC, and it actually pre-dates HTTP, but has been adapted to HTTP in recent years.

The most popular implementation of this pattern on HTTP is Google's [**gRPC**](https://grpc.io/). It defines a high-level universal language for describing your API, and includes tooling to generate the corresponding code in a wide variety of languages. It builds upon Google's binary data encoding standard, [Protocol Buffers (protobuf)](https://protobuf.dev/), which is used to define and encode/decode the data structures passed on the wire.

For example, say you wanted to expose an API that could return the basic [Open Graph](https://ogp.me/) properties for a given URL, so that the caller can display a preview card like the ones you see in a social media app. The service definition would look something like this:

```proto
service PreviewExtractor {
	// Extracts preview information for a given URL
	rpc Extract(ExtractRequest) returns (Preview) {}
}

message ExtractRequest {
	// The URL from which to extract preview properties
	string url = 1;
}

// Properties about a URL suitable for 
// showing in a preview card
message Preview {
	// The URL from which these properties were extracted
	string url = 1;
	// The type of content returned from the URL
	string content_type = 2;
	// A title for this content
	optional string title = 3;
	// Zero, one, or multiple preview images
	optional repeated Image preview_image = 4;
}

message Image {
	// The URL that will return this preview image
	string url = 1;
	// The mime type of the image (jpg, png, tiff, etc.)
	string mime_type = 2;
	// The width of the image if known
	optional uint32 width = 3;
	// The height of the image if known
	optional uint32 height = 4;
	// A textural description of the image
	optional string alt_description = 5;
}
```

When you run this through the gRPC tooling, it will generate classes in your specified programming language for each `message` defined in the file. It will also generate an empty `PreviewExtractor` service implementation that you can fill out for the server, as well as a stub class that clients can use to call the procedures. For example, the Python calling code looks as simple as this:

```python
# Connect to the server and create the stub once.
with grpc.secure_channel('...net address of server...', credentials) as channel:
    preview_extractor = PreviewExtractorStub(channel)

    # Calling an API then looks like calling a local method.
    preview = preview_extractor.Extract(ExtractRequest(url='https://ogp.me'))

    print(f'page title is {preview.title or "(No Title)"}')
```

The calling stub makes it look to client code like they are just calling a method on a class, but under the hood, the stub class actually makes an HTTP request to the server. The resource path contains the name of the procedure to run, and the request body contains the input message(s) encoded into [protobuf format](https://protobuf.dev/programming-guides/encoding/). The HTTP response body will similarly contain the message returned by the procedure, and the client stub will decode this back into an instance of the class generated for your programming language.

A gRPC API has a few natural advantages:

- The service definition file is effectively self-documenting.
- It is well supported in all the popular programming languages (especially those used by Google).
- Everything is statically typed. The procedures exposed by the service are real methods on the stub, and all inputs and return values are generated classes with explicit properties/methods. This allows IDEs to do statement completion, and compilers or type checkers to catch typos.
- Protobuf encoding is much more compact than JSON, so gRPC tends to be a bit faster than REST/JSON especially when the requests and responses include arrays of objects with many properties.
- It's easy to model APIs that are more action-oriented than resource-oriented.
- Because it is built upon HTTP/2, it supports bidirectional streaming without requiring WebSockets.

The only real drawback of gRPC is that (as of 2025) it's not possible to directly call a gRPC API from JavaScript running in a web browser. Native mobile apps, command line utilities, and other servers can easily call gRPC APIs, but you can't do so directly using the `fetch()` API in the browser.

There are a few options for working around this limitation, however. One of the most popular is [gRPC Web](https://github.com/grpc/grpc-web), which requires a separate HTTP proxy server sitting between the browser and your gRPC server. This proxy handles converting text-based request/response bodies into protobuf, and switching from HTTP/1 to 2 when necessary.

Another option is [gRPC Gateway](https://github.com/grpc-ecosystem/grpc-gateway), which also requires a separate HTTP proxy server, but this server effectively translates your gRPC API into a REST/JSON one. This translation can't be done automatically, so you do have to provide a good bit of configuration, but once you do, browser apps can go through the REST/JSON proxy, while all other clients can use your gRPC API directly.

## GraphQL

The REST APIs for large social media sites can return _a lot_ of data. Part of this is because they can't really know ahead of time what any given client might need, so they just return _everything_ they know about a resource. The response bodies of these APIs can get enormous, which slows down the network processing and can make the client feel sluggish. And if the client needs multiple resources in order to render a screen, performance gets even worse.

But these REST APIs are ultimately making queries to a database, and those databases already support a flexible query language that lets the caller specify which fields they actually want. They even let you execute multiple queries in one round-trip. So what if we applied those same techniques to our HTTP APIs?

Thus was born [GraphQL](https://graphql.org/learn/). It's essentially a query language for a graph database exposed through an HTTP API. Clients can ask for only the properties they really need, and can fetch multiple related resources all in one HTTP request. In addition to flexible querying, GraphQL APIs can also support [mutations](https://graphql.org/learn/mutations/) through syntax that looks a bit like gRPC.

Unfortunately, when GraphQL was first introduced it received _a lot_ of hype, which caused many engineers to use it regardless of whether it made any sense for their particular API. If your system doesn't have the needs that motivated its creation, GraphQL APIs can actually be harder to implement, complicated to use, and tricky to make performant. Eventually some sanity returned, and engineers realized that it's not _always_ the right choice, which led to even sillier proclamations that it was now dead.

Don't listen to hype cycles. If your API exposes a lot of data that can be organized into a graph, and you want to support clients with unpredictable needs, GraphQL might be a good choice. If not, REST or gRPC might be a better choice. Use the right tool for the job!

## Idempotency

Regardless of which style you choose, if your API allows creating new data, you need to handle the following scenario:

1. Client makes a request to your data creation API.
1. Your server receives and processes that request.
1. But for whatever reason, the client never receives the response. This can happen if your server takes longer to respond than the client's configured timeout, or if there is a network partition/failure that blocks the response from getting back to the client.
1. The client now has no way of knowing whether the operation succeeded or not. How can it safely retry the request without creating duplicate data?

In some systems, creating duplicate data may be OK. For example, if you end up posting the same picture twice to a photo sharing site, no real harm is done, and the user can always delete the duplicate post later. But if you are talking to a payments API, you really don't want to charge your customers payment card twice!

One way to make it safe to retry data creation requests is to make that request **idempotent**. Idempotent operations result in the same outcome whether they are executed just once, or multiple times. Read and delete operations are naturally idempotent, and update operations can be, but data creation operations need something extra to make them idempotent.

That extra thing is some unique value that the client sends with the original request, as well as any retries of that same operation. We often call this an **idempotency key** but others may call it a transaction ID or a logical request ID. Regardless, it is just some value (typically a [UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier)) that is unique for each data creation operation, and included in all retries of that same operation.

This idempotency key allows your API server to disambiguate between new data creation operations it hasn't yet seen, and retries of a previously-processed operation. If you see a request with an idempotency key you've already processed, your API server can stop processing and simply return a successful response.

There are two primary options for how to implement this:

- Use a [cache](building-blocks.md#caches) to track all the idempotency keys you've seen within the last hour (or whatever you want your idempotency duration to be). Each time you receive a request, check the idempotency key against your cache to see if you've already processed it. This works best when your idempotency duration is limited to a relatively short period of time.
- Save the idempotency key with the record created by the operation, and add a unique constraint to that field. If you try to insert the same record again with the same idempotency key, the database will reject the operation, and you can catch/handle that exception in your code. This works best when you want to enforce idempotency for the lifetime of the data created by the operation.

Regardless of which option you choose, make it clear in your API documentation how you support idempotency and how to use it. Your customers will thank you!
