# Identifiers

If you've ever taken a relational database course, your instructor probably told you to use database-assigned integers for your primary keys. For example, columns of type `serial` or `bigserial` in PostgreSQL will be automatically assigned a unique integer when inserting a row:

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

For each column like this, the database tracks a last-used integer value. When you insert a row, the database locks and increments this value, assigns the incremented value to the `id` column, and returns that value to your application. Your application can then return that ID in your API response, or use it to insert related records.

When you are just starting out, these database-assigned IDs seem to offer some handy features:

- The database handles all the locking and incrementing to ensure the new ID is unique even when there are multiple inserts happening at the same time.
- Because the IDs are always incremented, rows inserted earlier are guaranteed to have a lower ID than those inserted later, providing a natural creation ordering.
- Because the IDs are integers, they are compact both within the database and in URLs (e.g., `GET /posts/1`).

Unfortunately, as your system grows in scale and complexity, these database-assigned IDs start to become problematic, and most of these benefits start to erode. In this tutorial I'll explain why, and describe an alternative that you should use instead.

## Idempotency

One of things that make building distributed systems hard is the unreliability of the network. When one process makes a request across the network to another process, it sends a message and waits for a response. Typically that response comes back very quickly, but sometimes it takes a long time. This could be due to congestion on the network itself, or an overloaded/crashed target server.

When a client process makes any kind of network request, it specifies a **timeout**, which is the amount of time it is willing to wait for a response. If no response is received within that time duration, an error is returned/thrown to the client process.

This same thing happens when your API server (or any system component) executes an `INSERT` query against your database. The database client library turns that into a network request with a timeout. If the database doesn't respond within that timeout, the database client library returns/throws an error to your application code. This can easily happen when there is a network partition, or when your database becomes overloaded and slow to respond.

This sort of error leaves your application in a bit of quandary: did the database server even receive the `INSERT` request? If so, was it processed successfully? Your application has no way of knowing because it _never received a response_. The database might have inserted the record and tried to send a response, but it might have been too slow because it was overloaded, or a network partition might have kept the response from reaching your application.

To make matters worse, if the new record was actually inserted, your application has no idea what the new database-assigned ID is, because that ID is in the response your application never received! So you can't just query the database to see if it's actually there.

You could retry the `INSERT` query at this point, but when using database-assigned IDs, you run the risk of inserting that record twice. If the original insert went through, and you retry the insert with the same data, the database will just generate a new ID and insert the record again, creating a duplicate.

What we want is for our insert operations to be **idempotent**. An idempotent operation is one that results in the same effect whether it is executed once, or multiple times. Idempotent operations are very handy in distributed systems because they allow us to safely retry network requests that timeout.

One way to make database inserts idempotent is to add an `idempotency_key` column with a unique index. When your application wants to insert a new record, it generates a new unique value for this column, and uses that same value in the original request, as well as any subsequent retries. If the original insert never went through, the retry will succeed because there is no other record in the table with that same value in the `idempotency_key` column. But if the original insert did go through, the retry will fail with a unique constraint violation, which your application can catch. Because the `idempotency_key` field is indexed, you can now efficiently query for the row with the idempotency key you were using, and discover the ID of the previously-inserted record.

But this begs the question: why should we add another unique index to our table when every table already has a unique index on the primary key `id` column? Every additional index slows down inserts because the database must not only append to the base table, but also update each index. Extra indexes also increase the amount of data storage used by the database, which limits per-server data growth.

If we generated our IDs in another way, could we eliminate the need for this extra column and index, while still ensuring that record inserts are idempotent? Before we answer that, let's consider another scenario in which database-assigned IDs can become problematic.

## Partitioning

Database-assigned IDs are handy when you have only one database server, but most large-scale systems end up needing to **partition** (aka shard) data across _multiple_ database servers. This can happen for a few reasons.

The first and most obvious is that your system will eventually accumulate more data than will reasonably fit on a single database server. Databases must write data to persistent storage and that storage has a fixed size, so there is a limit to what a single server can hold. As you system gains more users, and is used by them more often, the amount of data will start to increase dramatically. If your system goes 'viral' you will reach the limits of a single database server surprisingly quickly.

