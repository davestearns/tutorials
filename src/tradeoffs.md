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


Unfortunately, there is no one right answer to any of these questions. If there was, everyone would just make the same decision and none of this would require expertise! 

If there is one thing I've learned over the years it is this:

> _Every decision in software engineering is a trade-off that optimizes for some things at the cost of other things. And while some of these decisions can be easily reversed in the future, others are trapdoors._

This implies two questions to answer when making these kinds of decisions:

- **What do I want to optimize for?** What will actually make a difference toward achieving my goals, and what doesn't really matter right now? For example, if you're a new startup searching for product-market fit, you should probably optimize for development velocity over things like operating costs or runtime execution speed. But if you are trying to differentiate yourself in a crowded market, operating costs or runtime speed might be the very thing that actually helps you stand out.
- **How easily could this decision be reversed in the future if necessary?** What you need to optimize for may change in the future, but some early decisions may become intractable to change after a short time. These so-called 'trapdoor' decisions can become a millstone around your neck if you make the wrong bet, so you must consider them carefully. But in other cases where your decision can be easily reversed when needs change, don't sweat it: just make the appropriate trade-off given your current needs and move on.

Let's put these principles into practice by analyzing a few of the most common early trade-offs system builders have to make, many of which can become trapdoors. For each we will consider what qualities you could optimize for, and which would be most important given particular contexts. For those that could be trapdoors, I'll also discuss a few techniques you can use to keep their effects constrained as much as possible.

## Choosing Programming Languages and Frameworks

One of the earliest choices you have to make is which programming languages and frameworks you will use to build your system. There is of course no one right answer for every context, and it ultimately comes down to what you want to optimize for:

- **Personal Familiarity:** Founding engineers will typically gravitate towards languages and frameworks they already know well, as this helps them make progress quickly. This is fine, but you should also consider...
- **Available Labor Pool:** If your system becomes successful, you will likely need to hire more engineers, so you should also consider how easy it will be to find, hire, and onboard engineers who can be productive in your chosen languages/frameworks. For example, the number of experienced Java or Python engineers in the world is far larger than the number of experienced Rust or Elixir engineers. It will be easier to find and cheaper to hire the former than the latter.
- **Runtime Execution Speed:** A fully-compiled language like Rust, Go, or C++ will startup and run faster than JIT-compiled languages like Java and interpreted languages like Python or JavaScript. Languages with explicit memory control, like Rust and C++, also have more consistent latency than garbage-collected languages like Go and Java. But this only really matters if your system is more compute-bound than I/O bound---that is, it spends more time executing the code you wrote than it does waiting for databases, network requests, and other I/O to complete. Most data-centric systems are actually I/O-bound, so the latency benefits of a fully-compiled language may or may not matter.
- **Runtime Efficiency:** Languages like Rust and C++ require less CPU and RAM than languages like Python or Ruby to accomplish the same tasks. This allows you to scale up with less resources, resulting in lower operating costs.
- **Async and Concurrent I/O Support:** The thing I/O-bound systems do benefit from greatly, however, is support for _asynchronous_ and _concurrent_ I/O. With synchronous I/O, the CPU thread processing a given API server request is blocked while it waits for database queries and other network I/O to complete. With async I/O, the CPU can continue to make progress on other requests while the operating system waits for the database query response. This greatly improves the number of concurrent requests a given server can handle, which allows you to scale up for less cost. You can also execute multiple independent I/O operations concurrently, which may improve your latency. Languages like Go and Node do all of this natively. Java, Kotlin, and Python have added support for it in the language runtime, but older libraries and frameworks may still use synchronous I/O.
- **Legibility:** Languages that use static typing (or type hints) are typically more legible than those that use dynamic typing. This means that it's easier for new engineers to read and understand the code. IDEs can also leverage the typing to provide features like statement completion, jump-to-source, and informational popups. Rust, Go, Java, Kotlin, and TypeScript are all statically-typed, while plain JavaScript is dynamically-typed. Languages like Python and Ruby have added support for optional type hints that various tooling can leverage to provide similar legibility features.
- **Libraries and Tooling:** Sometimes the choice of language is highly-swayed by particular libraries or tooling you want to use. For example, machine-learning pipelines often use Python because of libraries like NumPy, Pandas, and PyTorch. Command-line interfaces (CLIs) like Docker and Terraform are typically written in Go because the tooling makes it easy to build standalone executables for multiple platforms.
- **Developer Experience and Velocity:** How quickly will your engineers be able to implement a new feature, or make major changes to existing ones? Some languages simply require less code to accomplish the same tasks (e.g., Python or Ruby vs Java or Go), and some libraries make implementing particular features very easy. Static typing makes it easier to refactor existing code, and good compilers/linters make it easier to verify that refactors didn't break anything. All of these lead to more efficient and happy engineers, which shortens your time-to-market.

Which of these qualities you optimize for will depend greatly on your context:

- **Young Startup:** If you are a young startup looking for product-market fit (PMF), you should definitely optimize for developer velocity in the short run. If you end up running out of money because it took too long to build the right product, or onboard enough engineers, it won't matter which languages you chose.
- **Low-Margin Business:** If you're building a system to support a low-margin business, you might want to optimize for qualities that allow you to scale up while keeping operating costs low: async I/O, runtime efficiency, larger labor pool with cheaper engineers, etc.
- **Critical-Path Component:** If you are building a system component that needs to be very efficient and high performance (e.g., data storage service), you should obviously optimize for those qualities over others like developer velocity.
- **Open-Source Project:** If you are starting a new open-source project, you might want to optimize for available labor pool and legibility instead, so that you can attract and retain contributors.

Choosing a programming language is likely going to be your first big trapdoor decision. Once you write a bunch of code, it becomes very costly and time-consuming to rewrite it in another language (though AI might make some of this more tractable). But there are a few techniques you can use to constrain the effects of your choice and make it easier to change languages over time:

- **Service-Oriented Architecture:** Instead of putting all your logic into one large monolith server, you can break it up into multiple servers that talk to each other, either across process boundaries or the network. The most extreme version of this is known as **microservices**, where your system is comprised of hundreds or thousands of small servers spread out across a network, but this approach typically leads to a whole different set of problems that require sophisticated infrastructure that is beyond all but the largest of organizations. But you can avoid most of these simply by running multiple servers on the same machine, and communicating across process boundaries using something like named pipes.
- 



## Choosing Runtime Infrastructure

## Choosing Databases