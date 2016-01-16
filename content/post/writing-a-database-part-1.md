+++
date = "2016-01-16T00:21:42-06:00"
draft = false
title = "Writing a database in Go: Part 1"

+++

Databases are a dime a dozen these days. So why in the world would I write another one? Well, mainly for fun, but also for specialization. Building a database which specializes in one thing could reap huge benefits such as performance, data model, or even operations. But obviously it depends on what it is you are building...

##### In this series...

1. [Data model and query language](http://eliquious.github.io/post/writing-a-database-part-1/ "Part 1")
2. Lexing, Parsing and the AST
3. Query Execution
4. Database server
5. Database client
6. Benchmarks
7. Performance enhancements

So what are we building? We're going to build another key-value database. However, this one is going to be geared specifically for range scans and is called [PrefixDB][1]. So where should we start? We should start by defining the data model and then the query language.

</br>

### Defining the data model

If we want to do range scans, where do we start? Or maybe a better question is how do we want to query the data? Here are some questions that still need answering:

- What about data types? Are we going to have them?
- Is there a schema for the keys or values?
- What about multidimensional range scans?
- Does the database need to filter the keys or values by attributes?
- How are the keys and values going to be stored?

Since all of those questions will need to be answered, let's start with the basics. For PrefixDB, both the keys and values will be strings for simplicity. What about schemas? Well, that might depend on if we want to include multidimensional keys. Allowing for multidimensional keys would allow for more flexibility but also help distinguish PrefixDB from all the other options. I'd say let's go for it.

So if we're going to do multidimensional keys, what does that mean for a schema? For values, it doesn't really make sense, but at a minimum we probably need to validate the dimensions used in the keys.

What about data storage? For range scans we need something that can do efficient reads and writes. Fortunately, there's the excellent [BoltDB][2] we might can take advantage of as our initial storage engine.

BoltDB is a simple (but awesome!) B+Tree implementation in pure Go. I've used it in several projects and found it to be quite a joy to use. Not to mention the performance is incredible for range scans.

### Defining the query language

Now that we have a somewhat defined data model, let's start looking at the query language. From my experience, defining the query language (both the syntax and semantics) can help ferret out specific details for the data model.

So if we're going to use multidimensional keys, then we need a way to query the keys by multiple attributes as well as define what those attributes are going to be. Let's start by defining a keyspace.

#### CREATE KEYSPACE

Here's the syntax for creating keyspaces. A keyspace, at least for the purposes of PrefixDB, is simply a collection of key-value pairs which belong to a set of attributes.

```
// This will create a `users` keyspace with a single key.
CREATE KEYSPACE users WITH KEY username
```

So, what about multiple attributes for keys? Well, there's a few ways we can do it but I think I've settled on the following.

```
// This will create a `users.settings` keyspace with a two keys.
CREATE KEYSPACE users.settings WITH KEYS username, setting
```

By defining the keyspaces this way, we can not only setup multiple attributes for a particular keyspace, but we can also use the order to declare a hierarchy or priority for the keys. This hierarchy is what will also give us the ability to do efficient range scans. This subtle design may limit the query capability, but will help ensure applications are built to use the databases strengths.

Also, by defining the name of the keyspace `users.settings` it is immediately apparent as to what the values contain along with the attributes of the keys.

#### SELECT

PrefixDB wouldn't be much of a database without being able to read data. Here's how it's done.

```
SELECT FROM users WHERE username = "eliquious";
```

The above query would return the value stored in the `users` keyspace under the key `eliquious`. What about multiple keys?

```
SELECT FROM users.settings WHERE user = "eliquious";
```

The query above is cool and here's why. If you look at how the keyspace is defined above, there are two keys (`username` and `setting`). If we allow users to exclude attributes then we get some very neat semantics. However, they can't just exclude any attribute, they must query them in the order they were defined in the keyspace.

This limitation might seem bizarre for those coming from the traditional RDBMS world. However, we're focusing on efficient range queries and this limitation will give us a significant boost in query performance. This also changes how applications must define the keyspaces and in doing so reap the benefits of using PrefixDB.

Even though this limitation exists, you can sill query for values using all the attributes in a key.

```
SELECT FROM users.settings WHERE user = "eliquious" AND setting = "email";
```

The query above would return the email for the user `eliquious`.

Additionally, the efficient range scans would allow for queries like this:

```
SELECT FROM users.messages WHERE user = "eliquious" AND
    time BETWEEN "2016-12-16" AND "2016-01-16";
```

#### UPSERT

`UPSERT` is not used in many databases, however, it is used to insert the value if it doesn't exist or update it if the key is already there. Here's how it looks.

```
UPSERT "..." INTO users WHERE username = "eliquious";
```

It works similarly if the keyspace has multiple attributes.

```
UPSERT "Bugs Bunny" INTO users.setting WHERE username = "bugs.bunny" AND setting = "name";
```

#### DELETE

`DELETE` has the same semantics as `SELECT` in terms of querying. For instance, all the attributes for a keyspace are not required for deletion.

```
DELETE FROM users WHERE username = "elmer.fudd";
```

#### DROP KEYSPACE

`DROP` deletes an entire keyspace.

```
DROP KEYSPACE users;
```

### Conclusion

Obviously, this is just the start. Next time we'll get into code.

But with only 5 query statements we have quite a bit of functionality. Statements that could be added in the future are `DESCRIBE` (for describing keyspaces) and `BATCH` (for large inserts).

There are also a few extra clauses that may be beneficial in the short term. Such as `LIMIT` and possibly `ORDER BY`, although `ORDER BY` might be limited as well for efficiency.


[1]: http://github.com/eliquious/prefixdb/  "PrefixDB on GitHub"
[2]: http://github.com/boltdb/bolt/ "BoltDB on GitHub"

