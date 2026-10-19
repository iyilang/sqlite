# iyi-sqlite

An SQLite3 driver for [iyi](https://iyi.dev): a port of
[crystal-sqlite3](https://github.com/crystal-lang/crystal-sqlite3) 0.24.0,
written in iyi with no Crystal code and no Crystal standard library
underneath. It is a driver for [iyi-db](https://github.com/iyilang/db), the
common database interface, and binds the SQLite C library directly.

It keeps crystal-sqlite3's shape - `sqlite3:` URIs, `?` arguments, `Time` as
text, PRAGMAs from the URI, `REGEXP`, `dump` - and changes what iyi's rules
require: errors are values, types are closed, and tasks are structured.

## Requirements

iyi 0.18.0 or later, iyi-db 0.2.0 (required by this package's `iyi.mod`), and
the SQLite library (`libsqlite3`):

* **Linux:** the development package, `apt install libsqlite3-dev`.
* **macOS:** the system's.
* **Windows:** `sqlite3.lib` and `sqlite3.dll` in a directory named by
  `IYI_LIBRARY_PATH`. The DLL is on SQLite's
  [download page](https://sqlite.org/download.html) (`sqlite-dll-win-x64-*.zip`,
  with `sqlite3.def`); make the import library from it in a Visual Studio
  developer prompt with `lib /def:sqlite3.def /out:sqlite3.lib /machine:x64`.
  A build copies `sqlite3.dll` beside the executable.

## Installation

Add it to a project under the short name `sqlite`:

```sh
iyi get github.com/iyilang/sqlite --as sqlite
```

Files then import the package's root module as `sqlite`, and its other
modules as `sqlite/sqlite/...`: the short name, then the module's path in the
package. Importing `sqlite` (or `sqlite/sqlite/core`) registers the driver for
`sqlite3:` URIs.

## Usage

```iyi
module app

import db
import db/db/errors::{DBError}
import db/db/value::{DBValue}
import sqlite

def contacts : Nil | DBError
  Db.open("sqlite3:./data.db") do |db|
    db.exec("create table contacts (name text, age integer)")!
    db.exec("insert into contacts values (?, ?)", "John Doe", 30)!

    args = [] of DBValue
    args << "Sarah"
    args << 33
    db.exec("insert into contacts values (?, ?)", args: args)!

    puts "max age:"
    puts db.scalar("select max(age) from contacts")! # => 33

    puts "contacts:"
    db.query("select name, age from contacts order by age desc") do |rs|
      puts "#{rs.column_name(0)} (#{rs.column_name(1)})"
      # => name (age)
      rs.each do
        puts "#{rs.read(String)!} (#{rs.read(Int32)!})"
        # => Sarah (33)
        # => John Doe (30)
      end!
    end!

    contacts = db.query_all("SELECT name, age FROM contacts", as: NamedTuple(name: String, age: Int64))!
    puts contacts # => [{name: "John Doe", age: 30}, {name: "Sarah", age: 33}]
  end
end

case contacts
in Nil     then puts "done"
in DBError then puts "failed: #{it.message}"
end
```

The file is the URI's host and path: `sqlite3:./data.db`,
`sqlite3:///abs/path.db`, and `sqlite3:%3Amemory%3A` for an in-memory
database. The query takes iyi-db's pool options (`max_pool_size`, ...) and
the PRAGMAs below. With `prepared_statements=false` every query answers a
`DBError`: SQLite has no unprepared statements.

A failed SQLite call answers a `SQLite3Error` (module `sqlite/errors`), a
`DBError` with SQLite's message and its result `code`.

### Types

| SQLite | read as | bound from |
|---|---|---|
| `INTEGER` | `Int64`; `read(Int8 \| Int16 \| Int32 \| UInt8 \| UInt16 \| UInt32)` (1) | every integer type but `UInt64` |
| `FLOAT` | `Float64`; `read(Float32)` | `Float32`, `Float64` |
| `TEXT` | `String`; `read(Time)` (2) | `String`, `Time` |
| `BLOB` | `Bytes` | `Bytes` |
| `INTEGER` 0/1 | `read(Bool)` | `Bool` |
| `NULL` | `nil` | `nil` |

Each `read(T)` has its nilable `read(T | Nil)`. Any other argument - `Char`,
`UInt64`, `Span`, an array, a `CustomValue` - answers a `DBError`.

1: A value the type cannot hold answers `ColumnTypeMismatchError`.

2: A `Time` is kept as `TEXT` in UTC, written `DATE_FORMAT_SUBSECOND`
(`2016-02-15 10:15:30.123`) and read in that format or `DATE_FORMAT_SECOND`
(`2016-02-15 10:15:30`, SQLite's `current_timestamp`). `format_time(time,
subsecond: false)` and `parse_time(text)` (module `sqlite/dates`) are the two
formats; text in neither answers a `DBError`.

### Setting PRAGMAs

You can adjust certain [SQLite3 PRAGMAs](https://www.sqlite.org/pragma.html)
automatically when the connection is created by using the query parameters:

```iyi
Db.open("sqlite3:./data.db?journal_mode=wal&synchronous=normal") do |db|
  # this database now uses WAL journal and normal synchronous mode
  # (defaults were `delete` and `full`, respectively)
end
```

The following is the list of supported options:

| Name                      | Connection key  |
|---------------------------|-----------------|
| [Busy Timeout][pragma-to] | `busy_timeout` |
| [Cache Size][pragma-cs] | `cache_size` |
| [Foreign Keys][pragma-fk] | `foreign_keys` |
| [Journal Mode][pragma-jm] | `journal_mode` |
| [Synchronous][pragma-sync] | `synchronous` |
| [WAL autocheckoint][pragma-walck] | `wal_autocheckpoint` |

Please note there values passed using these connection keys are passed
directly to SQLite3 without check or evaluation. Using incorrect values result
in no error by the library.

[pragma-to]: https://www.sqlite.org/pragma.html#pragma_busy_timeout
[pragma-cs]: https://www.sqlite.org/pragma.html#pragma_cache_size
[pragma-fk]: https://www.sqlite.org/pragma.html#pragma_foreign_keys
[pragma-jm]: https://www.sqlite.org/pragma.html#pragma_journal_mode
[pragma-sync]: https://www.sqlite.org/pragma.html#pragma_synchronous
[pragma-walck]: https://www.sqlite.org/pragma.html#pragma_wal_autocheckpoint

### REGEXP

Every connection registers the `regexp` function, so `text REGEXP pattern`
works, with iyi's `Regex` (RE2 semantics: no backreferences or lookaround),
which reads text and pattern as UTF-8 characters, so `.` and `[ç]` match `ç`
whole. A pattern `Regex` refuses answers an SQL error naming it, and a `NULL`
operand answers `NULL`.

### Backups

`SQLite3Connection#dump(to)` copies the whole database into another
connection's, file to memory or the other way:

```iyi
db.using_connection do |source|
  backup.using_connection do |target|
    source.as(SQLite3Connection).dump(target.as(SQLite3Connection)) # import sqlite/sqlite/core::{SQLite3Connection}
  end!
end!
```

## Modules

| module | what it holds |
|---|---|
| `sqlite` | `VERSION`, `Any`; re-exports the three below |
| `sqlite/core` | `SQLite3Driver`, `SQLite3Connection` (and its `Options`), `SQLite3Statement`, `SQLite3ResultSet`, `SQLite3ConnectionBuilder`, the `Flag`, `Type` and `Code` enums; registers the driver |
| `sqlite/dates` | `DATE_FORMAT_SUBSECOND`, `DATE_FORMAT_SECOND`, `format_time`, `parse_time` |
| `sqlite/errors` | `SQLite3Error` |

## What is different from crystal-sqlite3

| crystal-sqlite3 | iyi-sqlite | why |
|---|---|---|
| raises `SQLite3::Exception`, `DB::ConnectionRefused` | answers `SQLite3Error` and `ConnectionRefused` (with the `SQLite3Error` as its cause) as `DBError`s | errors are values (SPEC.md III.1) |
| `SQLite3::Connection`, `SQLite3::Statement`, `SQLite3::ResultSet`, `SQLite3::Driver` | `SQLite3Connection`, `SQLite3Statement`, `SQLite3ResultSet`, `SQLite3Driver` in one module | the module imports iyi-db's `Connection`, ... by those names, and the four refer to one another (R-1) |
| `Statement.new` and `Connection.new` prepare and open | `SQLite3Statement.prepare`, `SQLite3Connection.open` answer the error | a constructor has no error to answer |
| `LibSQLite3` | private to `sqlite/core` | a `lib` cannot be exported |
| `Time#to_s(SQLite3::DATE_FORMAT_*)`, `Time.parse` | `format_time`, `parse_time` | `std/time` has no format strings |
| `Time.local` values, converted to UTC | not there | iyi's `Time` is UTC only |
| `to_u8` and friends raise `OverflowError` | `ColumnTypeMismatchError` | errors are values |
| strings and blobs bound `SQLITE_STATIC`, kept alive by the statement | bound `SQLITE_TRANSIENT`: SQLite copies them | nothing has to outlive the call |
| an empty `Bytes` argument bound as `NULL` (a null pointer) | bound as an empty blob | |
| `REGEXP` raised through SQLite's frames, and crashed on `NULL` | an SQL error, and `NULL` | |
| a statement whose step failed left active, and every later `COMMIT` on its connection refused | reset | |
| `SQLite3.flags(...)` macro | `Flag::READWRITE \| Flag::CREATE` | macros are derive-scoped (R-5) |

## Development

```sh
iyi test sqlite
```

The tests are crystal-sqlite3's specs, ported, beside the modules they test,
and iyi-db's shared driver specs (`sqlite/driver_specs_test.iyi`). They write
`./test.db` and `./test2.db` in the working directory and delete them.
`examples/` holds crystal-sqlite3's sample, ported.

## Contributors

crystal-sqlite3 is the work of [Brian J. Cardiff](https://github.com/bcardiff)
and its [contributors](https://github.com/crystal-lang/crystal-sqlite3/graphs/contributors);
this port keeps its licence (LICENSE).
