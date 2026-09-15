# Changelog

All notable changes to libpq-sys are recorded here. The format
is [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-09-15

The first release: forty-three entry points of the libpq C API, one
`@ffi` declaration each, and no logic.

### Added

- `libpq` — the whole surface, in five groups.
  - The connection: `PQconnectdb`, `PQfinish`, `PQreset`, `PQstatus`,
    `PQtransactionStatus`, `PQerrorMessage`, the four option
    accessors, `PQsocket`, `PQbackendPID`, the two version numbers,
    `PQping` and `PQlibVersion`.
  - The commands: `PQexec`, `PQexecParams`, `PQprepare` and
    `PQexecPrepared`.
  - The quoting: `PQescapeLiteral`, `PQescapeIdentifier` and
    `PQfreemem`.
  - The result: the status and its name, the error message, the row
    and column counts, the column name, number and type, the three
    field readers, the two command tags and `PQclear`.
  - The asynchronous calls: `PQsendQuery`, `PQgetResult`,
    `PQconsumeInput`, `PQisBusy`, `PQsetnonblocking` and `PQflush`.
- `tests/libpq_tests.nv` — ten tests over the signatures. They call
  the C library, so they need libpq installed. No test needs a
  PostgreSQL server: every connection is made to a Unix-domain socket
  directory that does not exist, and libpq answers a connection handle
  for a failed connection just as it does for a good one.

### Not a `0.0.x` interface release

An interface release is the shape whose every `pub fn` body is a
`todo()`. Every `pub fn` here is an `@ffi` declaration with no body, so
`novo pkg publish` reads the package as a release with bodies and
refuses a `0.0.x` version for it. The first release of a bindings
package is therefore `0.1.0`.

### Named as missing

**`PQsetdbLogin`.** It takes seven `const char *` arguments, most of
which are normally the null pointer, and a novo-lang `Str` cannot be
null. `PQconnectdb`'s keyword and value string carries the same
settings and more.

**Every entry point that takes a C function pointer.** The notice
receiver and the notice processor are absent, so a notice from the
server goes to standard error, which is libpq's default.

**Passing a structure by value.** `PQconnectdbParams`'s option array,
`PQconninfo`'s `PQconninfoOption` array and `PQnotifies`'s `PGnotify`
are all structures with fields at offsets a binding cannot promise.
That also removes the COPY interface, the large object interface, the
single-row mode and the pipeline mode from the first release.
