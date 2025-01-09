# System Building Blocks

If you were to look at the architectures of the various backend systems that run the web sites you use most often, you would probably notice a lot of similarities. Most will have a set of HTTP servers, with load balancers in front, that respond to requests from clients. Those servers will likely talk to databases, caches, queues, and buckets to manage data. They might also use machine-learning (ML) models to make predictions about that data. Event consumers will respond to new events written to the queues, or changes to those database records. Other jobs will run periodically to archive data, re-train ML models with new data, or perform other routine tasks. What these components _do_ will no doubt vary from system to system, but the _types_ of components used will come from a relatively small set of common building blocks.

This tutorial will give you an overview of these common building blocks, what they are typically used for, what they can and cannot do, and how to use them most effectively. Once you understand these, you can combine them in various ways to build just about any kind of system.

## API Gateways and Load Balancers

Requests to most systems go through 

## HTTP Servers

on-demand serverless functions

webhooks

websockets

## Persistent Databases

SQL vs K/V stores

## Ephemeral Caches

## Data Buckets

## Event Queues

## Event Consumers

## Periodic Jobs

## ML Models

## Consensus Services

## Content Delivery Networks (CDNs)
