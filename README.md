# (import (okdb))

![The abstraction of architecture of the Abu Dhabi Louvre Museum ceiling it’s a piece of art on it’s own.](alvaro-pinot-czDvRp5V2b0-unsplash.jpg)

## Status

**Rework draft.**

## Issues

- Limit the key space to `#x00` until `#xFF` excluded, and use the
  rest of the key space for something...

## Abstract

General purpose storage backend datastructure for building in-memory
or on-disk databases that can handle concurrency with ACID
transactions.

## Rationale

`okdb` can be the primitive datastructure for building many
datastructures. Low level extensions include counter, bag, set, and
mapping-multi. Higher level extensions include
[Entity-Attribute-Value](https://en.wikipedia.org/wiki/Entity%E2%80%93attribute%E2%80%93value_model)
possibly supported by datalog, Generic Tuple Store (nstore) inspired
from [Resource Description
Framework](https://en.wikipedia.org/wiki/Resource_Description_Framework)
that can trivialy match the query capabilities of
[SPARQL](https://www.w3.org/TR/rdf-sparql-query/), nstore can
painlessly implement
[RDF-star](https://w3c.github.io/rdf-star/cg-spec/2021-02-18.html), or
even the Versioned Generic Tuple Store (vnstore), that ease the
implementation of bitemporal databases, and datomic high level
interfaces. Also, it is possible to implement a property graph
database, ranked set, leaderboard, priority queue. It is possible to
implement efficiently geometric queries such as xz-ordered curves.

`okdb` is useful in the context of on-disk persistence. `okdb` is also
useful in a context such as client-server applications, where the
client need to cache heterogeneous data. It may be used in the
browser, or in microservice configuration as a shared in-memory
datastructure.

There is several existing databases that expose an interface similar
to `okdb`, and even more that use an ordered key-value store (okvs) as
their backing storage.

While `okdb` interface is lower-level than the mainstream SQL, it is
arguably more useful because the implementation stems from a
well-known datastructure part of every software engineering
curriculum, namely binary trees, also because it allows to implement
SQL, last but not least it reflects the current practice that builds
(distributed) databases systems based on a similar interface.

`okdb` extends, and departs from the common okvs interface inherited
from BerkeleyDB, to ease the implementation thanks to bounded keys and
values, while making the implementation of efficient extensions easier
also thanks to the ability to estimate between two keys the count of
keys, and the size of key-value pairs.

This wann-be [SRFI](https:/srfi.schemers.org/) should offer grounds
for apprentices to learn about data storage. It will also offer a
better story (the best?) for managing data that may be durable, read,
or written concurrently.

## Reference

### `(make-okdb) → okdb?`

Rationale: In SRFI-167, `make-okvs` could take various options. The
interface was difficult, and did not work well. Instead, of trying to
define a couple of options, a left aside others. With `okdb` it is
left to downstream to deal with options. It is the responsability of
the implementer, and possibly eventually to the user to deal with
options in an appropriate way. One way, for the implementer to enable
options is to create a super procedure that a) returns multiple
values, including the constructor, b) rely on generic methods, or
something else.

Return a handle of the database.

### `(okdb? obj) * → boolean?`

Returns `#t` if `OBJ` is an `<okdb>` instance. Otherwise, returns
`#f`.

### `(okdb-close! db) okdb?`

Close `DB`.

### `(okdb-transaction? obj) * → boolean?`

Returns `#t` if `OBJ` is an `<okdb-transaction>` instance. Otherwise,
returns `#f`.

### `(okdb-cursor? obj) * → boolean?`

Returns `#t` if `OBJ` is an `<okdb-cursor>` instance. Otherwise,
returns `#f`.

### `(okdb-handle? obj) * → boolean?`

Returns `#t` if `OBJ` satisfy either `okdb?`, `okdb-transaction?`, or
`okdb-cursor?`. Otherwise, returns `#f`.

### `(okdb-key-max-size handle) handle? → number?`

Return the maximum size of a key for the database associated with
`HANDLE`. If `okdb-key-max-size!` was never called, return a default
value.

### `(okdb-key-max-size! handle size) okdb? number?`

Set the maximum `SIZE` of a key for the database associated with
`HANDLE`. If `SIZE` is not supported, it is an error.

### `(okdb-value-max-size handle) handle? → number?`

Return the maximum size of a value of the database associated with
`HANDLE`. If `okdb-value-max-size!` was never called, return a default
value.

### `(okdb-value-max-size! handle size) handle? number?`

Set the maximum `SIZE` of a value of the database associated with
`OKDB`.

### `(okdb-conflict? obj)`

Returns `#t` if `OBJ` is a conflict error. Otherwise returns
`#f`. Such object may be raised by `okdb-in-transaction`.

### `(make-okdb-transaction-parameter init)` any? → procedure?

Returns a procedure that may take one or two arguments:

- One argument: the procedure takes a transaction as first argument
and returns the current value for the given transaction. `INIT` is the
initial value.

- Two arguments: the procedure takes a transaction as first argument,
and a new value for the associated transaction. It returns no values.

In the following example, `okdb-in-transaction` will return `#f`:

```scheme
(define read-only? (make-okdb-transaction-parameter #t))

(define (proc tx)
  (display (read-only? tx)) ;; => #t
  (okdb-set! tx #u(42) #u8(13 37))
  (read-only? tx #f)
  ...
  (read-only? tx))

(okdb-in-transaction okdb proc) ;; => #f
```

### `(okdb-transaction-parametrize ((parameter value) ...) expr ...)`

Similar to `parametrize`.

### `(okdb-transaction-hook-begin handle)`

Returns SRFI-173 hook associated with the beginning of the
transaction.  This hook give a chance to extension libraries to
initialize internal states.

### `(okdb-transaction-hook-pre-commit handle)`

Returns SRFI-173 hook associated with the end of the transaction.
This hook give a chance to extension libraries to execute triggers.

### `(okdb-transaction-hook-post-commit handle)`

Returns SRFI-173 hook associated with the success of the transaction.
This hook may be used to implement features such as notify or watches.

### `(okdb-transaction-hygiene [symbol])` symbol? → symbol?

The parameter `okdb-transaction-hygiene` may be used to get or set
transaction guarantees related to isolation. The following symbols
may be accepted:

- read-uncommitted
- read-committed
- snapshot
- serializable

ref: https://source.wiredtiger.com/10.0.0/transactions.html#transaction_isolation

### `(okdb-in-transaction okdb proc [failure [success]]) okdb? procedure? procedure? procedure? → any? ... ↑ okdb-conflict?`

Rationales:

- `okdb-in-transaction` does not include a retry logic when
`okdb-conflict?` is raised because retrying might require to wait
which depends on the implementation but also and more importantly on
user code. The user is in the best position to know when, and how to
retry the transaction. The last resort strategy is not even to retry
the transaction immediatly, but to put the operation in queue possibly
persisted in the database, and force the serialization through a
single thread. In any case, retry should be explicit in user code.

- Nested transactions were ruled out. Nested transactions are similar
to savepoints or autonomous transactions. They are out because it is
still unclear whether they put a strain on the implementation that
does not yield much benefit in user code.

`okvs-in-transaction` describes the extent of the atomic property, the
A in [ACID](https://en.wikipedia.org/wiki/ACID), of changes against
the underlying database. A transaction will apply all database
coperations in `PROC` or none: all or nothing. When
`okdb-in-transaction` returns successfully, the changes will be
visible for future transactions, and implement durability, D in
ACID. In case of error, changes will not be visible to other
transactions in all cases. Regarding isolation, and consistency,
respectively the I and C in ACID, it depends on the parameter
`okdb-transaction-hygiene`.

Begin a transaction against the database, and execute `PROC`. `PROC`
is called with first and only argument an object that satisfy
`okdb-transaction?`. In case of error, rollback the transaction and
execute `FAILURE` with the error object as argument. The default value
of `FAILURE` re-raise the error with `raise`. Otherwise, executes
`SUCCESS` with the returned values of `PROC`.  The default value of
`SUCCESS` is the procedure `values`.

When the transaction begin, `okdb-in-transaction` must call the
procedures associated with `okdb-transaction-hook-begin`.

Just before the transaction commit is tried, `okdb-in-transaction`
must call the procedures associated with
`okdb-transaction-hook-pre-commit`.

Just after the transaction commit is a success, `okdb-in-transaction`
must call the procedures associated with
`okdb-transaction-hook-post-commit`.

`okdb` does not support nested transactions.

In case `okvs-in-transactions` raise an error that satisfy
`okdb-conflict?`, the user may re-run the same transaction taking care
that `PROC` is idempotent.

### `(okdb-cursor-positioned? cursor) okdb-cursor? → boolean?`

Returns `#t` if `CURSOR` has a position, otherwise returns `#f`.

It is an error to call `okdb-cursor-next?`, `okdb-cursor-previous?`,
`okdb-cursor-key`, or `okdb-cursor-value`, when the cursor is not
positioned.

### `(okdb-call-with-cursor handle proc) okdb-handle? procedure? → any? ...`

Open a cursor against `HANDLE` and call `PROC` with it. When `PROC`
returns, the cursor is closed.

If `HANDLE` satisfy `okdb?`, `okdb-call-with-cursor` must wrap the
call to `PROC` with `okvs-in-transaction`.

If `HANDLE` satisfy `okdb-cursor?`, `okdb-call-with-cursor` must pass
a cursor positioned at the same position as `HANDLE`.

The produced cursor is not positioned, except when `HANDLE` satisfy
`okdb-cursor?` and te associated cursor is also positioned.

The cursor is stable: the cursor will see a snapshot of keys of the
database taken when `okdb-call-with-cursor` is called. During the
extent of `PROC`, `okdb-set!` and `okdb-remove!`  will not change the
position of the cursor, and the cursor will see removed keys and not
see added keys. Keys which value was changed during cursor navigation,
that exist when `okdb-call-with-cursor` is called, can be seen.

### `(okdb-estimate-key-count handle [key other [offset [limit]]]) handle? bytevector? bytevector? integer? integer? → integer?`

Rationale: It is helpful to know how big is a range to be able to tell
which index to use as seed. Imagine a query against two attributes,
each attribute with their own index and no compound index: being able
to tell which subspace contains less keys, can speed up significantly
query time.

Return an estimate count of keys between `KEY` and `OTHER`. If `KEY`
and `OTHER` are omitted return the approximate count of keys in the
whole database.

If `OFFSET` is provided, `okdb-estimate-key-count` will skip the first
`OFFSET` keys from the count.

If `LIMIT` is provided, `okdb-estimate-key-count` will consider `LIMIT`
keys from the count.

### `(okdb-estimate-bytes-count handle [key [other [offset [limit]]]]) okdb-handle? bytevector? bytevector? integer? integer? → integer?`

Rationale: That is useful in cases where the size of a transaction is
limited.

Return the estimated size in bytes of key-value pairs in the subspace
described by `KEY` and `OTHER`. If `OTHER` is omitted, return the
approximate size of the key-value pair associated with
`KEY`. Otherwise, return the estimated size of the whole database
associated with `HANDLE`.

If `OFFSET` is provided, `okdb-estimate-bytes-count` will skip the
first `OFFSET` keys from the count.

If `LIMIT` is provided, `okdb-estimate-bytes-count` will consider `LIMIT`
keys from the count.

### `(okdb-set! handle key value) okdb-handle? bytevector? bytevector?`

Associate the bytevector `KEY`, with the bytevector `VALUE`.

If `HANDLE` satisfy `okdb-cursor?`, `okdb-set!` does not change the
position of `HANDLE`.

### `(okdb-remove! handle key) okdb-handle? bytevector?`

Removes the bytevector `KEY`, and its associated value.

If `HANDLE` satisfy `okdb-cursor?`, `okdb-set!` does not change the
position of `HANDLE`.

### `(okdb-cursor-seek cursor strategy key) okdb-cursor? symbol? bytevector? → symbol?`

Position the `CURSOR` using `STRATEGY`. `STRATEGY` can be one of the
following symbol:

- `less-than-or-equal`
- `equal`
- `greater-than-or-equal`

The strategy `less-than-or-equal` will first seek for the biggest key
that is less than `KEY`, if there is one, it returns the symbol
`less`.  Otherwise, if there is a key that is equal to `KEY` it will
return the symbol `equal`. Otherwise, it fallbacks to the symbol
`not-found`.

The strategy `equal` will seek a key that is equal to `KEY`. If there
is one it will return the symbol `equal`. Otherwise, it returns the
symbol `not-found`.

The strategy `greater-than-equal` will first seek the smallest key
that is greater than `KEY`, if there is one, it returns the symbol
`greater`.  Otherwise, if there is a key that is equal to `KEY` it
will return the symbol `equal`. Otherwise, it fallbacks the symbol
`not-found`.

### `(okdb-cursor-next? cursor) okdb-cursor? → boolean?`

Move the `CURSOR` to the next key if any. Return `#t` if there is such
a key. Otherwise returns `#f`. `#f` means the cursor reached the end
of the key space.

### `(okdb-cursor-previous? cursor) okdb-cursor? → boolean?`

Move the `CURSOR` to the previous key if any. Return `#t` if there is
such a key. Otherwise returns `#f`. `#f` means the cursor reached the
begining of the key space.

### `(okdb-cursor-key cursor)` okdb-cursor? → bytevector?`

Return the key bytevector where `CURSOR` is positioned. It is an error
to call `okdb-cursor-key`, when `CURSOR` reached the begining or end
of the key space or when `CURSOR` is not positioned.

### `(okdb-cursor-value cursor)` okdb-cursor? → bytevector?`

Return the value bytevector where `CURSOR` is positioned. It is an
error to call `okdb-cursor-key`, when `CURSOR` reached the begining or
end of the key space or when `CURSOR` is not positioned.

### `(okdb-query handle key [other [offset [limit]]]) handle? bytevector? bytevector? integer? integer? → (either? bytevector? procedure?)`

`OKDB-QUERY` will query the associated database. If only `KEY` is
provided it will return the associated value bytevector or `#f`. If
`OTHER` is provided there is two cases:

- `KEY < OTHER` then `okdb-query` returns a generator with all the
  key-value pairs present in the database between `KEY` and `OTHER`
  excluded ie. without the key-value pair associated with `OTHER` if
  any.

- `OTHER < KEY` then `okdb-query` returns the equivalent of reversing
  the generator returned by `(okdb-query handle OTHER KEY)`.

If `OFFSET` is provided the generator will skip as much key-value
pairs from the start of the generator.

If `LIMIT` is provided the generator will generate at most `LIMIT`
key-value pairs.

### `(okdb-bytevector-next-prefix bytevector) bytevector? → bytevector?`

Return the bytevector that follows `BYTEVECTOR` according to
lexicographic order that is not prefix of `BYTEVECTOR` such as the
following code iterates over all keys that have `key` as prefix:

```scheme
(okdb-query handle key (okdb-bytevector-next-prefix key))
```
