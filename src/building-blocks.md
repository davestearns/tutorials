# System Building Blocks

If you were to look at the architectures of the various backend systems that run the web services you use most often, you would probably notice a lot of similarities. Most will have a set of HTTP servers, with load balancers in front, that respond to requests from clients. Those servers will likely talk to databases, caches, and queues to manage data. They might also use machine-learning models to make predictions about that data. Event consumers will respond to new events written to those queues, or changes to those database records. Other jobs will run periodically to check integrity, aggregate, archive, or clean up data. What these components _do_ will no doubt vary from system to system, but the _types_ of components used will come from a relatively small set of common building blocks.

This tutorial will give you an overview of these common building blocks, what they are typically used for, what they can and cannot do, and how to use them most effectively. Once you understand these, you can combine them in various ways to build just about any kind of system.

