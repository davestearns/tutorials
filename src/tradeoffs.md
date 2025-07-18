# Trade-Offs and Trapdoors

When you build a new system from scratch, you have to make _a lot_ of decisions:

- Which programming languages should I use?
- Which framework should I use for my API server?
- Should I build "serverless" functions or continuously-running servers?
- Which database should I use?
- Should I build a web client, native mobile apps, or both?
- Which framework should I use for my web client?
- Should I build my mobile apps using native platform languages and tools, or using a cross platform framework?

It can be a bit overwhelming. Unfortunately, there is no one right answer to any of these questions beyond "it depends." Perhaps a better way to say that is:

> Every decision in software engineering is a trade-off that optimizes for some things at the cost of other things. And while some of these decisions can be easily reversed in the future, others are trapdoors.

This implies two questions to answer when making these kinds of decisions:

- **What do I want to optimize for?** What will actually make a difference toward achieving my goals, and what doesn't really matter right now? Optimize for the things that actually matter, and don't worry about the rest until you need to.
- **How easily could this decision be reversed in the future if necessary?** If it's easy to reverse, don't sweat it: make the appropriate tradeoff based on your current situation and move on. But some decisions are more like trapdoors in that they become prohibitively expensive to change after a short period of time. These can become a millstone around your neck if you make the wrong bet, so they require more consideration.

Let's put these principles into practice by analyzing a few common trade-offs system builders have to make early on, many of which can become trapdoors. For each we will consider what qualities you could optimize for, and which would be most important given various contexts. For those that could be trapdoors, I'll also discuss a few techniques you can use to mitigate the downsides of making the wrong decision.

## Choosing Programming Languages and Frameworks

One of the earliest choices you have to make is which programming languages and frameworks you will use to build your system. Like all decisions in software, this is a trade-off, so it ultimately comes down to what you want to optimize for:

- **Personal Familiarity:** Founding engineers will typically gravitate towards languages and frameworks they already know well so they can make progress quickly. This is fine, but you should also consider...
- **Available Labor Pool:** If your system becomes successful, you will likely need to hire more engineers, so you should also consider how easy it will be to find, hire, and onboard engineers who can be productive in your chosen languages/frameworks. For example, the number of experienced Java or Python engineers in the world is far larger than the number of experienced Rust or Elixir engineers. It will be easier to find (and maybe cheaper to hire) the former than the latter.
- **Legibility:** Code bases written in languages that use static typing (or type hints) are typically more legible than those that use dynamic typing. This means that it's easier for new engineers to read and understand the code, enabling them to be productive sooner. IDEs can also leverage the typing to provide features like statement completion, jump-to-source, and informational popups. Rust, Go, Java, Kotlin, and TypeScript are all statically-typed, while plain JavaScript is not. Languages like Python and Ruby have added support for optional type hints that various tooling can leverage to provide similar legibility features, but they are not required by default.
- **Developer Experience and Velocity:** How quickly will your engineers be able to implement a new feature, or make major changes to existing ones? Some languages simply require less code to accomplish the same tasks (e.g., Python or Ruby vs Java or Go), and some libraries make implementing particular features very easy. Static typing makes it easier to refactor existing code, and good compilers/linters make it easier to verify that refactors didn't break anything. All of these lead to more efficient and happy engineers, which shortens your time-to-market.
- **Runtime Execution Speed:** A fully-compiled language like Rust, Go, or C++ will startup and run faster than JIT-compiled languages like Java and interpreted languages like Python or JavaScript. But this only really matters if your system is more compute-bound than I/O bound---that is, it spends more time executing the code you wrote than it does waiting for databases, network requests, and other I/O to complete. Most data-centric systems are actually I/O-bound, so the latency benefits of a fully-compiled language may or may not matter for them.
- **Async I/O Support:** The thing I/O-bound systems do benefit from greatly, however, is support for _asynchronous_ I/O. With synchronous I/O, the CPU thread processing a given API server request is blocked while it waits for database queries and other network I/O to complete. With async I/O, the CPU can continue to make progress on other requests while the operating system waits for the database response. This greatly improves the number of concurrent requests a given server can handle, which allows you to scale up for less cost. Languages like Go and Node do this natively. Java, Kotlin, and Python have added support for it in the language, but older libraries and frameworks may still use synchronous I/O.
- **Runtime Efficiency:** Generally speaking, languages like Rust and C++ require less CPU and RAM than languages like Python or Ruby to accomplish the same tasks. This allows you to scale up with less resources, resulting in lower operating costs, regardless of whether you are using async I/O.
- **Libraries and Tooling:** Sometimes the choice of language is highly-swayed by particular libraries or tooling you want to use. For example, machine-learning pipelines often use Python because of libraries like NumPy and Pandas. Command-line interfaces (CLIs) like Docker and Terraform are typically written in Go because the tooling makes it easy to build standalone executables for multiple platforms.
- **Longevity:** Languages, and especially frameworks, come and go. What's hot today may not be hot tomorrow. But some languages (C++, Java, Node, Python, Go) and frameworks have stood the test of time and are used pervasively. They are already very optimized, battle-tested, well-documented, and likely to be supported for decades to come. It's also relatively easy to find engineers who are at least familiar with them, if not experienced in using them.

