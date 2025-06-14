# Identifiers

If you've ever taken a relational database course, your instructor probably told you to use database-assigned integers for record identifiers (aka primary keys). For example, in PostgreSQL the database will automatically assign a unique integer to any column of type `serial` or `bigserial`:

```sql
-- PostgreSQL
create table posts (
	id bigserial primary key,
	-- other columns...
);
```

In MySQL you get a similar behavior when adding the `AUTO_INCREMENT` modifier to any numeric column:

```sql
-- MySQL
create table posts (
	id bigint unsigned auto_increment primary key,
	-- other columns...
);
```

In theory, this seems like a great feature--the database does all the heavy lifting, freeing the application engineer to focus on other things. But in practice, these database-assigned IDs become problematic as your system increases in scale and complexity.

In this tutorial I'll explain why these database-assigned IDs can become problematic at scale, and describe an alternative that you should use instead.

## Partitioning

The first and most obvious problem occurs when your system accumulates more data than can fit in a single database server. This can happen much quicker than you might expect, especially if your product goes viral. When you approach this limit, you must start to **partition** (aka 'shard') your data across multiple database servers.

For example, if you're building a social media system that just accumulates posts made by your customers over time, you'll eventually need to spread those across multiple database servers. How you choose which posts go to which database servers can be tricky: it's a tradeoff between spreading the load evenly across all the available servers, and keeping data you commonly need to access at the same time in the same place. For example, you could spread posts evenly across all of your shards in a round-robin style, but then fetching all recent posts from a given user would require querying all servers. Sometimes it is better to route the posts for a given user to just a subset of the available shards, as this constrains the number of servers you have to query later.

But when you partition your data across multiple servers, each one is now independently tracking and assigning ID values to new records. There are `posts` tables on each server, but the counters being used by those servers for the IDs are not coordinated. If you don't take this into account, different servers will assign the _same ID_ to _different records_, which will cause a lot of problems.

The typical solution is to segment your ID space: server 1 gets IDs from 0 up to _N_, server 2 gets IDs from _N_ up to _2N_, server 3 gets IDs _2N_ up to _3N_, etc. The value of _N_ is determined by the number of records each server could reasonably store and manage while maintaining adequate performance (a good load test will help you determine this). 

Most relational databases have features that help you partition your ID space, so you _can_ make database-assigned IDs work with partitions, but only if you are very, very careful. Essentially you specify a starting and maximum integer on each server that ensures they do not overlap.

On MySQL it would look something like this if you determined _N_ to be 1 billion:

```sql
-- MySQL server 1
create table posts (
	id bigint unsigned auto_increment primary key,
	-- other columns...,
	constraint max_account_id check (id < 1000000000)
);

-- MySQL server 2
create table posts (
	id bigint unsigned auto_increment = 1000000000 primary key,
	-- other columns...
	constraint max_account_id check (id < 2000000000)
);

-- MySQL server 3
create table posts (
	id bigint unsigned auto_increment = 2000000000 primary key,
	-- other columns...
	constraint max_account_id check (id < 3000000000)
);
```

This ensures each server has a non-overlapping ID space, but you now need to watch server 1 very carefully. Before it gets close to being full, you must add more partitions and adjust your application to use them so that you don't end up running out of IDs.

By default, auto-incrementing IDs do not wrap around once they hit their upper limit, for obvious reasons. Once you've hit the maximum ID value on a shard, all new inserts to that table will be rejected by the database. You might be thinking, "oh, I'll just archive the old data and delete it to free up those IDs," but this won't help. The last assigned ID value is tracked separately and doesn't get adjusted as you delete rows from the table. So is it imperative that you monitor how close each server is getting to its maximum ID, and route new records to newer servers.

As your system expands internationally, you may need to add even more partitions in other geographic regions, even if you're not running out of storage space in your home region. Several countries now require [particular kinds of data about their citizens to physically remain within the country](https://en.wikipedia.org/wiki/Data_localization). In some countries the scope is very broadly defined as "all personal data," which affects nearly any kind of system you might build.

## Natural Creation Ordering

Although segmenting the ID space can work, you do lose an important quality of database-assigned IDs: natural creation ordering.

When you have a single server, with a single ID range, the ID values are monotonically increasing. Each new record gets an ID that is at least one higher than the previous record. Sometimes you can get holes (e.g., when you rollback an insert), but the database guarantees that a record with a higher ID was inserted _after_ a record with a lower ID. When querying the data, if you sort by ID (which is the default) the results will naturally be in creation order.

