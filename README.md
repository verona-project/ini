# Verona INI

A literal UTF-8 INI entry parser and writer written in Verona, using Base and
Foundation. Parsing allocates no memory.

## Build and test

From this package directory, using the repository toolchain:

```sh
../../build/verona build ini ../../build/formats
../../build/verona build ini-tests ../../build/formats
../../build/formats/ini-tests
```

Consumers add `src`, `../base/src`, and `../foundation/src` to their module
paths and import `ini`. The static archive is a build artifact; Verona
consumers use source modules. Tests also use `../testing/src`.

## Shared contracts

- `result` contains `status: error:status` and `offset: u64`. Success is `(ok)`
  with offset equal to the input byte length. Syntax errors are `(invalid-input)`
  with a zero-based byte offset (EOF may equal length). Invalid UTF-8 is detected
  before parsing and reports offset zero; INI's NUL precheck also reports zero.
- Input is a borrowed `data:view`, not a NUL-terminated string. The caller keeps
  its bytes alive and unchanged throughout parsing and callbacks.
- `visitor` carries an opaque `(pointer void)` context and a typed callback.
  There is no global parser state. Callbacks return `unit`; cancellation is not
  supported. Events already delivered are not rolled back on a later error.
- Writers append to an initialized `data:data`, using its chosen allocator.
  They validate input and reserve a checked worst-case capacity before appending,
  so validation, overflow, or allocation failure preserves existing output.
  Source bytes must not alias output storage. Check every returned status.
- No implicit file I/O, document ownership, or reference counting. Arena reset
  or destruction invalidates all associated buffers. The algorithms use
  recursion; unbounded input lengths are not a stack-safety guarantee.

## API and dialect

| Export | Contract |
| --- | --- |
| `visitor(context, emit)` | `emit(context, section, key, value) -> unit`; all text arguments are `data:view` |
| `parse(source, visitor)` | Emit entries in source order; returns `result` |
| `write-section(name, out)` | Append `[name]` and LF; returns `error:status` |
| `write-entry(key, value, out)` | Append `key=value` and LF; returns `error:status` |

LF, CRLF, and CR delimit lines. ASCII spaces/tabs are trimmed around lines,
section names, keys, and values. `;` and `#` start comments only at the beginning
of a trimmed line. Entries split at the first `=`; the remaining value is literal,
including quotes, `=`, `#`, `;`, and backslashes. Empty values are valid.
Keys and section names must be nonempty. Brackets inside section names and NUL
anywhere are rejected. Headers must end at `]` after trimming; inline header
comments are not accepted. Quoting, escape decoding, interpolation, continuation
lines, BOM handling, and alternate separators are outside this dialect.

Entries before a section have an empty section view. Repeated sections and keys
are delivered in order; callers choose duplicate handling. Empty sections emit
no entry. All callback views borrow the input and remain valid for its lifetime.

Writers require already-trimmed UTF-8 without NUL or line breaks. Section names
cannot contain brackets; keys cannot contain `=` or begin with `[`, `#`, or `;`.
These restrictions preserve entry meaning when parsed again. Values may be empty.
Writers normalize spacing and line endings; comments are not retained.

## Example

```lisp
(import base)
(import ini)
(import data)

(%function on-entry
  ((context (pointer void)) (section data:view) (key data:view) (value data:view)) unit
  unit)

(%function inspect-ini () ini:result
  (ini:parse (data:view "[app]\nname=verona\n" 18)
    (ini:visitor (cast (pointer void) "") on-entry)))
```

Tests cover comments, trimming, empty values, duplicates, literal delimiters,
Unicode, line endings, syntax errors, error offsets, arena output, and failed
allocation without output mutation.
