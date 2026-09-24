# libpq-sys

libpq is the C application programmer's interface to PostgreSQL. It is
the library most other PostgreSQL client libraries are built on, and it
speaks the frontend/backend protocol to a server over a TCP socket or a
Unix-domain socket. Its API is documented in
[the libpq chapter of the PostgreSQL manual](https://www.postgresql.org/docs/current/libpq.html).
This package declares forty-three of that library's entry points to
novo-lang, one declaration each.

Every function here is a declaration of a function in libpq. The
package contains no logic of its own, and it does nothing without the C
library installed. The forty-three entry points are the ones a program
needs to connect, run commands with or without parameters, read the
rows back and report a failure. The section "What is not included" says
what a program cannot do with them alone.

## What it is

A **connection** is one session with one database on one server.
`PQconnectdb` opens it from a **connection string**: a sequence of
`keyword=value` pairs, such as
`"host=localhost dbname=orders connect_timeout=5"`, or a URI beginning
`postgresql://`. `PQfinish` closes it.

libpq answers a connection handle even when the connection failed, and
`PQstatus` is what says which happened. The handle must be closed
either way, and `PQerrorMessage` on it is what says why.

A **result** is everything one command produced: its status, its rows,
its column descriptions and, if it failed, its message. `PQexec` runs a
command and answers one, and `PQclear` releases it. A result holds its
own copy of the data, so it stays valid after the connection is closed.

A **result status** says what kind of result it is. `PGRES_COMMAND_OK`
(1) is a command that returned no rows, `PGRES_TUPLES_OK` (2) a command
that returned some, and `PGRES_FATAL_ERROR` (7) a command that failed.

| Code | Name | What it means |
| --- | --- | --- |
| 0 | `PGRES_EMPTY_QUERY` | the command string was empty |
| 1 | `PGRES_COMMAND_OK` | the command returned no rows |
| 2 | `PGRES_TUPLES_OK` | the command returned rows |
| 5 | `PGRES_BAD_RESPONSE` | the server's answer could not be understood |
| 6 | `PGRES_NONFATAL_ERROR` | a notice or a warning |
| 7 | `PGRES_FATAL_ERROR` | the command failed |

libpq passes a result of status 6 to the notice processor rather than
answering it from a query call. The statuses the table leaves out
belong to COPY, single-row mode and pipeline mode, which this package
does not include.

A **parameter** is a value supplied beside the command rather than
inside it. `PQexecParams` takes a command referring to `$1`, `$2` and
so on, and an array of values. The server never sees the values as part
of the command text, so nothing in a value can change what the command
does.

A **prepared statement** is a command the server parses and plans once,
under a name. `PQprepare` creates one and `PQexecPrepared` runs it.

## Install

```
novo pkg add libpq-sys
```

Adding the package does not install the C library, and it does not
install a PostgreSQL server. On Debian and Ubuntu the client library
and its header come from the system package `libpq-dev`:

```
sudo apt install libpq-dev
```

On macOS the Homebrew formula is `libpq`, and its library is not on the
default search path. On other systems it builds with the rest of the
PostgreSQL source.

The header is installed under `/usr/include/postgresql` rather than
`/usr/include`, so the manifest asks pkg-config for `libpq` rather
than relying on the default flag.

## Example

One query, and its rows printed:

```novo ignore
use libpq

fn main() [io, ffi]
    let conn = libpq.pq_connectdb("host=localhost dbname=orders connect_timeout=5")
    // A handle is answered even for a connection that failed.
    if libpq.pq_status(conn) != 0
        println(ptr.read_str(libpq.pq_error_message(conn)))
        libpq.pq_finish(conn)
        return

    let res = libpq.pq_exec(conn, "SELECT city, people FROM town ORDER BY people DESC")
    // 2 is PGRES_TUPLES_OK, a command that returned rows.
    if libpq.pq_result_status(res) != 2
        println(ptr.read_str(libpq.pq_result_error_message(res)))
        libpq.pq_clear(res)
        libpq.pq_finish(conn)
        return

    let rows = libpq.pq_ntuples(res)
    for i in 0..rows
        let city = ptr.read_str(libpq.pq_getvalue(res, i, 0))
        // An empty string is also what a NULL reads as, so the two are
        // told apart by asking.
        if libpq.pq_getisnull(res, i, 1) == 1
            println("${city}: unknown")
        else
            println("${city}: " + ptr.read_str(libpq.pq_getvalue(res, i, 1)))

    libpq.pq_clear(res)
    libpq.pq_finish(conn)
```

The fence reads `novo ignore`, so `novo doc` lists the example and does
not compile it. A compiled block is linked against libpq, and the
example also needs a running PostgreSQL server. The examples in the
declarations' comments are fenced the same way.

## What the package contains

| Module | Contents |
| --- | --- |
| `libpq` | Every entry point, in five groups: the connection, the commands, the quoting, the result and the asynchronous calls. |

The five groups and their sizes:

| Group | Entry points | What it does |
| --- | --- | --- |
| Connection | 16 | Opens and closes a session, describes it, and reports its status and its versions. |
| Commands | 4 | Runs a command, with parameters or without, directly or prepared. |
| Quoting | 3 | Quotes a literal or an identifier for this connection, and releases what it allocated. |
| Result | 14 | Reads the status, the rows, the columns and the message of one command. |
| Asynchronous | 6 | Sends a command without waiting, and drives the socket by hand. |

## How to choose an entry point

`PQexec` is the short path, with one call, one wait and one result. It
accepts several semicolon-separated commands, which run in a single
transaction unless the string contains BEGIN and COMMIT, and it
answers only the last command's result.

`PQexecParams` accepts exactly one command. A value supplied out of
line is never parsed as SQL, so it cannot add a second. Use it whenever
a value comes from outside the program.

`PQprepare` and `PQexecPrepared` are for a command run many times. The
server parses and plans it once.

`PQsendQuery` with `PQconsumeInput`, `PQisBusy` and `PQgetResult` is for
a program that cannot block. The program waits on the socket itself
between calls. `PQsocket` gives the descriptor to wait on.

## The rules a user needs

1. **A pointer is an `Int`, and zero is null.** The connection handle
   and the result handle arrive as the addresses libpq returned.
2. **`PQconnectdb` answers a handle even when it failed.** Check
   `PQstatus`, and call `PQfinish` either way. A handle is 0 only when
   memory ran out. PostgreSQL manual, libpq, "Database Connection
   Control Functions".
3. **An entry point that answers a C `int` answers it in 32 bits.**
   Write `as i32` before comparing the answer with a negative number.
   `PQsocket`, `PQfnumber`, `PQsetnonblocking` and `PQflush` all answer
   -1 in ordinary use.
4. **Every result must be released with `PQclear`.** A result is a
   separate allocation from the connection, and closing the connection
   does not release it.
5. **A NULL field and an empty field read the same.** `PQgetvalue`
   answers an empty string for both. `PQgetisnull` is the only way to
   tell them apart. PostgreSQL manual, libpq, "Retrieving Query Result
   Information".
6. **A string an accessor answers belongs to the connection or the
   result.** Copy it with `ptr.read_str` before the owner is closed or
   cleared. The two exceptions are `PQescapeLiteral` and
   `PQescapeIdentifier`, whose answers belong to the caller and are
   released with `PQfreemem`.
7. **`PQerrorMessage` is overwritten by the next operation.**
   `PQresultErrorMessage` is not. It stays valid for as long as its
   result does. Use the second when the message has to outlive the
   next call.
8. **Pass a value as a parameter rather than building it into the
   command text.** `PQexecParams` and `PQexecPrepared` send the values
   separately, and the server never parses them as SQL. When a value
   such as a table name must go into the text, `PQescapeIdentifier` and
   `PQescapeLiteral` are the calls. Both need the connection, because
   the quoting depends on its encoding.
9. **An array argument is an array of addresses.** `param_values` is
   the address of an array of `n_params` addresses, laid out with
   `ptr.alloc` and `ptr.write_word`. A 0 in it is a SQL NULL. Passing 0
   for `param_types`, `param_lengths` or `param_formats` asks libpq to
   infer the type, read each value up to its terminator, and treat
   every value as text.
10. **`PQgetResult` must be called until it answers 0.** That 0 is what
    says a command sent with `PQsendQuery` is finished, and the
    connection cannot take another command until it arrives.
11. **The socket from `PQsocket` is for waiting on, not for reading.**
    Reading or writing it directly corrupts the protocol stream.

## What is not included

- **`PQsetdbLogin`.** It takes seven `const char *` arguments, most of
  which are normally the null pointer, and a novo-lang `Str` cannot be
  null. `PQconnectdb`'s connection string carries the same settings and
  more.
- **`PQconnectdbParams` and `PQconninfo`.** They take and answer arrays
  of `PQconninfoOption` structures.
- **The asynchronous connection.** `PQconnectStart` and `PQconnectPoll`
  open a connection without blocking. They are left out of this
  release. `PQconnectdb` blocks until the connection succeeds or the
  timeout in the connection string expires.
- **`PQnotifies` and LISTEN/NOTIFY.** A notification is a `PGnotify`
  structure the caller reads fields out of.
- **The notice callbacks.** `PQsetNoticeReceiver` and
  `PQsetNoticeProcessor` take C function pointers. Without them a
  notice from the server is printed to standard error, which is
  libpq's default.
- **The COPY functions.** `PQputCopyData`, `PQgetCopyData` and their
  neighbours are the bulk load and unload path. They are left out of
  this release.
- **The large object functions.** `lo_open`, `lo_read` and their
  neighbours are a separate facility for values too large for a row.
- **Single-row mode and pipeline mode.** `PQsetSingleRowMode` and the
  `PQpipeline*` family change how results arrive. They are left out of
  this release.
- **`PQresultErrorField`.** It answers one field of the structured
  error, such as the SQLSTATE code, the constraint name or the
  position. It is left out of this release. `PQresultErrorMessage`
  carries the same information as text.
- **The cancellation functions.** `PQgetCancel` and `PQcancel` answer
  and use a `PGcancel` object. `PQbackendPID` is here, which is what a
  second connection needs to cancel a query with `pg_cancel_backend`.

## Related packages

[postgres-nv](https://novo-lang.org/packages/postgres-nv) is a
PostgreSQL client written in novo-lang, speaking the frontend/backend
protocol directly with no C library. It is published as an interface
release. Every function in it is declared and none has a body yet, so
a program that must reach a server today uses this package.

When its functions have bodies, `postgres-nv` is the choice for a
program that must build for WebAssembly or does without a C toolchain.
This package is the choice for a program that needs libpq's own
connection handling, which covers its service file, its `.pgpass`, its
SSL negotiation and its GSSAPI support. It is also the choice for a
program that must behave exactly as an existing libpq client does.

## Tests

`tests/libpq_tests.nv` holds ten tests over the forty-three entry
points. They call the C library, so `novo test` needs libpq installed and
linkable:

```
novo test tests/libpq_tests.nv
```

`novo pkg build` type-checks the declarations and needs nothing
installed.

No test needs a PostgreSQL server. Every connection in the suite is
made to a Unix-domain socket directory that does not exist, which fails
at once and with no network traffic. libpq answers a connection handle
for a failed connection just as it does for a good one, and most of the
connection surface answers on it.

The suite asserts that the client library reports its version, that a
result status has the name the reference gives it, that pinging a
server that is not there answers `PQPING_NO_RESPONSE`, that a failed
connection reports the database, user, host and port it was asked for
and a socket of -1, that reopening it leaves it bad, that a command on
it fails, that the parameterised calls accept their arrays of
addresses, that `PQescapeLiteral` turns `O'Hara` into `'O''Hara'`, and
that each of the six asynchronous calls refuses in the way the
reference says it does. The column accessors need rows from a live
server. Their assertions run only when a result has rows, so in this
suite the compiler checks the calls and the test asserts that a failed
connection gave no rows.

libpq is not installed on the machine where this package is written,
so the suite has not linked there and none of its assertions has been
observed to pass. `novo test` stops at the link, naming `-lpq`.

## Licence

Apache-2.0. See [LICENSE](LICENSE).

PostgreSQL and libpq are distributed under the PostgreSQL Licence, and
installing them is the reader's own step.
