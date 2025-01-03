# Basic Cryptographic Algorithms

If you want to build software services for the Internet, you need to know some basic cryptographic algorithms. The Internet is a dangerous place, and you should assume that anything you expose to the Internet will get attacked, sooner than later. There are various ways we can design our services to withstand these attacks, and most of them build upon a relatively small set of basic cryptographic algorithms. If you understand these algorithms, you'll also be set up to understand how things like HTTPS/TLS or authenticated sessions or blockchains work.

Thankfully, you don't need to understand the math behind these algorithms (few people do). You also don't need to implement these algorithms yourself--instead you should always use the canonical library for your chosen programming language. But you do need to have a basic conceptual understanding of what these algorithms can and cannot do, how to use them, and how to combine them to achieve various security guarantees.

In this tutorial I'll explain the basics of cryptographic hashing, symmetric and asymmetric (aka public key) encryption, message authentication codes, digital signatures, and certificates. Subsequent tutorials will refer to and build upon these concepts, so take the time to read carefully and fully understand them.

## Cryptographic Hashing

The first family of algorithms to understand are cryptographic hashes. These are one-way functions that generate a fixed-size fingerprint (known as a _hash_) of arbitrarily-sized data. For example, you can give a cryptographic hashing algorithm like SHA-256 a simple string, a document, a picture, or even an entire movie file, and it can 'hash' that data down to a binary value that is only 256 bits long (hence the '256' part of its name).

What makes these algorithms incredibly useful are their guarantees:

1. Given the same input data, the algorithm will always produce the same output hash.
1. The probability that two different inputs will generate the same output hash (known as a 'collision') is extremely low, and decreases exponentially with the size of the output hash. With a 256 or 512-bit output, this probability becomes so low that we can [effectively ignore it](https://stackoverflow.com/a/4014407) in most circumstances.
1. Since the function is one-way, you can't reverse it by directly calculating the input data from the output hash. This makes sense when you think about it--if you hash an entire movie file down to 256 bits, you obviously can't directly reconstruct that movie file from only those 256 bits.
1. Even if you try to just guess the input by hashing one thing after another until you find a match, it would be intractable to find the correct input value within any reasonable amount of time, provided the input data was sufficiently long and random (more on this later as well).

The first two guarantees are why people often refer to cryptographic hashes as "fingerprints" of the input data. If the same input data always generates the same output hash, and the probability of a collision is so low as to be ignorable, one can use a cryptographic hash as an optimized content identifier. For example, suppose you are a music sharing platform, and you want to detect if a new song upload is the same as one you already have in the catalog. Comparing the entire song file, byte-by-byte, would be slow and computationally intensive. Instead you could compare _hashes_ of the song files, which are much smaller (256 bits for SHA-256). If you store these hashes in an indexed database column, your system could detect a duplicate across a massive catalog within milliseconds.

This "fingerprint" quality is also why cryptographic hashes get used for tamper-detection. If you hash some data, like a document for example, and keep that hash secure, you can verify in the future that the document hasn't been altered by simply re-hashing it and comparing the result to your previously-calculated hash. We will discuss this scenario in more detail in the [digital signatures](#digital-signatures) section.

The other two guarantees are why cryptographic hashes are often used for protecting sensitive data like passwords. Instead of storing passwords in plain text in your database, you should only store a hash of the user's password--you can still verify the password provided during a subsequent sign-in by hashing it and comparing that hash to the one stored in your database. If an attacker gains access to your database, they can't directly reconstruct user passwords if the database only contains hashes of those passwords. 

But notice that caveat in that last guarantee: "provided the input data is sufficiently long and random." Unfortunately, passwords are chosen by our customers, and those customers often make poor choices when it comes to passwords--the values are often too short, too common, and insufficiently random. An attacker with a database of stolen hashes can simply compare them against pre-calculated hashes of common passwords and dictionary words (with common substitutions like `0` for `O` that customers think are really tricky). If they find a match, they then know the password for that account.

This is why there are specific algorithms for hashing values like passwords, such as [argon2](https://en.wikipedia.org/wiki/Argon2). These algorithms add a couple of additional techniques that defend against these brute-force attacks:

* A random value per-password, known as a _salt_, that is mixed with the password when hashing so that the result is different from just hashing the password alone. This makes pre-calculated hashes of common passwords and dictionary words useless, as the salt value is different and unpredictable for each password.
* A configuration parameter that makes the algorithm do more work, slow down, and consume more electricity. This limits the number of hashes an attacker can do per-second, and increases their operating expenses.

Most systems will also enforce minimum password complexity policies: for example, at least 8 characters, mix of casing, some numbers or symbols, etc. These are designed to stop customers from using especially weak and well-known passwords (like "password"), which forces the attacker to iterate through more random values.

Given enough time, it's still theoretically _possible_ for an attacker to eventually guess the correct input password that matches a given stolen hash and salt, but these techniques make that process so slow and expensive that many attackers will simply give up, especially if the value of a compromised account isn't worth the expense. It also gives the system operator time to notice the breach and invalidate the existing passwords.

## Symmetric Encryption



## Message Authentication Codes (MACs)


## Asymmetric Encryption


## Digital Signatures


## Certificates