Which of these qualities you optimize for will probably depend on your particular context:

- **Young Startup:** If you are a young startup looking for product-market fit (PMF), you should definitely optimize for developer velocity in the short run. If you end up running out of money because it took too long to build the right product, it won't matter which languages you chose. But if you do succeed, you will need to hire more engineers, which is when you want to optimize for legibility and available labor pool as well.
- **Low-Margin Business:** If you're building a system to support a low-margin business, you might want to optimize for qualities that allow you to scale up while keeping operating costs low: async I/O, runtime efficiency, larger labor pool with cheaper engineers, etc.
- **Critical-Path Component:** If you are building a system component that needs to be very efficient and high performance, you should obviously optimize for those qualities over others like developer velocity.
- **Open-Source Project:** If you are starting a new open-source project, you might want to optimize for available labor pool and legibility, so that you can attract and retain contributors.

If you just want some general advice, I generally follow these rules:

- **Favor statically-typed languages:** The benefits of static typing far outweighs the minor inconvenience of declaring method argument and return types (most languages automatically work out the types of local variables and constants). Code bases without declared types are fragile and hard to read.
- **Use async I/O when I/O-bound:** If your API servers spend most of their time talking to databases and other servers, use a language, framework, and libraries that all have solid support for async I/O. This will improve performance and allow you to scale with less operational cost. But beware of languages where async I/O was recently bolted-on, as most frameworks and libraries won't support it yet.
- **Avoid trendy but unproven languages/frameworks:** What works in a proof-of-concept may not work in production at scale. Choose languages and frameworks that have a proven track record.

When I build I/O-bound API servers or message queue consumers, I generally prefer Go or TypeScript on Node. Rust is very performant and efficient, and has a fabulous tool chain, but it's difficult to learn so it's harder to find other engineers who can be productive in it quickly. Java or Kotlin are also fine choices if you use a framework with good async I/O support (e.g., Reactor or Spring WebFlux). The same could be said for Python if you require the use of async I/O and type hints.

Regardless of which languages you choose, this decision will likely be a trapdoor one. Once you write a bunch of code, it becomes very costly and time-consuming to rewrite it in another language (though AI might make some of this more tractable). But there are a few techniques you can use to mitigate the costs of changing languages in the future:

