## v0.3.1

* Requires iyi-db 0.3.0, whose pool closes every connection it holds when
  a `Database` closes; it closed every other one, and left the rest of the
  SQLite files open.

## v0.3.0

* A statement whose step fails is reset. It stayed active, and SQLite
  refused every later `COMMIT` on its connection
  ([crystal-sqlite3#115](https://github.com/crystal-lang/crystal-sqlite3/issues/115)).
* SQLite is initialized before a connection opens. A library built with
  `SQLITE_OMIT_AUTOINIT`, Python's Windows DLL among them, faulted in
  `sqlite3_open_v2`.
* A query with more than one statement answers a `DBError`. The statements
  after the first were dropped unrun.
* `SQLite3Error#extended_code` is SQLite's extended result code
  ([crystal-sqlite3#66](https://github.com/crystal-lang/crystal-sqlite3/issues/66)),
  and `SQLite3Error.new` takes it as its third argument.

## v0.2.0

Requires iyi 0.18.0 and iyi-db 0.2.0.

* The entry point is the package's root module: `import sqlite` and
  `Sqlite::VERSION`, `Sqlite::Any`, where they were
  `import sqlite/sqlite/sqlite` and `Sqlite::Sqlite::...`. iyi 0.18.0 keeps
  a module's artifact free of the modules under its path
  ([iyi#124](https://github.com/iyilang/iyi/issues/124)), which kept the
  entry point off the root. The other modules are where they were.
* `REGEXP` reads text and pattern as UTF-8 characters, as iyi 0.18.0's
  `Regex` does: `'çay' REGEXP '^.ay$'` is 1, where it was 0.
* iyi-db's entry point is its root module too: `import db` and `Db.open`,
  where they were `import db/db/db` and `Db::Db.open`.

## v0.1.0

The first release of iyi-sqlite: crystal-sqlite3 0.24.0 ported to iyi, in
iyi, on iyi-db 0.1.1, with no Crystal code left.

* `sqlite3:` URIs open a file or `:memory:`; `busy_timeout`, `cache_size`,
  `foreign_keys`, `journal_mode`, `synchronous` and `wal_autocheckpoint` in
  the query run as PRAGMAs when a connection opens.
* Prepared statements with `?` arguments, transactions and savepoints,
  `last_insert_id` and `rows_affected`, `SQLite3Connection#dump` through the
  backup API, and a `regexp` function behind `REGEXP`.
* Every integer type, `Float32`, `Bool` and `Time` (as UTC text) read and
  bound as crystal-sqlite3 reads and binds them.

### What iyi's rules changed

* Errors are values: `SQLite3Error` (crystal-sqlite3's `SQLite3::Exception`)
  is a `DBError` with SQLite's `code`, and a connection that cannot be opened
  answers `ConnectionRefused` with it as the cause.
* The driver classes are `SQLite3Driver`, `SQLite3Connection`,
  `SQLite3Statement`, `SQLite3ResultSet` in `sqlite/core`, with the C binding
  private beside them.
* `Time` is written and read by `format_time` and `parse_time`; `Time.local`
  values are gone with the zones.
* An integer column a narrower type cannot hold answers
  `ColumnTypeMismatchError` where crystal-sqlite3 raised `OverflowError`.

### Fixed on the way

* Strings and blobs are bound `SQLITE_TRANSIENT`, so SQLite holds its own
  copy; crystal-sqlite3 bound them `SQLITE_STATIC` and kept references.
* An empty `Bytes` argument binds an empty blob, not `NULL`.
* `REGEXP` with a pattern `Regex` refuses answers an SQL error, and with a
  `NULL` operand answers `NULL`; crystal-sqlite3 raised through SQLite's
  frames and read a null pointer.
* A text column is read to its byte count, so a NUL inside it is kept.
* A handle `sqlite3_open_v2` allocated for a file it could not open is
  closed, and a failed `dump` finishes its backup.

crystal-sqlite3's history before the port is in its
[CHANGELOG](https://github.com/crystal-lang/crystal-sqlite3/blob/master/CHANGELOG.md).
