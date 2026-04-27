---
title: "Lessons from building a custom Postgres type with Rust and Pgrx"
author: Hugo Tunius
theme: 
    name: light
#  path: theme.yaml

---


<!-- jump_to_middle -->
```sql
-- There will be code, here's how large the font will be.
SELECT gen_plid('usr') AS user_id;
```

<!-- end_slide -->

# Me


_Hugo of all trades_


* Originally from Sweden, called Edinburgh home for more than 10 years
* Rust since 2017, professionally at Lookback since 2021
* Postgres on and off throughout my career

<!-- end_slide -->

# Background - Motivation

<!-- pause -->

<!-- incremental_lists: true -->

* UUIDv4 is nice, compact but opaque identifier. Poor index locality, bad string representation.
* UUIDv7 is better, time-based and more index locality. Still bad string representation.
* ulid is even better, time-based and lexicographically sortable, with good string representation.
* I like the idea of prefixed IDs, but still want compact representation.
* I wanted to learn about extending Postgres.

<!-- end_slide -->

# Background - plid

`usr_06DJX8T67BP71A4MYW9VXNR`
=

<!-- pause -->

<!-- incremental_lists: true -->

* 128 bits / 16 bytes.
* 48 bit / 6 byte timestamp (ms since UNIX epoch, until 10889 AD).
* 16 bit / 2 byte prefix:
* 64 bits / 8 bytes of randomness.
* Lexographically sortable.
* Double click to select the whole ID.

<!-- end_slide -->

# Background - Format

```
0                   1                   2                   3
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         prefix              |0|      timestamp first 16 bits  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     timestamp last 32 bits                    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      64 bits  of random data                  |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```


<!-- end_slide -->
# Rust

<!-- jump_to_middle -->
```rust {all|1}
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
struct Foo {
    bar: usize
}
```


<!-- end_slide -->

# Pgrx 

> pgrx is a framework for developing PostgreSQL extensions in Rust and strives to be as idiomatic and safe as possible.

<!-- end_slide -->
# Pgrx - Functions

```rust
use pgrx::prelude::*;

pgrx::pg_module_magic!(name, version);

#[pg_extern]
fn hello_world() -> &'static str {
    "Hello, world!"
}
```

<!-- pause -->

```
plid=# SELECT hello_world();
  hello_world
---------------
 Hello, world!
(1 row)
```

<!-- end_slide -->
# Pgrx - Types

```rust
use pgrx::prelude::*;
use serde::{Serialize, Deserialize};

pgrx::pg_module_magic!(name, version);

#[derive(Serialize, Deserialize, PostgresType)]
struct MyType {
    values: Vec<String>,
    thing: Option<Box<MyType>>
}
```

<!-- pause -->
```
plid=# SELECT '{ "values": ["a", "b", "c"], "thing": { "values": ["a"], "thing": null } }'::MyType;
                             mytype
----------------------------------------------------------------
 {"values":["a","b","c"],"thing":{"values":["a"],"thing":null}}
(1 row)
```



<!-- end_slide -->

# Pgrx - Testing

```rust
#[cfg(any(test, feature = "pg_test"))]
#[pg_schema]
mod tests {
    use pgrx::prelude::*;

    #[pg_test]
    fn test_hello_world() {
        let value: String = Spi::get_one("SELECT hello_world();")
            .expect("hello_world() to not fail")
            .expect("hello_world() to return a value");
        assert_eq!(value, "Hello, world!");
    }
}
```



<!-- end_slide -->

# Initial stab with Pgrx

```rust
use pgrx::prelude::*;
use serde::{Serialize, Deserialize};

pgrx::pg_module_magic!(name, version);

#[derive(
    Copy, Clone, PostgresType, PartialEq, Eq, Hash, PartialOrd, Ord, Serialize, Deserialize,
)]
#[pgvarlena_inoutfuncs]
// 16 bytes stored directly inline, no allocations required.
struct Plid(u128);

impl PgVarlenaInOutFuncs for Plid {
    // Omitted 
}
```

<!-- end_slide -->
# Initial stab with Pgrx


