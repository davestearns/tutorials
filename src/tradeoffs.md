# Trade-Offs and Trapdoors

When you build a new system from scratch, you have to make _a lot_ of decisions:

- Which programming languages should I use?
- Which framework should I use for my API server?
- Should I build "serverless" functions or continuously-running servers?
- Which style of API should I build?
- Which database should I use?
- How should I authenticate customers and authorize requests?
- Should I build a web client, native mobile apps, or both?
- Which framework should I use for my web client?
- Should I build my mobile apps using native platform languages and tools, or using a cross platform framework?

It can be a bit overwhelming. Unfortunately, there is no one right answer to any of these questions beyond "it depends." Perhaps a better way to say that is:

> _Every decision in software engineering is a trade-off that optimizes for some things at the cost of other things. And while some of these decisions can be easily reversed in the future, others are trapdoors._

This implies two questions to answer when making these kinds of decisions:

- **What do I want to optimize for?** What will actually make a difference toward achieving my goals, and what doesn't really matter right now? Optimize for the things that actually matter, and don't worry about the rest until you need to.
- **How easily could this decision be reversed in the future if necessary?** If it will be fairly easy to reverse the decision in the future, don't sweat it: make the appropriate tradeoff based on your current situation and move on. But some decisions are more like trapdoors in that they become prohibitively expensive to change after a short period of time. These can become a millstone around your neck if you make the wrong bet.

Let's put these principles into practice by analyzing a few common trade-offs system builders have to make early on, many of which can become trapdoors. For each we will consider what qualities you could optimize for, and which would be most important given various contexts. For those that could be trapdoors, I'll also discuss a few techniques you can use to mitigate the downsides of making the wrong decision.

## Choosing Programming Languages and Frameworks

One of the earliest choices you have to make is which programming languages and frameworks you will use to build your system. This is a trade-off, so it ultimately comes down to what you want to optimize for:

- **Personal Familiarity:** Founding engineers will typically gravitate towards languages and frameworks they already know well, as this helps them make progress quickly. This is fine, but you should also consider...
- **Available Labor Pool:** If your system becomes successful, you will likely need to hire more engineers, so you should also consider how easy it will be to find, hire, and onboard engineers who can be productive in your chosen languages/frameworks. For example, the number of experienced Java or Python engineers in the world is far larger than the number of experienced Rust or Elixir engineers. It will be easier to find (and maybe cheaper to hire) the former than the latter.
- **Runtime Execution Speed:** A fully-compiled language like Rust, Go, or C++ will startup and run faster than JIT-compiled languages like Java and interpreted languages like Python or JavaScript. Languages with explicit memory control, like Rust and C++, also have more consistent latency than garbage-collected languages like Go and Java. But this only really matters if your system is more compute-bound than I/O bound---that is, it spends more time executing the code you wrote than it does waiting for databases, network requests, and other I/O to complete. Most data-centric systems are actually I/O-bound, so the latency benefits of a fully-compiled language may or may not matter for them.
- **Async and Concurrent I/O Support:** The thing I/O-bound systems do benefit from greatly, however, is support for _asynchronous_ and _concurrent_ I/O. With synchronous I/O, the CPU thread processing a given API server request is blocked while it waits for database queries and other network I/O to complete. With async I/O, the CPU can continue to make progress on other requests while the operating system waits for the database query response. This greatly improves the number of concurrent requests a given server can handle, which allows you to scale up for less cost. You can also execute multiple independent I/O operations concurrently, which may improve your latency. Languages like Go and Node do this natively. Java, Kotlin, and Python have added support for it in the language runtime, but older libraries and frameworks may still use synchronous I/O.
- **Runtime Efficiency:** Generally speaking, languages like Rust and C++ require less CPU and RAM than languages like Python or Ruby to accomplish the same tasks. This allows you to scale up with less resources, resulting in lower operating costs, regardless of whether you are using async I/O.
- **Legibility:** Code bases written in languages that use static typing (or type hints) are typically more legible than those that use dynamic typing. This means that it's easier for new engineers to read and understand the code. IDEs can also leverage the typing to provide features like statement completion, jump-to-source, and informational popups. Rust, Go, Java, Kotlin, and TypeScript are all statically-typed, while plain JavaScript is dynamically-typed. Languages like Python and Ruby have added support for optional type hints that various tooling can leverage to provide similar legibility features, but they are not required by default.
- **Libraries and Tooling:** Sometimes the choice of language is highly-swayed by particular libraries or tooling you want to use. For example, machine-learning pipelines often use Python because of libraries like NumPy and Pandas. Command-line interfaces (CLIs) like Docker and Terraform are typically written in Go because the tooling makes it easy to build standalone executables for multiple platforms.
- **Developer Experience and Velocity:** How quickly will your engineers be able to implement a new feature, or make major changes to existing ones? Some languages simply require less code to accomplish the same tasks (e.g., Python or Ruby vs Java or Go), and some libraries make implementing particular features very easy. Static typing makes it easier to refactor existing code, and good compilers/linters make it easier to verify that refactors didn't break anything. All of these lead to more efficient and happy engineers, which shortens your time-to-market.
- **Longevity:** Languages, and especially frameworks, come and go. What's hot today may not be hot tomorrow. But some languages (C++, Java, Node, Python, Go) and frameworks have stood the test of time and are used pervasively. They are already very optimized, battle-tested, well-documented, and likely to be supported for decades to come. It's also relatively easy to find engineers who are at least familiar with them, if not experienced in using them.