When you subdivide your ID range like we did in the previous section, you lose this guarantee. The first record written to server 2 in the example above will have the ID `1000000000` but the second record written to server 1 will have the ID `1`. You can no longer sort records by their ID to list them in the order in which they were created.

A common mitigation for this problem is to add an `inserted_at` timestamp column to your table, with an index, and let your database server assign it based on its system clock. This works, but it requires sorting by a column other than the primary key, which is typically slower in relational databases, especially as the number of query results increases. The extra index also slows down new inserts, as the database must update the index as well as insert the record into the base table.

This technique also used to suffer from clock skew, where the system clocks on your database servers were not totally aligned. If one server's clock drifted behind the others, the timestamps would make it appear as if records inserted to that server came before all the others, even though they were really inserted around the same time. But this has become less and less of an issue in the large cloud providers, thanks to [hyper-accurate clocks and time sync services](https://aws.amazon.com/blogs/compute/its-about-time-microsecond-accurate-clocks-on-amazon-ec2-instances/).

## Enumeration Attacks

The monotonically-increasing nature of database-assigned IDs also introduces a small security risk--if I know that my record has an ID of `123`, I can also reasonably assume that there are other records, potentially owned by other people, with the IDs `122` and `124`.

These IDs often show up in various ways in our APIs or applications. For example, an API server might support a request like `GET /payments/{id}` to get the details of a particular payment given its ID. If I know one ID that I created, I can simply try another request incrementing that ID to see if I can read someone else's record.

In APIs, this is typically mitigated by proper authentication and authorization. Authentication allows the API server to know _who_ is making the request, so it can also determine if the requestor is _authorized_ to read or manipulate the identified resource. Although you'd expect all APIs to do this sort of authentication and authorization, you'd be surprised how often it gets overlooked.

But even if these mitigations are in place, sometimes these identifiers can be entered directly into application user interfaces. For example, when you purchase something online with a payment card, you must enter your card number, which is an identifier. The number is 16 digits long, but the first 6 or 8 of those identify the card network and issuing bank, and the last one is a check digit that is calculated deterministically from the previous 15 using a well-known algorithm. That leaves 7 to 9 digits for the individual account number, but many issuers will use the first few of those to group accounts by funding model (credit vs debit) and product tier (basic vs rewards).

If an attacker knows a valid card number, they can likely guess another valid card number from the same issuer simply by iteratively incrementing the account number portion and recalculating the appropriate check digit. If a valid card number is the only thing needed to make a purchase, it would be relatively easy for an attacker to commit fraud. This is why most checkout forms require you to enter other information associated with the card, but can't be known from the number itself: the 3-digit card verification value, the expiration date, and the billing zip code.

Unfortunately, it's all too easy for a new system to overlook these kinds of risks, and fail to add mitigations against them. But if your IDs are not sequential, and are instead longer and more random, it becomes much harder for an attacker to guess another valid ID from an existing one.

## Idempotency

A more subtle problem with database-assigned IDs is related to idempotency, or the ability to safely retry an `INSERT` query that timed out.

When your API server executes a database query, the library supplied by the database vendor turns that into a network request to the database server. It also typically enforces a timeout on that network request--a maximum amount of time it is willing to wait for a response. When the database is overloaded, or when there is a network interruption, the library may not get a response within that timeout duration, and it will generate an exception. Your API server can catch that exception, but how should you handle it? Is it safe to retry the insert?

The trouble is, your API server never got a response, so it has no idea if the original insert request even made it to the database server, or if it was processed successfully. If the original request did go through, and you just blindly retry the request, you will end up with duplicate database records, each with a different database-assigned ID.

The common way to mitigate this risk is to add a column to your table with a `unique` constraint on it. When the application inserts a new record, it generates a new unique value for this column (e.g., a UUID), and it sends this value in both the original insert request, as well as any retries. If the original request never made it to the database, the retry will go through since that value hasn't been inserted in that column yet. But if the original request did get processed successfully and your application just didn't get the response, the retry will be appropriately blocked for violating the unique constraint. This ensures you don't end up with duplicate records.

This works, but it requires a second unique constraint on your table, when every table already has one on the primary key column. Each unique constraint adds an index to the table, and each additional index slows down inserts, as the database must not only persist the record in the base table, but also update each index.

But if your _application_ assigns new IDs instead of the database, you can leverage the existing unique constraint on the primary key column to get this idempotent behavior for free. Effectively, the application-assigned ID becomes your idempotency key, allowing you to safely retry insert requests that timeout.

But this begs the question: how can we create IDs within our application code that are guaranteed to be unique across all the API servers and all database partitions, maintain a natural ordering, avoid enumeration attacks, and automatically act as idempotency keys?

