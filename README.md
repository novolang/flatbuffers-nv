# flatbuffers-nv

FlatBuffers is a serialization format that is read *in place*: the bytes on the
wire are the bytes a program reads from, with no parsing and no unpacking step.
It is specified by
[the FlatBuffers internals page](https://flatbuffers.dev/internals/) and
implemented by [the flatbuffers project](https://github.com/google/flatbuffers).
This package reads and writes it in novo-lang, and performs no input or output
itself.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it is
implemented. Version 0.1.0 will be the first working release.

## What FlatBuffers is

A **buffer** begins with a four-byte offset to its root table, and may carry a
four-byte *file identifier* after it. Every value in it is little-endian, on
every machine, and every value sits at a multiple of its own size.

A **table** does not contain its own layout. It begins with a signed 32-bit
offset that is *subtracted* from the table's position to find a **vtable**,
which is a list of 16-bit offsets — one per declared field — saying where each
field sits inside the table. A vtable entry of zero means the field is absent,
and the reader uses the schema's default.

Three things follow from that, and they are the format.

- **An optional field costs nothing.** A table with two of forty fields set is
  a vtable and two values.
- **Adding a field is compatible in both directions.** A new field gets a new
  slot at the end. An old reader's idea of the vtable is shorter than the
  writer's, so it reads the new field as absent and uses its default; a new
  reader meeting old data does the same. The one rule is that slots are never
  reordered and never reused.
- **Reading a field is one indirection.** Field, vtable, offset, value. Nothing
  is copied and nothing is parsed, so a gigabyte buffer costs nothing to open
  and a field nobody reads costs nothing at all.

A **struct** is the opposite kind of thing. It is stored *inline* in its table,
has a fixed size, no vtable and no optional fields, and may hold only scalars
and other structs. It costs nothing to read and it can never grow: its size is
part of every table and every vector that contains it. A table is the thing
that evolves; a struct is the thing that does not.

A **vector** is a 32-bit length followed by its elements. A **string** is a
vector of bytes with a zero after it, which is not counted in the length. A
**union** is two vtable slots: a `u8` naming which member, and an offset to
that member's table.

Because a field read is an offset dereference into bytes somebody else sent,
the format comes with a **verifier**: a pass that walks every reachable offset
once and checks that it lands inside the buffer, is aligned, and describes a
well-formed table, vector or string. A caller verifies once and then reads
without checking.

| Value | Size |
| --- | --- |
| Root offset | 4 bytes, at position 0 |
| File identifier | 4 bytes, at position 4, optional |
| uoffset (forward reference) | 4 bytes, unsigned |
| soffset (table to vtable) | 4 bytes, signed, subtracted |
| voffset (vtable entry) | 2 bytes |
| Vtable header | 4 bytes: its own size, then the table's |
| Largest table | 65,535 bytes, because a vtable entry is 16-bit |
| Vector length prefix | 4 bytes |
| String terminator | 1 byte, not counted in the length |
| Union | 2 vtable slots; type 0 is always `NONE` |

## Install

```
novo pkg add flatbuffers-nv
```

## Example

```novo
use std.bytes
use fbbuf
use fbtable
use fbverify

fn main() [io]
    // Verify once, then read without checking.
    match fbverify.verify(fbbuf.buffer(bytes.zeros(16)),
                          fbverify.default_budget())
        Err(f) => println(f.message())
        Ok(b)  =>
            match fbtable.root_table(b)
                Err(f) => println(f.message())
                // The default comes from the schema, because a field
                // equal to its default is not in the buffer at all.
                Ok(t)  => println("${fbtable.field_u32(t, fbtable.FB_FIRST_SLOT, 0)}")
```

Build and test with:

```
novo pkg build                  # type- and effect-check the package
novo test --isolate tests/fbread_tests.nv
```

Today `novo test` fails on purpose: every assertion reaches
`not implemented: flatbuffers-nv.<module>.<fn>`.

## What the package contains

| Module | Contents |
| --- | --- |
| `fbbuf` | The buffer: the root offset, the file identifier, little-endian scalars, and the alignment rules. |
| `fbtable` | The vtable and the table: slot lookup, the scalar accessors with their defaults, and the checked pair. |
| `fbvector` | Vectors, strings and the struct layout rules. |
| `fbunion` | Unions as the two vtable slots they are, and enums at the width the schema declares. |
| `fbverify` | The verifier, and the budgets that keep the verifier itself bounded. |
| `fbbuild` | The builder: back to front, with alignment and vtable deduplication. |
| `fbschema` | The `.fbs` schema language, read into a value. No code generation. |
| `fbaccess` | Reading a buffer by field *name*, from a schema value, at run time. |
| `fberror` | `FbFault` with the offset and the vtable slot it happened at, and the four questions `is_malformed`, `is_limit`, `is_schema_error` and `is_unsupported`. |

No function in this package opens a file, reads a clock or waits for anything,
and none of them generates source code.

## How to choose an entry point

**You have a buffer from somewhere you do not control: `fbverify.verify`
first.** Then read with `fbtable`'s accessors, which assume the offsets are
sound.

**You will read a handful of fields and would rather not verify:
`fbtable.checked_u32` and its siblings.** They bounds-check every step and
answer a `Result`.

**You know the schema at compile time and want speed:
`fbschema.slot_of` once, then `fbtable`'s slot accessors.** Resolving a name is
a scan; resolving it once and reusing the slot is what generated code does.

**The schema arrives at run time: `fbaccess`.** `document`, then `field` by
name, or `fields` for everything, or `to_json` for a dump. This is what a
generator cannot do.

**You are writing a buffer: `fbbuild`.** Read "The builder writes backwards"
first.

**You are on a fixed memory budget: `fbbuild.builder_into`.** It fills a buffer
you sized and answers `FbBufferTooSmall` rather than growing.

## The rules a user needs

1. **Reading in place means every field read is an offset dereference into
   bytes you did not write.** The format has no length on a table, no
   terminator, no checksum and no tag to notice a wrong offset with. That is
   why `fbverify` exists and why `FbBuf` carries a `verified` flag: it is a fact
   about this program's history, not about the bytes.
2. **The verifier needs its own budgets.** Two tables may legally share a
   vtable and two fields may legally point at the same subtree, so the reachable
   graph is a DAG and a small buffer can describe an enormous traversal.
   `FbVerifierBudgetReached` is a policy meeting a file, and
   `fberror.is_malformed` answers false for it. `fbverify.measure` is how to
   size the budgets from real messages.
3. **Every scalar accessor takes its default, and the default comes from the
   schema.** A field equal to its default is not written into the buffer at all,
   so the only place it exists is the `.fbs`. A reader that passed zero where
   the schema said `-1` reads the wrong value and nothing says so.
4. **A vtable slot is not a field index.** The first declared field is slot 4,
   and slots step by two. `fbtable.slot_of_field` is the conversion.
5. **Slots are never reordered and never reused.** A new field goes at the end;
   a removed one is marked `(deprecated)` and keeps its slot forever. The `id:`
   attribute pins them explicitly, and `fbschema.check` refuses a schema that
   breaks the rule — because an old buffer read through a reordered schema
   produces values of the wrong fields, silently.
6. **A union is two slots, and type 0 is always `NONE`.** Every union's enum has
   an implicit `NONE = 0` the schema does not write, so a reader that treated 0
   as the first declared member is wrong on every union. The format does not
   enforce that the two slots agree, and `FbUnionHalfPresent` is that case.
7. **The file identifier is the only thing that says what kind of message a
   buffer is, and it is optional.** A reader that skips it will read a
   `Monster` as a `Weapon` and get plausible nonsense.
   `fbbuf.check_identifier` and `fbaccess.document` are where it is checked, and
   a wrong one is not a malformed buffer.
8. **An old reader meeting a new writer's table is not an error.** A slot past
   the end of the vtable the reader knows about reads as absent, which is the
   whole of the format's forward compatibility.
9. **A struct's padding is invisible in the schema and real in the bytes.**
   `struct Vec3 { x:float; y:float; z:float; }` is twelve bytes;
   `struct Mixed { a:byte; b:double; }` is sixteen.
   `fbvector.struct_layout` is the rule.
10. **A string is not checked for UTF-8 by the ordinary accessor.** The format
    says a string is UTF-8 and does not say a reader must verify; checking per
    access would cost the format its reason for existing.
    `fbvector.utf8_string` is the checking one, for a caller that calls it once.
11. **An enum's width is the schema's and the buffer does not say.**
    `enum Colour : byte` is one byte and `enum Big : long` is eight, so an enum
    field cannot be read without the schema. `fbunion.field_enum` takes the
    size.
12. **Every public type, variant and module name in this package starts `Fb`
    or `fb`.** Type and variant names are unique across a whole program,
    dependencies included, so two packages that both declared `Table` could not
    be used together. `Table`, `Vector`, `Union`, `Schema` and `Builder` are all
    names another package will want, so this one takes none of them.

## The builder writes backwards

A uoffset is always *forward* from its own position, so a thing must exist
before anything can point at it. The only way to write a parent after its
children while keeping the parent earlier in the buffer is to fill the buffer
from the end towards the start. The root offset, written last, lands at
position 0.

So the call order is inside out, and no API hides it. To build a table with a
string field:

```novo ignore
let s = fbbuild.write_string(b, "hello")!      // the string first
let t = fbbuild.start_table(s.builder)!
let t2 = fbbuild.set_offset(t, slot, s.offset)!
let done = fbbuild.end_table(t2)!
let out = fbbuild.finish(done.builder, done.offset, "MONS")!
```

Writing the string between `start_table` and `end_table` would interleave two
objects in one region, and that is what `FbBuilderOutOfOrder` names. **A struct
is the exception**: it is written inline, inside its table, so
`fbbuild.set_struct` does belong between the two.

Vtables are deduplicated: a builder writing ten thousand tables of the same
shape writes one vtable and ten thousand offsets to it, which is most of what
makes the format small for repeated records. `fbbuild.vtables_written` is what
says how much it saved, and a verifier must not assume two tables have
different vtables.

## No code generation

The reference tooling for this format is `flatc`, a compiler that reads a
`.fbs` and writes accessor classes in one of a dozen languages. This package
does not do that. A schema here is a value at run time, and `fbaccess` reads a
buffer by field name from it.

Three things follow, and none of them is possible with generated code: a schema
that arrives at run time can be used, which is what a dumper, a converter or a
schema registry is made of; a buffer can be read through a schema a caller
chose from several; and this is a library rather than a build step. A generator
can be written on top of `fbschema`, and that is a program.

The price is a lookup per field, and `fbaccess.field` says so: resolving a name
to a slot is a scan of the table's fields. A caller reading one field of a
million records resolves once with `fbschema.slot_of` and uses `fbtable`'s slot
accessors.

## What is not included

- **FlexBuffers.** A second, schemaless format that shares a name with this one
  and nothing else: a self-describing value written back to front with a type
  byte per value, its own reading rules, its own builder and no vtables at all.
  Putting it here would be putting two formats in one package.
  `FbSchemaUnsupported("flexbuffers")` names it, and a `flexbuffers-nv` row is
  where it goes.
- **Code generation.** See above.
- **`include` in a `.fbs`.** An include is a file read, this package is `core`,
  and a caller that has several files concatenates them or resolves the
  includes itself. A schema with one is `FbSchemaUnsupported("include")`, which
  says so rather than silently reading a schema with a type missing.
- **`rpc_service` declarations.** Legal `.fbs`, and an RPC surface rather than a
  format.
- **The `native_*` and other code-generation attributes.** They configure a
  generator this package does not have.
- **Sorted vectors and `(key)` lookup by binary search.** The attribute is
  parsed and carried; sorting on write and searching on read is a 0.2.0 row.
- **A build for a microcontroller.** The reading half would fit, and
  `fbbuild.builder_into` is the shape a device needs. The `.fbs` parser would
  not, and a package makes one claim for all of its modules. A
  `flatbuffers-core-nv` split covering `fbbuf`, `fbtable`, `fbvector` and
  `fbunion` is the row to file if a device lane wants one.

## Related packages

- [protobuf-nv](https://novo-lang.org/packages/protobuf-nv) is the other
  schema-first binary encoding. Its wire format carries a tag per present field
  and is parsed into objects; this one carries a vtable per table shape and is
  read in place. Protobuf is smaller on the wire for sparse messages;
  FlatBuffers is free to open.
- [avro-nv](https://novo-lang.org/packages/avro-nv) carries no markers at all
  and resolves two schemas against each other instead. It is the third point of
  the same triangle.
- [capnproto-nv](https://novo-lang.org/packages/capnproto-nv) is the closest
  neighbour: also read in place, with pointers instead of vtables.

## The reference implementation

[flatbuffers](https://github.com/google/flatbuffers), with
[the internals page](https://flatbuffers.dev/internals/) as the format
document. This package keeps the reference's own names where they are good —
vtable, uoffset, soffset, voffset, the file identifier — so a reader with that
page open recognises what they are.

## Test vectors

The flatbuffers project's `tests/` directory is the oracle: `monster_test.fbs`
with its generated `monsterdata_test.mon`, the `.bfbs` reflection schemas, and
a set of deliberately corrupted buffers the reference verifier must refuse.
When the bodies land, `monsterdata_test.mon` is read field by field and
compared against the values the project's own tests assert, a buffer this
package builds is verified by the reference verifier, and every corrupted
buffer is refused — as a generated run beside the two suites in `tests/`.

## Implementation status

Every function is a `todo()`. Every type is declared.

| Module | Implemented |
| --- | --- |
| `fbbuf` — `FbBuf` and its fifteen functions | no |
| `fbtable` — `FbTable` and its seventeen functions | no |
| `fbvector` — `FbVector`, `FbStructLayout` and their fifteen functions | no |
| `fbunion` — `FbUnion`, `FbUnionVector` and their seven functions | no |
| `fbverify` — `FbBudget`, `FbCost` and their ten functions | no |
| `fbbuild` — `FbBuilder`, `FbWritten` and their eighteen functions | no |
| `fbschema` — `FbBaseType`, `FbField`, `FbObject`, `FbEnum`, `FbSchema` and their thirteen functions | no |
| `fbaccess` — `FbValue`, `FbDocument` and their nine functions | no |
| `fberror` — `FbWhere`, `FbFaultKind`, `FbFault` and their eight functions | no |

## Licence

Apache-2.0. See `LICENSE`.
