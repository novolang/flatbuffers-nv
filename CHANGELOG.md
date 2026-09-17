# Changelog

All notable changes to flatbuffers-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `fbtable` — the vtable, which is the format. A table carries a
  SIGNED offset that is SUBTRACTED to find a vtable of 16-bit field
  offsets, and an entry of zero means absent. That is what makes an
  optional field free, makes adding a field compatible in both
  directions, and makes a field read one indirection. Two accessor
  surfaces are published, not one: `field_u32` for a buffer somebody
  verified and `checked_u32` for one nobody did, because verifying once
  and then reading a million fields is the shape this format is for and
  making every read pay for the verification would take that away.
- `fbverify` — the other half of reading in place. Every field read is
  an offset dereference into bytes the program did not write, and the
  format has no length, no terminator and no checksum to notice a wrong
  offset with. The verifier carries its OWN budgets, because two tables
  may legally share a vtable and two fields may legally point at the
  same subtree — the reachable graph is a DAG and a small buffer can
  describe an enormous traversal. `FbBuf.verified` records that
  somebody did it; `trust_unverified` is the escape hatch, named to be
  seen.
- `fbbuild` — the builder, back to front, because a uoffset points
  forward and a thing must exist before anything can point at it. The
  call order is therefore inside out and no API hides it;
  `FbBuilderOutOfOrder` is what says so. A struct is the one exception,
  written inline between `start_table` and `end_table`. Vtables are
  deduplicated, which is most of what makes the format small for
  repeated records.
- `fbschema` — **the decision this row asked for: the `.fbs` schema is
  read into a VALUE and no code is generated.** `flatc` is a code
  generator; this is not one. Three things follow that generated code
  cannot do — a schema that arrives at run time, a buffer read through
  a schema chosen from several, and a library rather than a build step
  — and a generator can be written on top of this.
- `fbaccess` — the module the package exists for: a field by NAME, from
  a schema value, at run time. The price is a lookup per field and the
  doc says so rather than pretending the convenience is free.
- `fbbuf`, `fbvector`, `fbunion` — the buffer, the vectors and strings,
  the struct layout rules, and unions as the two vtable slots they
  really are. `FB_UNION_NONE` is zero, which a reader that treated as
  the first declared member gets wrong on every union.
- `fberror` — nineteen fault kinds and FOUR predicates: `is_malformed`
  for the bytes, `is_limit` for a bound this program set,
  `is_schema_error` for a developer's problem, and `is_unsupported` for
  legal `.fbs` this package does not implement.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the two API suites reaches `not implemented:
  flatbuffers-nv.<module>.<fn>`.
- **No device claim, and not because the format would not fit.** The
  reading half would, and `fbbuild.builder_into` is the shape a device
  needs. The `.fbs` parser would not, and a package makes one claim for
  all of its modules. A `flatbuffers-core-nv` split covering `fbbuf`,
  `fbtable`, `fbvector` and `fbunion` is the row to file if a device
  lane wants one; the manifest says so.
- **FlexBuffers is not here**, and is named rather than left out
  quietly: `FbSchemaUnsupported("flexbuffers")`. It shares a name with
  this format and nothing else.
- **Sorted vectors and `(key)` lookup** are parsed and carried but not
  implemented; the README lists them as a 0.2.0 row.
