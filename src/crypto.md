# Basic Cryptographic Algorithms

If you want to build software services for the Internet, you need to know some basic cryptographic algorithms. The Internet is a dangerous place, and you should assume that anything you expose to the Internet will get attacked, sooner than later. There are various ways we can design our services to withstand these attacks, and most of them build upon a relatively small set of basic cryptographic algorithms. If you understand these algorithms, you'll also be set up to understand how things like HTTPS/TLS or authenticated sessions or blockchains work.

Thankfully, you don't need to understand the math behind these algorithms (few people do). You also don't need to implement these algorithms yourself--instead you should always use the canonical library for your chosen programming language. But you do need to have a basic conceptual understanding of what these algorithms can and cannot do, how to use them, and how to combine them to achieve various security guarantees.

In this tutorial I'll explain the basics of cryptographic hashing, symmetric and asymmetric (aka public key) encryption, message authentication codes, digital signatures, and certificates. Subsequent tutorials will refer to and build upon these concepts, so take the time to read carefully and fully understand them.

## Cryptographic Hashing

The first family of algorithms to understand are cryptographic hashing functions. These are one-way functions that turn arbitrarily-sized input data into a relatively small, fixed-sized output value, known as a _hash_. For example, the SHA-256 hashing algorithm will turn megabytes of input data into a 256-bit hash value (hence the '256' part of its name).

What makes these algorithms incredibly interesting and useful are the guarantees they make about that output hash value:

* **Deterministic:** Given the same input data, the algorithm will always produce the same output hash.
* **Collision-Resistant:** The probability that two different inputs will generate the same output hash (known as a 'collision') is extremely low, and decreases exponentially with the size of the output hash. With a 256 or 512-bit output, this probability becomes so low that we can [effectively ignore it](https://stackoverflow.com/a/4014407) in most circumstances.
* **Irreversible:** Since the function is one-way, you can't directly calculate the input data from the output hash. You _could_ try hashing every possible input value until you find a match, but that quickly becomes intractable as the input value space grows.

These guarantees are why people often refer to cryptographic hashes as "fingerprints" of their input data. Our fingerprints are relatively small compared to our entire bodies, but they remain unique (enough) to identify us. Similarly, a cryptographic hashing algorithm can reduce gigabytes of data to a relatively short fingerprint that is both deterministic and collision-resistant.

Cryptographic hashes are very useful in a few different ways:

* They can act like short identifiers for potentially huge content. For example, if you have a catalog of songs, along with their hashes, you can quickly determine if a new song uploaded to your catalog is the same as one you already have--you only need to compare the short hashes, not the large song files themselves. Decentralized source code control systems like `git` also use hashes to determine if you already have a commit fetched from a remote branch. 
* They can be used to detect data tampering. For example, if you want to verify that a document or photo hasn't changed since the last time you saw it, you can hash the current version and compare it with the hash you calculated previously and securely stored. Blockchains like bitcoin also use cryptographic hashes to ensure the integrity of the ledger, even though multiple untrusted computers can propose new blocks.
* They can be stored as irreversible yet verifiable tokens of sensitive data you should never store in plain text. For example, we always store hashes of user passwords, never the passwords themselves. During sign-in, we can still verify the provided password by hashing it and comparing that to our stored hash, but an attacker can't directly calculate the original password from the stored hash (more on password hashing in a future tutorial).





## Symmetric Encryption



## Message Authentication Codes (MACs)


## Asymmetric Encryption


## Digital Signatures


## Certificates

