# Identifiers

If you've ever taken a relational database course, your instructor probably told you to use database-assigned integers for record identifiers (aka primary keys). For example, in PostgreSQL the database will automatically assign a unique value to any column of type `serial` or `bigserial`:

```sql
-- PostgreSQL
create table accounts (
	id bigserial primary key,
	-- other columns...
);
```

In MySQL you get a similar behavior when adding the `AUTO_INCREMENT` modifier to any numeric column:

```sql
-- MySQL
create table accounts (
	id bigint unsigned auto_increment primary key,
	-- other columns...
);
```

These database engines accomplish this by tracking a last-used (or next) value for each of these columns. As new records are inserted, the database locks and increments this value so that each record gets a unique value, even if multiple inserts are happening at the same time.

In theory, this seems like a great feature--the database does all the heavy lifting, freeing the application engineer to focus on other things. But in practice, these database-assigned IDs become problematic as your system increases in scale and complexity.

In this tutorial I'll explain why these database-assigned IDs can become problematic at scale, and describe an alternative that you should use instead.

## Partitioning

The first and most obvious problem occurs when your system accumulates more data than can fit in a single database server. This can happen much quicker than you might expect, especially if your product goes viral. When you approach this limit, you must start to **partition** (aka 'shard') your data across multiple database servers. 

When you partition your data across multiple servers, each one is now independently tracking and assigning ID values to new records. If you don't take this into account, different servers will assign the _same ID_ to _different records_, which will cause a lot of problems.

Both PostgreSQL and MySQL have features that help you partition your ID space as well, so you _can_ make this work, but only if you are very careful. For example, if you have two servers, you can tell the first server to start ID values at zero, and the second server to start ID values half way through the range of the column's data type. You would also typically add a constraint to the first server to ensure that it stops assigning IDs when it reaches the value you used as the starting point for the second server (and before you get close to that, you'd add more partitions). 

On MySQL with unsigned big integers (which have a range of 0 to 2^64 - 1) it would look something like this:

```sql
-- MySQL Server 1
create table accounts (
	id bigint unsigned auto_increment primary key,
	-- other columns...,
	constraint max_account_id check (id < 9223372036854775807)
);

-- MySQL Server 2
create table accounts (
	id bigint unsigned auto_increment = 9223372036854775807 primary key,
	-- other columns...
);
```

This can work, but it becomes very cumbersome and error-prone as you increase the number of partitions. For example, when you add a third or fourth server, you can't simply reassign the starting value of the second server, as it has already created records starting at `9223372036854775807`. Instead, you must carefully subdivide the ID spaces of server 1 and 2 for the third and fourth servers, effectively interleaving them, like so:

```sql
-- MySQL Server 1
-- Give half of server 1's ID space to new server 1.5
alter constraint max_account_id check (id < 4611686018427387903)

-- MySQL Server 1.5
create table accounts (
	id bigint unsigned auto_increment = 4611686018427387903 primary key,
	-- other columns...
	constraint max_account_id check (id < 9223372036854775807)
);

-- MySQL Server 2
-- Give half of server 2's ID space to new server 2.5
alter table accounts
	add constraint max_account_id check (id < 13835058055282163711);

-- MySQL Server 2.5
create table accounts (
	id bigint unsigned auto_increment = 13835058055282163711 primary key,
	-- other columns...
);
```

As you might have guessed, the ID space available to each partition halves as you add another partition in-between. As you add more and more partitions, you may start running out of IDs before you run out of physical disk space. This is especially true when using PostgreSQL, where `bigserial` is actually a _signed_ data type with only half the range of MySQL's `bigint`.

By default, auto-incrementing IDs do not wrap around once they hit their upper limit, for obvious reasons. So even if you archive old data from these partitions to make room for new data, that doesn't automatically recycle the deleted IDs. Some databases allow you to change this behavior so the IDs will wrap around, but this is obviously quite dangerous--if you don't archive the data in time, or if you can't archive all rows within a particular ID range, you run the risk of assigning duplicate IDs, which will fail to insert due to primary key constraints.

As your system expands internationally, you may need to add even more partitions in other geographic regions, even if you're not running out of storage space in your home region. Several countries now require [particular kinds of data about their citizens to physically remain within the country](https://en.wikipedia.org/wiki/Data_localization). In some countries the scope is very broadly defines as "all personal data," which affects nearly any kind of system you might build. As more countries adopt similar regulations, the number of partitions you must run increases, further reducing the ID space available to each one.