- **Separate System Components:** Your API servers, message consumers, and periodic jobs will naturally be separate executables, so they _can_ be implemented in different languages. Early on you will likely want to use the same language for all of them so you can share code and reduce the cognitive load. But if you decide to change languages in the future, you can rewrite the components incrementally without having to change the others at the same time.
- **Serverless Functions:** If it makes sense to build your system with so-called ["serverless" functions](building-blocks.md#serverless-functions), those are also separate components that can be ported to a new language incrementally. API gateways and other functions call them via network requests, so you can change the implementation language over time without having to rewrite the callers as well.
- **Segmented API Servers:** If you have a single API server and decide to rewrite it in a different language, you can do so incrementally by segmenting it into multiple servers behind an [API gateway](building-blocks.md). Taken to an extreme, this approach is known as **microservices** and it can [introduce more problems than it solves](https://www.shopify.com/enterprise/blog/disadvantages-microservices#4), but you don't have to be so extreme. Assuming you are just changing languages, and don't need to scale them differently, the segmented servers can still all run on the same machine, and communicate with each other over an efficient local inter-process channel such as [Unix domain sockets](https://en.wikipedia.org/wiki/Unix_domain_socket).

## Choosing Databases

Another critical decision system builders need to make early on is which kind of database to use. This also quickly becomes a trapdoor decision once you start writing production data to it---at that point, switching databases can get very complicated, especially if your system can't have any downtime.

Like all decisions in software, choosing a database is all about making trade-offs. But unlike programming languages, the set of things you can optimize for tends to collapse into just a few dimensions:

- **Administration vs Operating Cost:** In the bad old days, we had to run our own database instances, monitor them closely, scale up and partition manually, and run periodic backups. As the amount of data grew, this typically required hiring a dedicated team of database administrators (DBAs). These days you can outsource this to the various cloud providers (AWS, Google, Azure) by using one of their hosted auto-scaling solutions---for example, DynamoDB, AWS Aurora Serverless, Spanner, or Cosmos DB. These typically cost more to run than a self-hosted solution, but you also don't have to hire a team of DBAs, nor do any ongoing administration.
- **Features vs Scalability and Performance:** Very simple key/value stores like DynamoDB can scale as much as you need while maintaining great performance. But they also don't offer much in the way of features, so your application logic often has to make up for that. Thankfully, cloud providers have recently made this trade-off less onerous by offering hosted auto-scaling relational database solutions that are quite feature-rich (e.g., AWS Aurora Serverless and Spanner).
- **Database vs Application Schema:** Relational databases have fairly rigid schemas, which need to be migrated when you add or change the shape of tables. Document-oriented databases like DynamoDB or MongoDB allow records to contain whatever properties they want, and vary from record to record. This flexibility can be handy, but the truth is that you must enforce _some_ amount of schema _somewhere_ if you want to do anything intelligent with the data you store in your database. If you can't count on records having a particular shape, you can't really write logic that manipulates or reacts to them. So the trade-off here is really about _where_ (not _if_) you define schema and handle migration of old records. With relational databases you do this in the database itself; with document-oriented databases you do this in your application code.

Which side you optimize for on these trade-off dimensions will likely depend on your context, but these days (2025), using a hosted auto-scaling solution is almost always the right choice. The only reason to run your own database instances is if you are a low-margin business and you think you can do it cheaper than the hosted solution.

The auto-scaling relational solutions like Spanner or AWS Aurora Serverless also make the second trade-off less relevant than it used to be. These days you can get automatic scaling with consistent performance _and_ have access to most of the powerful relational features: flexible queries with joins, range updates and deletes, pre-defined views, multiple strongly-consistent indexes, constraints, etc.

If you need those relational features, then that determines the last trade-off as well: schema will be defined in the database and migrations will typically be done by executing Data Definition Language (DDL) scripts. But if you don't need relational features, you can define your schema and implement migration logic in your application code instead.

Although choosing a database is a trapdoor decision, you can make it easier to change databases in the future by using a [layered internal architecture](api-servers.md#internal-architecture) that isolates your database-specific code in the persistence layer. If you change databases, you only need to rewrite that layer---all the layers above shouldn't have to change.

Moving the existing production data to your new database can be more challenging. If you can afford a small bit of downtime, you can shut down your API servers, copy the data to the new database, and deploy the new version of your API servers pointed at the new database. But if you can't afford downtime, you can do dual-writes while you cut over to the new database---that is, you send inserts and updates to both the new and old databases to keep them in-sync while you shift which database is the source of truth. Once the cut-over is complete, you stop writing to the old database and only write to the new one.

# Conclusion

Every decision in software engineering is a trade-off that optimizes for some things at the cost of other things. Some of those decisions can be easily reversed in the future, but some are more like trapdoors that become prohibitively expensive to change after a short period of time.

Making good decisions can be tricky, but if you focus on what you actually need to optimize for, as opposed to what would be nice to have, that will often lead you to the correct choice.
