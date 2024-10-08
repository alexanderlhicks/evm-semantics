Byte-Buffer Abstraction
=======================

```k
requires "evm.md"

module BUF-SYNTAX
    imports EVM
```

Both `#bufStrict(SIZE, DATA)` and `#buf(SIZE, DATA)` represents a symbolic byte array of length `SIZE` bytes, left-padded with zeroes.
Version `#bufStrict` is partial and only defined when `DATA` is in the range given by `SIZE`.
It rewrites to `#buf` when data is in range, and is expected to immediately evaluate into `#buf` in all contexts.
Version `#buf` is total and artificially defined `modulo 2 ^Int (8 *Int SIZE)`.
This division is required to facilitate symbolic reasoning in Haskell backend, because Haskell has limitations
when dealing with partial functions.

**Usage:** All symbolic byte arrays must be originally created as `#bufStrict`.
This ensures `#buf` is never present in out of range mode.
For this, definition rule RHS should always use `#bufStrict` when array is first created, but may use `#buf` when array
is just carried on from LHS without changes. Definition rule LHS should only use `#buf`.
Claims should always use `#bufStrict` in LHS and `#buf` in RHS.

```k
    syntax Bytes ::= #bufStrict ( Int , Int ) [symbol(#bufStrict), function]
    syntax Bytes ::= #buf ( Int , Int ) [symbol(buf), function, total, smtlib(buf)]

    syntax Int ::= #ceil32 ( Int ) [symbol(#ceil32), macro]
 // -------------------------------------------------------
    rule #ceil32(N) => notMaxUInt5 &Int ( N +Int maxUInt5 )

endmodule

module BUF
    imports BUF-SYNTAX

    syntax Int ::= #powByteLen ( Int ) [symbol(#powByteLen), function, no-evaluators]
 // ---------------------------------------------------------------------------------
 // rule #powByteLen(SIZE) => 2 ^Int (8 *Int SIZE)
    rule 2 ^Int (8 *Int SIZE) => #powByteLen(SIZE) [symbolic(SIZE), simplification]

    rule 0    <Int #powByteLen(SIZE) => true requires 0 <=Int SIZE [simplification, preserves-definedness]
    rule SIZE <Int #powByteLen(SIZE) => true requires 0 <=Int SIZE [simplification, preserves-definedness]
    rule #write(WM, IDX, VAL) => WM [ IDX := #buf(1, VAL) ] [simplification]

    rule #bufStrict(SIZE, DATA) => #buf(SIZE, DATA)
      requires #range(0 <= DATA < (2 ^Int (8 *Int SIZE)))

    rule #buf(SIZE, DATA) => #padToWidth(SIZE, #asByteStack(DATA %Int (2 ^Int (8 *Int SIZE))))
      requires 0 <Int SIZE
      [concrete]
    rule #buf(_SIZE, _) => .Bytes [owise, concrete] // SIZE <= 0

endmodule
```