## Application-Assigned IDs

Many large-scale distributed systems use application-assigned IDs instead of database-assigned ones, for the reasons outlined above. There are various formats and algorithms out there, but the common recipe is as follows:

- Start with the number of millisecond or nanoseconds since a well-known timestamp (known as the 'epoch'). This provides a natural creation ordering.
- Add a value that will keep multiple IDs generated within the same milli/nanosecond unique. This could be a sufficiently long random value, or a pre-configured machine ID plus a machine-specific step counter, or a shorter random value plus a machine-specific counter.
- Encode the combined values in a URL-friendly string format, like base 36 (a-z, 0-9) or 62 (a-z, A-Z, 0-9).
- Optionally add a prefix indicating what type of entity the ID identifies. For example, `acct_` for Accounts, or `pay_` for Payments. This allows humans and machines to distinguish the entity type given just the ID, and quickly load the associated database record.

There are a few commonly-used implementations of this recipe that should be available via a library in most programming languages:

- **[UUIDv7](https://www.uuidgenerator.net/version7):** A newer version of the [Universal Unique Identifier standard](https://en.wikipedia.org/wiki/Universally_unique_identifier), which uses a 48-bit timestamp with millisecond granularity, 74 randomly assigned bits, and a few hard-coded bits required by the standard to indicate format and versioning. Overall the ID is 128 bits long, which is typically encoded into a 36 character hexadecimal string (e.g., `01976b10-a45c-7786-988e-e261ef5d015b`). This is rather long to include in URLs, but the 74 random bits allows you to generate millions of IDs within the same millisecond, without central coordination, with an extremely low collision probability.
- **[Snowflake IDs](https://en.wikipedia.org/wiki/Snowflake_ID):** First developed at Twitter (before it was X), but now used by several social media platforms. It uses a 41-bit timestamp with millisecond granularity, plus a 10-bit pre-configured machine ID, plus a 12-bit machine-specific sequence number. The machine ID requires some central coordination (something has to tell each new API server what it's unique ID is), but it also reduces the number of bits needed to ensure uniqueness of IDs generated within the same millisecond. The machine ID plus the 12-bit sequence number allows you to generate 2^12 = 4,096 IDs per-machine, per-millisecond. Because they use less bits, the encoded form is also shorter (e.g., `6820698575169822721`). But because they use counters instead of randomness, it is relatively easy to predict other valid IDs from a known valid one, especially in high-velocity systems producing more than one ID per millisecond.
- **[MongoDB Object IDs](https://www.mongodb.com/docs/manual/reference/method/ObjectId/):** Available in every MongoDB client library, these IDs use a 32 bit timestamp with second granularity, plus a 40 bit random value, plus a 24 bit machine-specific counter. The overall ID in binary form is 96 bits, and when encoded in hexadecimal the resulting string is only 24 characters.

If you don't like any of those options, it's actually fairly easy to implement this recipe yourself, especially if your programming language can generate cryptographically random bits and handle large integers. For example, an implementation in Python would look something like this:

```python
import time
import secrets
import string

DEFAULT_ALPHABET = string.digits + string.ascii_lowercase


def new_id(num_random_bits=64, alphabet=DEFAULT_ALPHABET):
    """
    Returns a unique ID consisting of nanoseconds since Unix epoch (Jan 1, 1970 UTC)
    followed by a cryptographically secure random value, encoded to a string using the
    provided alphabet.

    The number of random bits determines how many IDs you can safely generate within
    the same nanosecond. Use a birthday paradox calculator to estimate how many bits
    you will need depending on the number of IDs you need to generate per-nanosecond.
    """
    ns_since_epoch = time.monotonic_ns()
    random_value = secrets.randbelow(2**num_random_bits)

    binary_id = (ns_since_epoch << num_random_bits) | random_value

    alphabet_len = len(alphabet)
    encoded_id = ""
    while binary_id > 0:
        binary_id, remainder = divmod(binary_id, alphabet_len)
        encoded_id = alphabet[remainder] + encoded_id

    return encoded_id
```

The IDs generated by this look like `1fr8yhohkh793fxw1j4fi40j0` by default. They are relatively short (25 characters), case-insensitive, and safe to embed in URLs. Using 64 random bits allows for generating 100,000 IDs per-nanosecond with an extremely low probability of collision. If you need more or less than that, just increase/decrease the `num_random_bits` parameter. And if you want shorter encodings, you can expand the set of characters in the provided `alphabet`, but be careful about adding characters that can't be embedded safely in URLs.