Which of these qualities you optimize for will depend greatly on your context:

- **Young Startup:** If you are a young startup looking for product-market fit (PMF), you should definitely optimize for developer velocity in the short run. If you end up running out of money because it took too long to build the right product, it won't matter which languages you chose. But if you do succeed, you will want to start optimizing for available labor pool and legibility as you add more engineers, so keep that in mind.
- **Low-Margin Business:** If you're building a system to support a low-margin business, you might want to optimize for qualities that allow you to scale up while keeping operating costs low: async I/O, runtime efficiency, larger labor pool with cheaper engineers, etc.
- **Critical-Path Component:** If you are building a system component that needs to be very efficient and high performance, you should obviously optimize for those qualities over others like developer velocity.
- **Open-Source Project:** If you are starting a new open-source project, you might want to optimize for available labor pool and legibility, so that you can attract and retain contributors.

If you are unsure and just want some general advice, I generally follow these rules:

- **Favor statically-typed languages:** The benefits of static typing far outweighs the minor inconvenience of declaring method argument and return types (most languages automatically work out the types of local variables and constants). Code bases without declared types are fragile and hard to read. The hundreds of engineers you hire after you startup is successful will thank you.
- **Use Async I/O when I/O-Bound:** If your API servers spend most of their time talking to databases and other servers, use a language, framework, and libraries that all have solid support for async I/O. Beware of languages where async I/O was recently bolted-on, as most frameworks and libraries won't support it yet.
- **Avoid trendy but unproven languages/frameworks:** Nothing is worse than discovering the hip language/framework you chose starts to have problems at scale or lacks a critical feature you discover you need. Use something tried and true whenever possible.

When I build I/O-bound API servers, I generally prefer Go, Java, or TypeScript on Node. Rust is very performant and efficient, and has a fabulous tool chain, but it's difficult to learn so it's harder to find other engineers who can be productive in it quickly. I'm warming up to Python now that it has type hints and async I/O, so that can be a good option as well.

Regardless of which languages you choose, this decision will likely be a trapdoor one. Once you write a bunch of code, it becomes very costly and time-consuming to rewrite it in another language (though AI might make some of this more tractable). But there are a few techniques you can use to mitigate the costs of changing languages in the future:

- **Separate System Components:** Your API servers, message consumers, and periodic jobs will naturally be separate executables, so they _can_ be implemented in different languages. Early on you will likely want to use the same language for all of them, just to reduce the cognitive load, but if you decide to change languages in the future, you can rewrite the components incrementally.
- **Serverless Functions:** If it makes sense to build your system with so-called ["serverless" functions](building-blocks.md#serverless-functions), those are also separate components that can be ported to a new language incrementally. API gateways and other functions call them via network requests, so you can change the implementation language over time without having to rewrite the callers as well.
- **Segmented API Servers:** If you have a single API server and decide to rewrite it in a different language, you can do so incrementally by segmenting it into multiple servers behind an [API gateway](building-blocks.md). Taken to an extreme, this approach is known as **microservices** and it can [introduce more problems than it solves](https://www.shopify.com/enterprise/blog/disadvantages-microservices#4), but you don't have to be so extreme. Assuming you are just changing languages, and don't need to scale them differently, the segmented servers can still all run on the same machine, and communicate with each other over an efficient local inter-process channel such as [Unix domain sockets](https://en.wikipedia.org/wiki/Unix_domain_socket).

## Choosing Databases

The next choice you will likely need to make is which database(s) to use.