Even if your single server has enormous data storage capacity, it still has only one database engine processing all the queries. As your system gets more and more requests from clients, your API servers will send more and more queries to the database. Eventually that load will saturate that database's CPU and memory, causing a sharp increase in query latency, which will naturally make your API slower as well. So you may end up having to partition your data for compute reasons even before you run out of data storage space.

Lastly, if your system is used in a country that has [data locality regulations](https://en.wikipedia.org/wiki/Data_localization), you might be required to keep the data created by its residents on a database server physically located within that country's jurisdiction. Some of these regulations apply only to narrow types of data (e.g., financial) but others apply very broadly to "all personal data." So you may end up needing to partition your data by country simply to comply with these data locality regulations, even if you have plenty of space in the database located in your home country.

Regardless of what causes you to partition your data, once you do, these database-assigned IDs become a bit problematic. Remember that each database server is tracking a last-used value for each ID column, but they are doing so _independently_. There is no central coordination between the various servers. So what keeps the servers from assigning the _same_ ID to _different_ records stored in different servers?

If you want to keep using database-assigned IDs, the typical solution is to partition the ID space as well. For example, if you decide that each shard can handle about a trillion records, you can manually set the starting and max ID values on each shard to ensure the IDs won't overlap. This is how you'd do it in PostgreSQL:

```sql
-- SERVER 1
create sequence posts_id_seq_1
	minvalue 1
	maxvalue 1000000000000;

create table posts (
	id bigint primary key default nextval('posts_id_seq_1')
);


-- SERVER 2
create sequence posts_id_seq_2
	minvalue 1000000000001
	maxvalue 2000000000000;

create table posts (
	id bigint primary key default nextval('posts_id_seq_2')
);
```

_**Note:** You can actually use the same sequence name on both servers, as they are totally separate servers with separate namespaces, but I added the index to this example just to keep things clear._

With these changes, the first record inserted into server 1 will get the ID `1`, but the first record inserted into server 2 will get the ID `1000000000001`. IDs will keep incrementing within each server's respective ranges, so they will never overlap, but the number of records you can store per-server is now capped at a trillion. You can of course make this cap larger from the start, but you can't change it once you start inserting records.

So we successfully avoided duplicate IDs, but we also lost one of the important benefits of database-assigned IDs noted above: **natural creation ordering.** 

For example, if rows are evenly spread between servers 1 and 2 in the example above, the first row inserted will get ID `1`, the second will get ID `1000000000001`, and the third will get ID `2`. If you sort the records by ID, they will no longer be in creation order.

This may or may not be important, depending on your system's specific needs and goals. And you could add a `created_at` timestamp column to your table and sort by that instead when combining the results from multiple servers. But at this point we should step back and ask, "is there a better alternative?" Could we generate our IDs in another way that ensures uniqueness, doesn't require partitioning the ID space and setting arbitrary limits, but still retains natural creation ordering?

## Application-Assigned IDs

Many large-scale distributed systems use application-assigned IDs instead of database-assigned ones, for the reasons outlined above. Application-assigned IDs provide a natural idempotency, as the same value can be used in the original insert request, as well as all subsequent retries. And application-assigned IDs can be designed to maintain a natural creation ordering regardless of the number of database partitions you have.

There are various formats and algorithms out there, but the common recipe is as follows:

- Start with the number of seconds or milliseconds since a well-known timestamp (known as the 'epoch'). This provides a natural creation ordering. The major cloud providers now offer [hyper-accurate clocks and time-sync services](https://aws.amazon.com/blogs/compute/its-about-time-microsecond-accurate-clocks-on-amazon-ec2-instances/) that keep all servers generating these timestamps within a few microseconds of each other.
- Append a value that will keep multiple IDs generated within the same time duration unique. This could be a sufficiently long random value, or a pre-configured machine ID plus a machine-specific step counter, or a shorter random value plus a machine-specific counter.
- Encode the combined values in a URL-friendly string format, like base 16 (0-9, A-F), base 36 (0-9, a-z) or base 62 (0-9, a-z, A-Z).
- Optionally add a prefix indicating what type of entity the ID identifies. For example, `post_` for Posts, or `pay_` for Payments. This allows humans and machines to distinguish the entity type given just the ID, and quickly load the database record from the appropriate table.

There are a few commonly-used implementations of this recipe that should be available via libraries in most programming languages:

- **[UUIDv7](https://en.wikipedia.org/wiki/Universally_unique_identifier#Version_7_(timestamp_and_random)):** A newer version of the [Universal Unique Identifier standard](https://en.wikipedia.org/wiki/Universally_unique_identifier), which uses a 48-bit timestamp with millisecond granularity, 74 randomly assigned bits, and a few hard-coded bits required by the standard to indicate format and versioning. Overall the ID is 128 bits long, which is typically encoded into a 36 character hexadecimal string (e.g., `01976b10-a45c-7786-988e-e261ef5d015b`). This is rather long to include in URLs, but you can implement a base36 encoding instead to make it shorter (see below).
- **[Snowflake IDs](https://en.wikipedia.org/wiki/Snowflake_ID):** First developed at Twitter (before it was X), but now used by several social media platforms. It uses a 41-bit timestamp with millisecond granularity, plus a 10-bit pre-configured machine ID, plus a 12-bit machine-specific sequence number. The machine ID requires some central coordination (something has to tell each new API server what it's unique ID is), but it also reduces the number of bits needed to ensure uniqueness of IDs generated within the same millisecond. The machine ID plus the 12-bit sequence number allows you to generate 2^12 = 4,096 IDs per-machine per-millisecond. Because they use less bits, the encoded form is also shorter (e.g., `6820698575169822721`).
- **[MongoDB Object IDs](https://www.mongodb.com/docs/manual/reference/method/ObjectId/):** Available in every MongoDB/BSON library, these IDs use a 32 bit timestamp with second granularity, plus a 40 bit random value, plus a 24 bit machine-specific counter. The overall ID in binary form is 96 bits, and when encoded in hexadecimal the resulting string is 24 characters (e.g., `507f191e810c19729de860ea`).

## Separate ID Types

Regardless of which implementation you use, it's a good idea to declare separate types for each of your IDs. For example, an `AccountID` should be a different type from a `SessionID`. That way you can type parameters that expect `AccountID` values appropriately, and your tooling will flag any code that tries to pass the incorrect type.

In Python you can support this using a base class like so:

```python
import string
from typing import Final, Type
import uuid_utils as uuid


class BaseID(str):
    """
    Abstract base class for all prefixed ID types.

    To define a new ID type, create a class that inherits from
    `BaseID`, and set its `PREFIX` class variable to a string
    value that is unique across all BaseID subclasses.

    Example:
        >>> class TestID(BaseID):
        ...     PREFIX = "test"

    If the PREFIX value is not unique across all subclasses
    of BaseID, a ValueError will be raised when the class is
    created.

    To generate a new ID, just create a new instance of your
    derived class with no arguments:

    Example:
        >>> id = TestID()

    The value of the new `id` will have the form
    `"{PREFIX}_{uuid7-in-base36}"`. UUIDv7 values start with
    a timestamp so they have a natural creation ordering. The
    UUID is encoded in base36 instead of hex (base16) to keep
    it shorter.

    The `id` will be typed as a `TestID`, but since it inherits
    from `BaseID` and that inherits from `str`, you can treat
    `id` as a string. Database libraries and other encoders
    will also see it as a string, so it should work seamlessly.

    To rehydrate a string ID back into a `TestID`, pass it
    to the constructor as an argument:

    Example:
        >>> rehydrated_id = TestID(encoded_id)

    A `ValueError` will be raised if `encoded_id` doesn't have
    the right prefix.

    If you have a string ID but aren't sure what type it is,
    use `BaseID.parse()` to parse it into the appropriate type.

    Example:
        >>> parsed_id = BaseID.parse(encoded_id)

    You can then test the `type(parsed_id)` to determine
    which type it is.

    author: Dave Stearns <https://github.com/davestearns>
    """

    PREFIX_SEPARATOR: Final = "_"
    ALPHABET: Final = string.digits + string.ascii_lowercase
    ALPHABET_LEN: Final = len(ALPHABET)

    PREFIX: str
    """
    Each derived class must set PREFIX to a unique string.
    """

    prefix_to_class_map: dict[str, Type["BaseID"]] = {}

    def __new__(cls, encoded_id: str | None = None):
        if encoded_id is None:
            # Generate a new UUID
            id_int = uuid.uuid7().int

            # Base36 encode it
            encoded_chars = []
            while id_int > 0:
                id_int, remainder = divmod(id_int, cls.ALPHABET_LEN)
                encoded_chars.append(cls.ALPHABET[remainder])
            encoded = "".join(reversed(encoded_chars))

            # Build the full prefixed ID and initialize str with it
            prefixed_id = f"{cls.PREFIX}{cls.PREFIX_SEPARATOR}{encoded}"
            return super().__new__(cls, prefixed_id)
        else:
            # Validate encoded_id
            expected_prefix = cls.PREFIX + cls.PREFIX_SEPARATOR
            if not encoded_id.startswith(expected_prefix):
                raise ValueError(
                    f"Encoded ID {encoded_id} does not have expected prefix {cls.PREFIX}"
                )
            return super().__new__(cls, encoded_id)

    def __repr__(self) -> str:
        """
        Returns the detailed representation, which include the specific
        ID class name wrapped around the string ID value.
        """
        return f"{self.__class__.__name__}('{self.__str__()}')"

    def __init_subclass__(cls):
        """
        Called when new subclasses are initialized. This is where we ensure
        that the PREFIX value on a new subclass is unique across the system.
        """
        if not hasattr(cls, "PREFIX"):
            raise AttributeError(
                "ID classes must define a class property named"
                "`PREFIX` set to a unique prefix string."
            )
        if cls.PREFIX in cls.prefix_to_class_map:
            raise ValueError(
                f"The ID prefix '{cls.PREFIX}' is used on both"
                f" {cls.prefix_to_class_map[cls.PREFIX]} and {cls}."
                " ID prefixes must be unique across the set of all ID classes."
            )
        cls.prefix_to_class_map[cls.PREFIX] = cls
        return super().__init_subclass__()

    @classmethod
    def parse(cls, encoded_id: str) -> "BaseID":
        """
        Parses an string ID of an unknown type into the appropriate
        class ID instance. If the prefix does not match any of the
        registered ones, this raises `ValueError`.
        """
        for prefix, cls in cls.prefix_to_class_map.items():
            if encoded_id.startswith(prefix):
                return cls(encoded_id)

        raise ValueError(
            f"The prefix of ID '{encoded_id}' does not match a known ID prefix."
        )
```

_[Full source code and tests](https://github.com/davestearns/ids)_

This `BaseID` class is generic and can be used in any project. You can even package it into a reusable library if you wish. It uses a UUIDv7 for the unique ID portion, but encodes it to characters using base36 instead of base16 to keep the string form shorter. The base36 alphabet is just the characters 0-9 and a-z, so the IDs remain case-insensitive and URL-safe.

To define your specific ID type, create classes that inherit from `BaseID` and set the class variable `PREFIX` to a unique string. The base class ensures that these prefix strings remain unique across all sub-classes.

```python
class AccountID(BaseID):
    PREFIX = "acct"


class SessionID(BaseID):
    PREFIX = "ses"
```

Now you can create these various strongly-typed IDs, turn them into strings, and parse them back into concrete types:

```python
def func_wants_account_id(account_id: AccountID) -> None:
    print(repr(account_id))

# Generate a new AccountID
id = AccountID()

# Pass it to functions expecting an AccountID.
# Trying to pass a different type will trigger
# a static type checking error.
func_wants_account_id(id)
# func_wants_account_id(SessionID()) -- type error!

# When you get an id string from a client, you can
# re-hydrate it back into an instance of ID class.
# (will raise ValueError if wrong prefix)
id_from_client: str = str(id)
rehydrated_id = AccountID(id)

assert type(rehydrated_id) is AccountID
assert rehydrated_id == id
```
