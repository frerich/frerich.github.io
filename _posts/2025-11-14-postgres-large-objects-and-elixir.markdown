---
layout: post
title:  "PostgreSQL Large Objects and Elixir"
date:   2025-11-14 20:36:24 +0100
categories: postgres elixir
---
An application wishing to store larger amounts of data, e.g. big images, movies
or files, typically has two main options for doing so:

1. A new column on some table can be introduced; Postgres features a `bytea`
   type for this purpose. This is easy to implement but requires holding the
   the complete data in memory when reading or writing, something which
   may not be viable beyond a few dozen megabytes. Efficient streaming or
   random-access operations (such as reading specific portions of the file) are
   not practical.
2. A separate cloud storage (e.g. AWS S3) could be used. This permits streaming,
   but requires complicating the tech stack by depending on a new service.
   Coordinating the two systems, e.g. when querying the database for a specific
   set of users and then fetching the documents uploaded by each user from S3
   requires Elixir support. This is especially tricky when writing data, when a
   failure to write to one of the involved systems should cause changes to the
   other to be reverted.

There are further solutions with different trade offs, but there's one in
particular which maybe doesn't get the attention it deserves: Postgres 'Large
Objects'.

# Large Objects

The [PostgreSQL 7.1 documentation](https://www.postgresql.org/docs/7.1/largeobjects.html) explains:

> Originally, Postgres 4.2 supported three standard implementations of large objects

PostgreSQL 4.2 was released on Jun 30th, 1994. The large objects facility has
been around for a long time!

It enables efficient streaming access to large (up to 4TB) files as well as
random access operations (e.g. reading or writing a subset of a file). This
solves the above-mentioned problems:

1. Unlike values in table columns, large objects can be streamed into/out of
   the database and permit random access operations.
2. Unlike e.g. S3, no new technology is needed. Large objects live side-by-side
   with the tables referencing them, operations like ‘Delete all uploads for a
   given user ID’ are just one `SELECT` statement.

Yet, this feature is fairly unknown to many programmers - or considered too
unwieldy to use for productive usage. This is not by accident - there are
various trade offs to consider when deciding if large objects are a good
mechanism for storing large amounts of binary data.

# Large Object API

The official [Large Objects
documentation](https://www.postgresql.org/docs/current/largeobjects.html) does
a great job at describing implementation features and the available APIs. For
the most part however, it focuses on the client-side API which is composed of a
set of C functions. The API is largely  modeled after the Unix file-system
interface, with analogues of `open`, `read`, `write`, `lseek`, etc..

Furthermore, there are APIs for
[importing](https://www.postgresql.org/docs/current/lo-interfaces.html#LO-IMPORT)
and
[exporting](https://www.postgresql.org/docs/current/lo-interfaces.html#LO-EXPORT)
large objects from files, streaming the data such that the process uses a
constant amount of memory. However, these APIs require that the files being
read from or written to exist on the PostgreSQL server itself! There's no
ready-made API for streaming data from/to a client.

A new library seeks to remedy this situation for Elixir developers: the
[pg_large_objects](https://hexdocs.pm/pg_large_objects) library.

The library features four main benefits:

# 1. High-level API for memory-efficient streaming

The high-level API of
[PgLargeObjects](https://hexdocs.pm/pg_large_objects/PgLargeObjects.html)
permits streaming large amounts of data (up to 4TB) while using a constant
amount of memory. The API references objects by 'object IDs', which are plain
integers.  For example, this is how to export a local object to a local file in
a streaming fashion:

```elixir
Repo.transact(fn ->
  path = "/tmp/bigfile.dat"

  :ok = Repo.export_large_object(object_id, into: File.stream!(path))

  {:ok, path}
end)
```

Importing works much the same way. For instance, this is how to create a new
large object given a binary of data:

```elixir
Repo.transact(fn ->
  large_binary = "This is a large binary."
  {:ok, object_id} = Repo.export_large_object(large_binary)
end)
```

What's important to note is that internally, the file descriptors used to
reference large objects are only valid for the duration of a transaction - they
are closed automatically as the transaction ends. Thus, the vast majority of
operations on large objects - such as streaming data into or out of them - need
to happen in a transaction as in the examples above to avoid operating on
invalid (closed) handles.

# 2. Random-Access Reads and Writes via Low-Level API

Instead of operating on objects as a whole, it's also possible to read and
write small chunks of data to arbitrary positions within the file by using the
low-level
[PgLargeObjects.LargeObject](https://hexdocs.pm/pg_large_objects/PgLargeObjects.LargeObject.html)
API.

For example, this is how to read the last 1024 bytes of an object:

```elixir
alias PgLargeObjects.LargeObject

{:ok, data} =
  Repo.transact(fn ->
    with {:ok, object} <- LargeObject.open(object_id, mode: :read),
         {:ok, _position} <- LargeObject.seek(object, -1024, :end) do
      LargeObject.read(object, 1024)

      # Object gets closed automatically at end of transaction.
    end
  end)
```

For writing the word 'Hello' at offset 10 in an object, you could use

```elixir
alias PgLargeObjects.LargeObject

{:ok, data} =
  Repo.transact(fn ->
    with {:ok, object} <- LargeObject.open(object_id, mode: :write),
         {:ok, _position} <- LargeObject.seek(object, 10) do
      LargeObject.write(object, "Hello")
    end
  end)
```

Furthermore, `LargeObject` structures as returned by e.g. `LargeObject.open/1`
implement both the `Enumerable` as well as `Collectable` protocol and thus can
be accessed via functions in the `Enum` or `Stream` modules.

This low-level API not only provides a huge amount of control, it also enables
integrating the library with other Elixir libraries, e.g. by defining an
appropriate implementation of
[Phoenix.LiveView.UploadWriter](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.UploadWriter.html).


# 3. Extensions to the Ecto query DSL

The
[PgLargeObjects.EctoQuery](https://hexdocs.pm/pg_large_objects/PgLargeObjects.EctoQuery.html)
module defines macros which permit calling the low-level API for large objects
directly as part of a query. Quoting the documentation, here is an example of
how to remove any uploads associated with some imaginary user in an
application's table of uploads:

```elixir
import Ecto.Query
import PgLargeObjects.EctoQuery

# Delete all data uploaded by a given user.
from(upload in Upload,
  where: upload.user_id == ^user_id,
  select: lo_unlink(upload.object_id)
) |> Repo.all()
```

# 4. Integration with Phoenix LiveView uploads

[PgLargeObjects.UploadWriter](https://hexdocs.pm/pg_large_objects/PgLargeObjects.UploadWriter.html)
is a ready-to-use definition ot the
[Phoenix.LiveView.UploadWriter](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.UploadWriter.html)
behaviour, making it easy to have LiveView uploads stream data directly into
large objects.

First, the call to `Phoenix.LiveView.allow_upload/3` is adjusted to pass the
`:writer` option, referencing the module:

```elixir
socket
|> allow_upload(:photo,
  accept: :any,
  writer: fn _name, _entry, _socket ->
    {PgLargeObjects.UploadWriter, repo: MyApp.Repo}
  end
)
```

This will cause any upload to get written into a large object. The object IDs
referencing these objects can then be fetched via
`Phoenix.LiveView.consume_uploaded_entries/3`, e.g.:

```elixir
consume_uploaded_entries(socket, :photo, fn meta, _entry ->
  %{object_id: object_id} = meta

  # Store `object_id` in database to retain handle to uploaded data.

  {:ok, nil}
end)
```

# Trade-Offs

Despite the appealing features of large objects and the easy of use provided by
the pg_large_objects library, using this facility for storing large data blobs
is not a no-brainer. As usual, a set of trade-offs needs to be considered to
decide whether this is a viable storage mechanism for an application.

## Partial vs. Complete Data

The pg_large_objects library, at its highest level, exposes large objects as data
streams by defining appropriate implementations of the Enumerable (for reading)
and Collectable (for writing) behaviour. This is only possible by taking advantage
of the fact that large objects enable working with *partial* data. Objects can be
read and written in small chunks, and operations like
`PgLargeObjects.LargeObject.seek/2` enable accessing individual parts of a large
object without loading the entire data into memory.

If your application always only needs to work with the entire data as a whole,
and loading it into memory as a whole is possible and convenient, the
application might be better off with storing the data in a `bytea` column of
a table.

## Storage Costs

Storing large objects in a PostgreSQL database may greatly increase the amount
of disk space used by the database. This may be more expensive than other
mechanisms for storing large objects.

For example, the [AWS RDS
documentation](https://aws.amazon.com/rds/postgresql/pricing/) (RDS is Amazon's
managed database offering) explains that at the time of this writing, 1GB of
General Purpose storage for a in the us-east-1 region costs $0.115 per month
for a PostgreSQL database. The [AWS S3
documentation](https://aws.amazon.com/s3/pricing/) (S3 is Amazon's object
storage offering) documents, at the time of this writing, that storing 1GB of
data in the us-east-1 region is a mere $0.023 per month!

I.e. when using Amazon cloud services in the us-east-1 region, storing data in
RDS is five times as expensive as storing it in S3. Depending on the amount of
data and your budget, this might be a significant difference.

Make sure to check the pricing (if applicable) for storage used by your
PostgreSQL database and consider the change in the decision whether to use
large objects or not.

## Backups

Given that large objects may quickly end up being the bulk of data stored in a
database, it's common to configure backups to exclude them from backups or only
include them in weekly backups or similar.

For example, the `pg_dump` command line utility features four related options:

```
frerich@Mac ~ % pg_dump --help
[..]
  -b, --large-objects          include large objects in dump
  --blobs                      (same as --large-objects, deprecated)
  -B, --no-large-objects       exclude large objects in dump
  --no-blobs                   (same as --no-large-objects, deprecated)
[..]
```

Consider your current backup mechanism and see if it's configured to include or
exclude large objects. Decide on the important of large objects for your use
case and include that in your decision on how often large objects should be
included in backups.