```
plid=# CREATE TABLE foo(id plid PRIMARY KEY);
CREATE TABLE
plid=# INSERT INTO foo(id) VALUES ('cba_06DHM1W511DKKJG500Z161R');
INSERT 0 1
plid=# SELECT * FROM foo;
             id
-----------------------------
 cba_06DHM1W511DKKJG500Z161R
(1 row)
```
<!-- end_slide -->


<!-- jump_to_middle -->
But what is actually stored on disk?
=====


<!-- end_slide -->

# Where even is it?


```sql
SELECT pg_relation_filepath('foo'), ctid FROM your_table;
```

<!-- pause -->

```
plid=# SELECT pg_relation_filepath('foo'), ctid FROM foo;
 pg_relation_filepath | ctid
----------------------+-------
 base/32768/32821     | (0,1)
(1 row)
```


<!-- end_slide -->

# What even is it


```bash {0-12|10-11}
❯ dd if=/Users/hugotunius/.pgrx/data-18/base/32768/32864 bs=8192 skip=0 count=1 | hexdump -C
1+0 records in
1+0 records out
8192 bytes transferred in 0.000033 secs (248242424 bytes/sec)
00000000  00 00 00 00 28 fa 2b 02  fa 51 00 00 1c 00 d0 1f  |....(.+..Q......|
00000010  00 20 04 20 00 00 00 00  d0 9f 52 00 00 00 00 00  |. . ......R.....|
00000020  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001fd0  0c 03 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001fe0  01 00 01 00 02 09 18 00  23 07 13 3e 00 05 ca 39  |........#..>...9|
00001ff0  5b 08 85 07 1a 9b 01 82  18 00 00 00 00 00 00 00  |[...............|
00002000
```

<!-- end_slide -->

# That sure looks like 16 bytes to me

```sql
SELECT pg_column_size(id) FROM foo LIMIT 1;
```

<!-- pause -->

```
plid=# SELECT pg_column_size(id) FROM foo;
 pg_column_size
----------------
             17
(1 row)
```

<!-- end_slide -->

# Looking closer


```plain {1-2|1-1}
23 
07 13 3e 00 05 ca 39 5b 08 85 07 1a 9b 01 82 18
```

<!-- pause -->

```
>>> (0x23 & 0xFE >> 1)
17
```

<!-- end_slide -->



<!-- jump_to_middle -->
Back to the drawing board
=====

<!-- end_slide -->

# Looking under the hood

```sql {1-9|6,9}
-- plid--0.0.0.sql

-- src/lib.rs:16
-- plid::Plid
CREATE TYPE Plid (
  INTERNALLENGTH = variable,
  INPUT = plid_in, /* plid::plid_in */
  OUTPUT = plid_out, /* plid::plid_out */
  STORAGE = extended
);
```

<!-- end_slide -->

# What we want

```sql {6,9}
-- plid--0.0.0.sql

-- src/lib.rs:16
-- plid::Plid
CREATE TYPE Plid (
  INTERNALLENGTH = 16,
  INPUT = plid_in, /* plid::plid_in */
  OUTPUT = plid_out, /* plid::plid_out */
  STORAGE = plain
);
```

<!-- end_slide -->

# Leaving the comfort of `#[derive(PostgresType)]`

```rust {1-13|8-12}
use pgrx::prelude::*;

::pgrx::pg_module_magic!(name, version);

#[derive(Debug, Copy, Clone, PartialEq, PartialOrd, Eq, Hash, Ord)]
pub struct Plid(u128);

extension_sql!(
    r#"CREATE TYPE plid;"#,
    name = "create_plid_shell_type",
    creates = [Type(Plid)]
);

```
<!-- end_slide -->

# Leaving the comfort of `#[derive(PostgresType)]`

```rust {1-24|1-8|10-24}
#[pg_extern(immutable, strict, parallel_safe)]
fn plid_in(input: &core::ffi::CStr) -> Plid {
    // Parse from string, omitted for brevity
}

#[pg_extern(immutable, strict, parallel_safe)]
fn plid_out(plid: Plid) -> &'static CStr {
    // Convert to string, omitted for brevity
}

extension_sql!(
    r#"
    CREATE TYPE plid (
        INTERNALLENGTH = 16,
        INPUT = plid_in,
        OUTPUT = plid_out
    );
"#,
    name = "create_plid_type",
    requires = [
        "create_plid_shell_type",
        plid_in,
        plid_out,
    ],
);
```

