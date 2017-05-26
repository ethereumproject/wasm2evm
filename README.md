# wasm2evm
A WebAssembly to EVM compiler

Right now we aim to compile the hello world program below from WASM to
EVM.

```
(module
  (memory 1 
   (segment 8 "hello world!\n")
  )
  (import $__fwrite "env" "_fwrite" (param i32 i32 i32 i32) (result i32))
  (import $_get__stdout "env" "get__stdout" (param) (result i32))
  (export "main" $main)

  (func $main (result i32)
    (local $stdout i32)
    (set_local $stdout (call_import $_get__stdout))

    (return (call_import $__fwrite
       (i32.const 8)         ;; void *ptr    => Address of our string
       (i32.const 1)         ;; size_t size  => Data size
       (i32.const 13)        ;; size_t nmemb => Length of our string
       (get_local $stdout))  ;; stream
    )
  )
)
```

## Integer Size

The compiled version will only utilize at most 64 bit of integers
except for hash values and addresses. This is under the assumption
that most integers never exceed `u64::max_value()`.

This could lead to a small increase in gas cost. But under the above
assumption, using a well-optimized EVM interpreter, it does not
necessily lead to slower code.

## Initialization

First, we use the `PUSHX` opcodes and `MSTORE` opcode to initialize
the memory as specified by WASM. This writes `"hello world!\n"` from
memory location 8.

## Table

WebAssembly's table is probamatic. Currently, it is only used to
implement indirect function calls in C/C++ using an integer index as
the pointer-to-function and the table to hold the array of
indirectly-callable functions. As a result, it can be ignored as of
now.

In the future, table is mostly dealt with GC. This is not existant in
the Ethereum world. So it can also be ignored.

## Imports and Exports

Exports are ignored (or they can be used in ABI in the
future). Imports other than `"env"` is not allowed right now.

In the future, imports from other modules will be compacted to one
single module.

## Unwrap `$main`

Code in main is unwrapped directly below initialization. Other
functions are added below, with a `JUMPDEST` opcode. Those functions
will need to manage their own call stack and return values. Local
variables are set to the memory, with the exception of external
objects like this returned by `$_get__stdout`. This is analyzed away.

When dealing with nested function calls. The return values are dealt
by stack. `__fwrite` call serves two functions -- when working with
stdout, it uses `LOG` opcode; when working with other things, it uses
a filesystem library built on top of the contract storage.

The hello world example will be directly compiled to a `LOG` opcode
and a `RETURN` opcode (with several `PUSH` opcodes to write the
parameters, probably).
