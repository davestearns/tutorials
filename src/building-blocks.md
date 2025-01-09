# System Building Blocks

If you were to look at the architectures of the various backend systems that run the web sites you use most often, you would probably notice a lot of similarities. Most will have a set of HTTP servers, with load balancers in front, that respond to requests from clients. Those servers will likely talk to databases, caches, queues, and buckets to manage data. They might also use machine-learning (ML) models to make predictions about that data. Event consumers will respond to new events written to the queues, or changes to those database records. Other jobs will run periodically to archive data, re-train ML models with new data, or perform other routine tasks. What these components _do_ will no doubt vary from system to system, but the _types_ of components used will come from a relatively small set of common building blocks.

This tutorial will give you an overview of these common building blocks, what they are typically used for, what they can and cannot do, and how to use them most effectively. Once you understand these, you can combine them in various ways to build just about any kind of system.

## Load Balancers and API Gateways

When a request is made to your web servers, either by a web browser or by a client calling your application programming interface (API), the first major system component to receive those requests is typically a load balancer. These are fairly generic components, meaning they are typically open-source software programs that can be used "off the shelf" with just a minimal amount of configuration. Popular examples include [NGINX](https://nginx.org/) and [HAProxy](https://www.haproxy.org/).

Load balancers perform several jobs that are critical to building highly-scalable and reliable systems:

* **Load Balancing:** Not surprisingly, the primary job of a load balancer is to distribute requests across a fleet of your "downstream" HTTP servers (i.e., load balancing). The IP addresses for your domain will point to your load balancers, and each load balancer is configured with a set of addresses it can forward those requests to. The set of downstream servers can change over time as you roll out new versions or scale/up down. Balancers typically offer multiple strategies for balancing the load, from simple [round-robin](https://www.cloudflare.com/learning/performance/types-of-load-balancing-algorithms/) to more sophisticated ones that pay attention to how long requests are taking and how many outstanding requests each downstream server already has.
* **Blocking and Rate Limiting:** Load balancers also protect your downstream HTTP servers from attack and abuse. They are specifically designed to handle massive amounts of requests, and can be configured to block particular sources of traffic. They can also limit the number of requests a given clients can make during a given time duration, so that particular clients can't hog all the system resources.
* **Caching:** If your downstream HTTP servers mostly return static content that rarely changes, you can configure your load balancers to cache and replay responses for a period of time. This reduces the load on your downstream servers.
* **[HTTPS](https://en.wikipedia.org/wiki/HTTPS) Termination:** If your load balancer and downstream servers are all inside a trusted private network, and you don't require secure/encrypted connections between your own servers, you can configure your load balancer to talk HTTPS with the Internet, but HTTP with your downstream servers. This used to be a common configuration when CPU speeds made HTTPS much slower than HTTP, but these days (in 2025) it's common to just use HTTPS everywhere.
* **Request/Response Logging:** Load balancers can be configured to log some data about each request and response so that you can analyze your traffic, or diagnose problems reported by your customers.

Although load balancers can be used off-the-shelf with minimal configuration, many now support much more customized behavior via scripting. This customized behavior can be very specific to your use case, but common examples include:

* **Authentication:** If your servers and clients exchange [digitally-signed](crypto.md#digital-signatures) authentication tokens, your load balancer scripts can verify the token signatures, and block obvious attempts to hijack sessions by tampering with those tokens. This reduces obviously fraudulent load on your downstream servers, saving resources.
* **Authorization:** Your scripts could also look up whether the authenticated user has permission to make the request they are making.
* **Request Validation:** If requests contain data that is easy to validate, you can perform those simple validations on your load balancers to block obviously invalid requests before they get to your downstream servers.
* **Request Versioning:** Sometimes you will want to change the API on your downstream servers, but you can't force your existing clients to change their code, so you have to support multiple versions of your API at the same time. Your load balancer scripts can translate version 1 requests into version 2, and version 2 responses back into version 1 responses.

All of this custom functionality _could_ occur within your downstream HTTP servers, but if you can move it into your load balancers, you can reduce the obviously invalid load on your servers, which enables you to handle more legitimate requests with fewer costly resources.

When load balancers become highly-customized for a given system, we often start to refer to them as "API Gateways" instead. But they are still serving the same purpose: the front door or your system, through which all requests must pass.

## HTTP Servers

## Persistent Databases

## Ephemeral Caches

## Data Buckets

## Event Queues

## Event Consumers

## Periodic Jobs

## ML Models

## Consensus Services

## Content Delivery Networks (CDNs)