<!-- end_slide -->

# Can't do that here, mate

```
error[E0277]: the trait bound `Plid: ArgAbi<'_>` is not satisfied
   --> src/lib.rs:305:13
    |
304 | #[pg_extern(immutable, strict, parallel_safe)]
    | ---------------------------------------------- in this attribute macro expansion
305 | fn plid_out(plid: Plid) -> &'static CStr {
    |             ^^^^ unsatisfied trait bound
    |
help: the trait `ArgAbi<'_>` is not implemented for `Plid`
   --> src/lib.rs:78:1
    |
 78 | pub struct Plid(u128);
    | ^^^^^^^^^^^^^^^
    = help: the following other types implement trait `ArgAbi<'fcx>`:
              &'fcx T
              &'fcx [u8]
              &'fcx str
              *mut FunctionCallInfoBaseData
              AnyArray
              AnyElement
              AnyNumeric
              Array<'fcx, T>
            and 37 others
note: required by a bound in `pgrx::callconv::Args::<'a, 'fcx>::next_arg_unchecked`
   --> /Users/hugotunius/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/pgrx-0.16.1/src/callconv.rs:929:41
    |
929 |     pub unsafe fn next_arg_unchecked<T: ArgAbi<'fcx>>(&mut self) -> Option<T> {
    |                                         ^^^^^^^^^^^^ required by this bound in `Args::<'a, 'fcx>::next_arg_unchecked`
    = note: this error originates in the attribute macro `pg_extern` (in Nightly builds, run with -Z macro-backtrace for more info)
```

<!-- end_slide -->

# The `Datum` of it all

```c
/*
 * A Datum contains either a value of a pass-by-value type or a pointer to a
 * value of a pass-by-reference type.  Therefore, we must have
 * sizeof(Datum) >= sizeof(void *).  No current or foreseeable Postgres
 * platform has pointers wider than 8 bytes, and standardizing on Datum being
 * exactly 8 bytes has advantages in reducing cross-platform differences.
 *
 * The functions below and the analogous functions for other types should be used to
 * convert between a Datum and the appropriate C type.
 */

typedef uint64_t Datum;
```

<!-- end_slide -->

# ~~Admitting defeat~~ Allocating


```rust {1|4|9}
type BoxedPlid = PbBox<Plid>;

#[pg_extern(immutable, strict, parallel_safe)]
fn plid_in(input: &core::ffi::CStr) -> BoxedPlid {
    // Parse from string, omitted for brevity
}

#[pg_extern(immutable, strict, parallel_safe)]
fn plid_out(plid: BoxedPlid) -> &'static CStr {
    // Convert to string, omitted for brevity
}

