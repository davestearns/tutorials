# Introduction

The world doesn't need another "Hello, World!" tutorial.

You know the ones I mean: those articles that promise to teach you in ten minutes how to build a web server in whatever the trendy programming language is this week. You copy and paste some code (typically all in just one file) and, voilà! You have a web server running on your laptop that replies to every request with a simple greeting.

These tutorials can get you started, but they also leave you with a false sense of accomplishment. You've learned practically nothing about building a real system. You're left at the edge of a cliff, wondering how to:

- expand this to do something meaningful?
- design an easy-to-use API?
- properly structure the code so it doesn't turn into a [giant ball of spaghetti](https://en.wikipedia.org/wiki/Spaghetti_code) as it grows and changes?
- protect the system against attack?
- authenticate callers and authorize their requests?
- reliably perform asynchronous and scheduled work?
- notify clients about new things that happen on the server?
- structure your data and handle changes to that structure over time?
- ensure the authenticity and integrity of that data?
- deploy to production and upgrade the system without downtime?
- monitor the system and automatically generate alerts when something goes wrong?
- scale up to meet increased usage, and scale back down during lulls?
- allow the system to be extended by others?

In short, how do I build not just a toy, but a _complete system_? How do I build something as complex and scalable as a social media platform, or an online retail store, or a consumer payment system?

These tutorials, and the courses that use them, will teach you how to build such things. It will definitely take you longer to go through these than one of those "Hello, World!" tutorials, but you'll also learn a lot more. If you're taking one of my courses, you'll also get to put them into practice, which is the best way to really learn the concepts and techniques. By the end, you'll have a conceptual foundation upon which you can build a successful career as a systems engineer.

So let's get started! First up: a tour of the various [building blocks](building-blocks.md) that make up just about any transaction processing system.