## Natural Ordering

Although segmenting the ID space can work, at least for a while, you do lose an important quality of database-assigned IDs: natural ordering.

When you have a single server, with a single ID range, the ID values are monotonically increasing. Each new record gets an ID that is at least one higher than the previous record. Sometimes you can get holes (e.g., when you rollback an insert), but the database guarantees that a record with a higher ID was inserted _after_ a record with a lower ID.

When you subdivide your ID range like we did in the previous section, you lose this guarantee. You can no longer sort records by their ID to list them in the order in which they were created.

A common mitigation for this problem is to add an `inserted_at` timestamp column to your table, and let your database server assign it based on its system clock. This works, but it requires sorting by a column other than the primary key, which is typically slower in relational databases, especially as the number of query results increases.

This technique also used to suffer from clock skew, where the system clocks on your database servers were not totally aligned. But this has become less and less of an issue in the large cloud providers, thanks to [hyper-accurate clocks and time sync services](https://aws.amazon.com/blogs/compute/its-about-time-microsecond-accurate-clocks-on-amazon-ec2-instances/). It's still theoretically _possible_ that a record with a higher timestamp was actually inserted before a record with a lower timestamp, but in _practice_ this risk is no longer much of a concern in most situations.

## Enumeration Attacks

The monotonically-increasing nature of database-assigned IDs also introduces a small security risk--if I know that my record has an ID of `123`, I can also reasonably assume that there are other records, owned by other people, with the IDs `122` and `124`.

These IDs often show up in various ways in our APIs or applications. For example, an API server might support a request like `GET /payments/{id}` to get the details of a particular payment given its ID. If I know one ID that I created, I can simply try another request incrementing that ID to see if I can read someone else's record.

In APIs, this is typically mitigated by proper authentication and authorization. Authentication allows the API server to know _who_ is making the request, so it can also determine if the requestor is _authorized_ to read or manipulate the identified resource. Although you'd expect all APIs to do this sort of authentication and authorization, you'd be surprised how often it gets overlooked.

But even if these mitigations are in place, sometimes these identifiers can be entered directly into application user interfaces. For example, when you purchase something online with a payment card, you must enter your card number, which is an identifier. The number is 16 digits long, but the first 6 or 8 of those identify the card network and issuing bank, and the last one is a check digit that is calculated deterministically from the previous 15 using a well-known algorithm. That leaves 7 to 9 digits for the individual account number, but many issuers will use the first few of those to group accounts by funding model (credit vs debit) and product tier (basic vs rewards).

If an attacker knows a valid card number, they can likely guess another valid card number from the same issuer simply by iteratively incrementing the account number portion and recalculating the appropriate check digit. If a valid card number is the only thing needed to make a purchase, it would be relatively easy for an attacker to commit fraud. This is why most checkout forms require you to enter other information associated with the card, but can't be known from the number itself: the 3-digit card verification value, the expiration date, and the billing zip code.

Unfortunately, it's all too easy for a new system to overlook these kinds of risks, and fail to add mitigations against them. But if your IDs are not sequential, and are instead longer and more random, it becomes much harder for an attacker to guess another valid ID from an existing one.

## Idempotency

A more subtle problem with database-assigned IDs is related to idempotency, or the ability to safely retry an `INSERT` query that timed out.

When your API server executes a database query, the library supplied by the database vendor turns that into a network request. It also typically enforces a timeout on that network request--a maximum amount of time it is willing to wait for a response. When the database is overloaded, or when there is a network interruption, the library may not get a response within that timeout duration, and it will generate an exception. Your API server can catch that exception, but how should you handle it? Is it safe to retry the insert?

The trouble is, your API server never got a response, so you have no idea if the original insert request made it to the database server, nor if it was processed successfully. If you blindly retry, and you're using database-assigned IDs, you will insert the record again, creating a duplicate, if the original request actually made it through.

The common way to mitigate this risk is to add a column to your table with a `unique` constraint on it. When the application inserts a new record, it generates a new unique value for this column (e.g., a UUID), and it sends this value in both the original insert request, as well as any retries. If the original request never made it to the database, the retry will go through since that value hasn't been inserted in that column yet. But if the original request did get processed successfully and your application just didn't get the response, the retry will be appropriately blocked for violating the unique constraint. This ensures you don't end up with duplicate records.

This works, but it requires a second unique constraint on your table, when every table already has one on the primary key column. Each unique constraint adds an index to the table, and each additional index slows down inserts, as the database must not only persist the record in the base table, but also update each index.

If your application assigns new IDs instead of the database, you can leverage the existing unique constraint on the primary key column to get this idempotent behavior for free. Effectively, the application-assigned ID becomes your idempotency key, allowing you to safely retry insert requests that timeout.

But this begs the question: how can we create IDs within our application code that are guaranteed to be unique across all the database partitions, maintain a natural ordering, avoid enumeration attacks, and automatically act as idempotency keys?

## Application-Assigned IDs

Many large-scale distributed systems use application-assigned IDs instead of database-assigned ones, for the reasons outlined above. There are various formats and algorithms out there, but the common recipe is as follows:

- Start with the number of millisecond or nanoseconds since a well-known timestamp (known as the 'epoch'). This provides a natural creation ordering.
- Add a value that will keep multiple IDs generated within the same milli/nanosecond unique. This could be a sufficiently long random value given how quickly you expect to generate new IDs, or a pre-configured machine ID plus a machine-specific step counter.
- Encode the combined values in a URL-friendly string format, like base 36 (a-z, 0-9) or 62 (a-z, A-Z, 0-9).
- Optionally add a prefix indicating what type of entity the ID identifies. For example, `acct_` for Accounts, or `pay_` for Payments. This allows humans and machines to distinguish the entity type given just the ID, and quickly load the associated database record.

There are two commonly-used implementations of this recipe that should be available via a library in most programming languages:

- **[UUIDv7](https://www.uuidgenerator.net/version7):** A newer version of the [Universal Unique Identifier standard](https://en.wikipedia.org/wiki/Universally_unique_identifier), which uses a 48-bit timestamp with millisecond granularity, 74 randomly assigned bits, and a few hard-coded bits required by the standard to indicate format and versioning. Overall the ID is 128 bits long, which is typically encoded into a 36 character hexadecimal string (e.g., `01976b10-a45c-7786-988e-e261ef5d015b`). This is rather long to include in URLs, but the 74 random bits allows you to generate millions of IDs within the same millisecond, without central coordination, with an extremely low collision probability.
- **[Snowflake IDs](https://en.wikipedia.org/wiki/Snowflake_ID):** First developed at Twitter (before it was X), but now used by several social media platforms. It uses a 41-bit timestamp with millisecond granularity plus a 10-bit pre-configured machine ID plus a 12-bit machine-specific sequence number. The machine ID requires some central coordination, but it also reduces the number of bits needed to ensure uniqueness of IDs generated within the same millisecond, and allows them to use a counter instead of randomness. The machine ID plus the 12-bit sequence number allows you to generate 2^12 = 4,096 IDs per-machine, per-millisecond. Because they use less bits, the encoded form is also shorter (e.g., `6820698575169822721`). But because they use counters instead of randomness, it is relatively easy to predict other valid IDs from a known valid one, especially in high-velocity systems producing more than one ID per millisecond.

If you don't like either of those options, it's actually fairly easy to implement this recipe yourself, especially if your programming language can generate cryptographically random bits and handle large integers. For example, an implementation in Python that supports generating 1,000 IDs per-nanosecond with an extremely low probability of collision would look something like this:

```python
def new_id(num_random_bits=64, alphabet=ALPHABET):
    """
    Returns a unique ID consisting of nanoseconds since Unix epoch (Jan 1, 1970 UTC)
    followed by a cryptographically secure random value, encoded to a string using the
    provided alphabet.

    The number of random bits determines how many IDs you can safely generate within
    the same nanosecond. Use a birthday paradox calculator to estimate how many bits
    you will need depending on the number of IDs you need to generate per-nanosecond.
    """
    ns_since_epoch = time.time_ns()
    random_value = secrets.randbelow(2**num_random_bits)

    binary_id = (ns_since_epoch << num_random_bits) | random_value

    alphabet_len = len(alphabet)
    encoded_id = ""
    while binary_id > 0:
        encoded_id = alphabet[binary_id % alphabet_len] + encoded_id
        binary_id //= alphabet_len

    return encoded_id
```

The IDs generated by this look like `1fr8yhohkh793fxw1j4fi40j0` by default. They are relatively short (25 characters), case-insensitive, and safe to embed in URLs. Using 64 random bits allows for generating 100,000 IDs per-nanosecond with an extremely low probability of collision. If you need more than that, just increase the `num_random_bits` parameter. And if you want shorter IDs, you can expand the set of characters in the provided `alphabet`, but be careful about adding characters that can't be embedded safely in URLs.

