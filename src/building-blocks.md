# System Building Blocks

If you were to look behind the scenes of the web sites you use most often, you would probably notice a lot of similarities. Most will have a set of HTTP servers, with load balancers in front, that respond to requests from clients. Those servers will likely talk to databases, caches, queues, and buckets to manage data. They might also use machine-learning (ML) models to make predictions. Event consumers will respond to new events written to the queues, or changes to those database records. Other jobs will run periodically to archive data, re-train ML models with new data, or perform other routine tasks. What these components _do_ will no doubt vary from system to system, but the _types_ of components used will come from a relatively small set of common building blocks.

This tutorial will give you an overview of these common building blocks. We will learn what each is typically used for, and some common options or variations you might see in practice. Once you understand these building blocks, you can combine them in various ways to build just about any kind of system.

## Load Balancers and API Gateways

When a request is made to your web servers, either by a web browser or by a client calling your application programming interface (API), the first major system component to receive those requests is typically a **load balancer**. These are the front door to your system.

Load balancers are fairly generic components, meaning they are typically open-source software programs that can be used "off the shelf" with just a minimal amount of configuration. Popular examples include [NGINX](https://nginx.org/) and [HAProxy](https://www.haproxy.org/). Cloud providers also offer these as hosted services you can simply deploy--for example, [AWS Elastic Load Balancer](https://aws.amazon.com/elasticloadbalancing/) or [Azure Load Balancer](https://learn.microsoft.com/en-us/azure/load-balancer/load-balancer-overview).

Load balancers perform several jobs that are critical to building highly-scalable and reliable systems:

* **Load Balancing:** Not surprisingly, the primary job of a load balancer is to distribute requests across a fleet of your "downstream" HTTP servers (i.e., load balancing). You configure the IP addresses for your domain to point to the load balancers, and each load balancer is configured with a set of addresses it can forward those requests to. The set of downstream addresses can then change over time as you roll out new versions or scale up/down. Balancers typically offer multiple strategies for balancing the load, from simple [round-robin](https://www.cloudflare.com/learning/performance/types-of-load-balancing-algorithms/) to more sophisticated ones that pay attention to how long requests are taking and how many outstanding requests each downstream server already has.
* **Blocking and Rate Limiting:** Load balancers also protect your downstream HTTP servers from attack and abuse. They are specifically designed to handle massive amounts of requests, and can be configured to block particular sources of traffic. They can also limit the number of requests a given clients can make during a given time duration, so that particular clients can't hog all the system resources.
* **Caching:** If your downstream HTTP servers mostly return static content that rarely changes, you can configure your load balancers to cache and replay responses for a period of time. This reduces the load on your downstream servers.
* **Request/Response Logging:** Load balancers can be configured to log some data about each request and response so that you can analyze your traffic, or diagnose problems reported by your customers.
* **[HTTPS](https://en.wikipedia.org/wiki/HTTPS) Termination:** If your load balancer and downstream servers are all inside a trusted private network, and you don't require secure/encrypted connections between your own servers, you can configure your load balancer to talk HTTPS with the Internet, but HTTP with your downstream servers. This used to be a common configuration when CPU speeds made HTTPS much slower than HTTP, but these days (in 2025) it's common to just use HTTPS everywhere.

Although load balancers can be used off-the-shelf with minimal configuration, many now support much more customized behavior via scripting. This customized behavior can be very specific to your use case, but common examples include:

* **Authentication:** If your servers and clients exchange [digitally-signed](crypto.md#digital-signatures) authentication tokens, your load balancer scripts can verify the token signatures, and block obvious attempts to hijack sessions by tampering with those tokens. This reduces obviously fraudulent load on your downstream servers, saving resources.
* **Authorization:** Your scripts could also look up whether the authenticated user has permission to make the request they are making.
* **Request Validation:** If requests contain data that is easy to validate, you can perform those simple validations on your load balancers to block obviously invalid requests before they get to your downstream servers.
* **Request Versioning:** Sometimes you will want to change the API on your downstream servers, but you can't force your existing clients to change their code, so you have to support multiple versions of your API at the same time. Your load balancer scripts can translate version 1 requests into version 2, and version 2 responses back into version 1 responses.

All of this custom functionality _could_ occur within your downstream HTTP servers, but if you can move some of it into your load balancers, you can block obviously invalid requests early, reducing the load on your downstream servers. Load balancers are typically written in a high-performance language like C++ or Rust, so they can often handle much more load than downstream servers written in less-performant scripting languages like Python, Ruby, or JavaScript.

When load balancers become highly-customized for a given system, we often start to refer to them as **API Gateways** instead to reflect that they are specific to a given API. But they still serve the basic jobs described above, acting as the front door to the rest of your system.

## HTTP Servers

Load balancers forward requests to one or more **HTTP servers**. These are simply programs that run continuously, listen for incoming requests on a given network port number, and write responses. 

What makes them HTTP servers in particular is that the requests and responses adhere to the [Hypertext Transfer Protocol (HTTP)](http.md) standard. This started out as a really simple text-based protocol, but has since gotten a bit more complicated. Thankfully it's still very easy to build one, which is probably why they are the most common building-block you'll see in transaction processing systems.

HTTP requests can contain several pieces of information, but the two critical ones are the **resource path** and the **method** the client wants to perform on that resource. Resource paths are the "nouns" of the request. They are just arbitrary strings that name specific resources the server can manipulate, such as files, database records, other servers, or even specialized hardware. Methods are the "verbs" or operation the client wants to perform on those resources. For example, a `GET /` might be asking to read the root resource of the server, while a `PUT /foo` request might be asking to update the content of the `/foo` resource

Some HTTP servers are really just glorified file servers: the resource paths map directly to files on the server's disk, and clients can receive a copy of them by making HTTP `GET` requests for specific file paths. These files are typically those that comprise web pages: HTML, CSS, various image and video formats, and JavaScript.

But these days (in 2025) it's more common to build HTTP servers that send and receive _raw data_ instead of formatted content. This raw data is still encoded in some sort of easily interpreted text-based format, such as [JSON](https://www.json.org/json-en.html), but clients are then free to do with it what they want. This decoupling is very powerful, as it allows the same HTTP server to support clients on multiple types of application platforms: web apps, native mobile apps, VR simulations, AI agents, or whatever springs up in the future.

Requests to these data-only HTTP servers still include a resource path and method, but that resource path no longer maps to a file on disk. Instead it could map to a database table, or a specific record in that table, or some procedure/function in the server's code that the client can execute. The method might indicate a [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) operation to perform on the database table/record, or it might a read-only vs read-write mode. 

The set of resource paths and methods supported by your server are known as its Application Programming Interface (API). What those look like are entirely up to you, but most systems use one of a few common patterns:

* **[REST](rest.md):** An API style where the resource paths name objects/records, or collections of those, and the methods indicate the CRUD operation the client wishes to perform on them. The [original articulation of this style](https://ics.uci.edu/~fielding/pubs/dissertation/fielding_dissertation.pdf#page=94) is much more powerful and nuanced, but very few systems actually implement it in full. Nevertheless, this is the most common API style used today (2025).
* **RPC:** An API style where the resource path names a specific procedure or method to invoke on the server. The HTTP method doesn't really matter much in this style, so it's commonly just `POST` for all requests. This style is often used for _internal_ services, and less so for public APIs, but it is more appropriate for systems that expose 'actions' more than 'resources'.
* **[GraphQL](https://graphql.org/):** An API style that is similar to making SQL queries against a database--servers expose a graph of objects, and clients can query that graph using a structured syntax. This is commonly used by social media systems, as they can't often predict what clients will need from their extensive content graph.

Because these kinds of HTTP servers expose an API, they are often referred to as "API servers." Strictly speaking, they are "HTTP API servers" but HTTP has become such a default protocol for API servers that people often just leave off the "HTTP" part.

You will also hear people refer to these kinds of HTTP API servers as simply "services" because each one provides a particular service to the rest of the system. This also keeps one's focus on the _logical_ service being provided, as opposed to the _physical_ runtime packaging, which could change over time.

## Persistent Databases



## Ephemeral Caches

## Data Buckets

## Event Queues

## Event Consumers

## Periodic Jobs

## ML Models

## Consensus Services

## Content Delivery Networks (CDNs)