```
<!-- end_slide -->

# So what is actually stored on disk?

<!-- incremental_lists: true -->

* Whatever your return from your `INPUT` function.
* In the `pgvarlena_inoutfuncs` case a varlena value.
* In the `[#derive(PostgresType, Serialize, Deserialize)]` case a CBOR encoded blob.
* If you return allocated data, whatever bytes your allocated data contains verbatim.

<!-- end_slide -->



# Staying aligned


```
LOG:  client backend (PID 12529) was terminated by signal 11: Segmentation fault: 11
```

<!-- end_slide -->

# Staying aligned


```rust {1,9}
pub struct Plid(u128);

extension_sql!(
    r#"
    CREATE TYPE plid (
        INTERNALLENGTH = 16,
        INPUT = plid_in,
        OUTPUT = plid_out,
        ALIGNMENT = 16 -- Not actually allowed
    );
"#,
    name = "create_plid_type",
    requires = [
        "create_plid_shell_type",
        plid_in,
        plid_out,
    ],
);
```
<!-- end_slide -->

# Even more (less?) alignment


```rust {1,9}
pub struct Plid([u8; 16]);

extension_sql!(
    r#"
    CREATE TYPE plid (
        INTERNALLENGTH = 16,
        INPUT = plid_in,
        OUTPUT = plid_out,
        ALIGNMENT = char
    );
"#,
    name = "create_plid_type",
    requires = [
        "create_plid_shell_type",
        plid_in,
        plid_out,
    ],
);
```

<!-- end_slide -->

# The worst id in the world

```
plid=# CREATE TABLE foo (id plid PRIMARY KEY);
ERROR:  data type plid has no default operator class for access method "btree"
HINT:  You must specify an operator class for the index or define a default operator class for the data type.
```

<!-- end_slide -->

# Operators and operator classes

```rust
extension_sql!(
    r#"
    CREATE OPERATOR CLASS plid_btree_ops
    DEFAULT FOR TYPE plid USING btree AS
        OPERATOR 1 < ,
        OPERATOR 2 <= ,
        OPERATOR 3 = ,
        OPERATOR 4 >= ,
        OPERATOR 5 > ,
        FUNCTION 1 btree_plid_cmp(plid, plid);
"#,
    name = "create_plid_btree_ops",
    requires = [
        "create_plid_type",
        "create_plid_lt_operator",
        "create_plid_le_operator",
        "create_plid_eq_operator",
        "create_plid_ge_operator",
        "create_plid_gt_operator",
        btree_plid_cmp
    ]
);
```
<!-- end_slide -->

# Operators

```rust
#[pg_extern(immutable, strict, parallel_safe)]
fn plid_lt(plid1: BoxedPlid, plid2: BoxedPlid) -> bool {
    plid1.0 < plid2.0
}

extension_sql!(
    r#"
    CREATE OPERATOR <  (
        LEFTARG = plid,
        RIGHTARG = plid,
        PROCEDURE = plid_lt,
        COMMUTATOR = >,
        NEGATOR = >=,
        RESTRICT = scalarltsel,
        JOIN = scalarltjoinsel
    );
"#,
    name = "create_plid_lt_operator",
    requires = ["create_plid_type", plid_lt]
);
```

<!-- end_slide -->

# The best id in the world

```
plid=# CREATE TABLE foo (id plid PRIMARY KEY);
CREATE TABLE
```

<!-- end_slide -->

# But is it fast?


```text {1-25|11, 24}
plid=# CREATE TABLE users (id plid PRIMARY KEY DEFAULT gen_plid_monotonic('usr'));
CREATE TABLE
plid=# EXPLAIN ANALYZE INSERT INTO users DEFAULT VALUES FROM generate_series(1, 1000000);
                                                                 QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------------
 Insert on test_plid  (cost=0.00..12500.00 rows=0 width=0) (actual time=14373.450..14373.455 rows=0.00 loops=1)
   Buffers: shared hit=2108066 read=1 dirtied=9259 written=9262, temp read=1709 written=1709
   ->  Function Scan on generate_series  (cost=0.00..12500.00 rows=1000000 width=16) (actual time=99.407..12392.568 rows=1000000.00 loops=1)
         Buffers: temp read=1709 written=1709
 Planning Time: 0.441 ms
 Execution Time: 14375.142 ms
(6 rows)

plid=# CREATE TABLE test (key uuid PRIMARY KEY);
CREATE TABLE
plid=# EXPLAIN ANALYZE INSERT INTO test SELECT uuidv7() FROM generate_series(1, 1000000);
                                                                 QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------------
 Insert on test  (cost=0.00..12500.00 rows=0 width=0) (actual time=13877.219..13877.219 rows=0.00 loops=1)
   Buffers: shared hit=2108066 read=1 dirtied=9259 written=9780, temp read=1709 written=1709
   ->  Function Scan on generate_series  (cost=0.00..12500.00 rows=1000000 width=16) (actual time=92.268..11956.284 rows=1000000.00 loops=1)
         Buffers: temp read=1709 written=1709
 Planning Time: 0.040 ms
 Execution Time: 13878.977 ms
(6 rows)
```

<!-- end_slide -->

<!-- jump_to_middle -->
Demo
==

<!-- end_slide -->
# Fin


* James Blackwood-Sewell - Fearless Extension Development With Rust and PGRX **https://www.youtube.com/watch?v=C9TopAI1Hnk**
* pgrx @ Github - **https://github.com/pgcentralfoundation/pgrx**
* The Rust Programming Language - **https://doc.rust-lang.org/book/**
* plid - **https://github.com/k0nserv/plid**

