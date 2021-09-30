# Development backend for WebAssembly

## Plan

- Initial bringup
  - [x] Get a wasm backend working for some of the number tests.
  - [x] Use a separate `gen_wasm` directory for now, to avoid trying to do bringup and integration at the same time.
- Get the fundamentals working

  - [x] Come up with a way to do control flow
  - [x] Flesh out the details of value representations between local variables and stack memory
  - [x] Set up a way to write tests with any return value rather than just i64 and f64
  - [x] Implement stack memory
    - [x] Push and pop stack frames
    - [x] Deal with returning structs
    - [x] Distinguish which variables go in locals, own stack frame, caller stack frame, etc.
    - [ ] Ensure early Return statements don't skip stack cleanup
  - [ ] Vendor-in parity_wasm library so that we can use `bumpalo::Vec`
  - [ ] Implement relocations
    - Requires knowing the _byte_ offset of each call site. This is awkward as the backend builds a `Vec<Instruction>` rather than a `Vec<u8>`. It may be worth serialising each instruction as it is inserted.

- Refactor for code sharing with CPU backends

  - [ ] Implement a `scan_ast` pre-pass like `Backend` does, but for reusing Wasm locals rather than CPU registers
  - [ ] Extract a trait from `WasmBackend` that looks as similar as possible to `Backend`, to prepare for code sharing
  - [ ] Refactor to actually share code between `WasmBackend` and `Backend` if it seems feasible

- Integration
  - Move wasm files to `gen_dev/src/wasm`
  - Share tests between wasm and x64, with some way of saying which tests work on which backends, and dispatching to different eval helpers based on that.
  - Get `build_module` in object_builder.rs to dispatch to the wasm generator (adding some Wasm options to the `Triple` struct)
  - Get `build_module` to write to a file, or maybe return `Vec<u8>`, instead of returning an Object structure

## Structured control flow

One of the security features of WebAssembly is that it does not allow unrestricted "jumps" to anywhere you like. It does not have an instruction for that. All of the [control instructions][control-inst] can only implement "structured" control flow, and have names like `if`, `loop`, `block` that you'd normally associate with high-level languages. There are branch (`br`) instructions that can jump to labelled blocks within the same function, but the blocks have to be nested in sensible ways.

[control-inst]: https://webassembly.github.io/spec/core/syntax/instructions.html#control-instructions

This way of representing control flow is similar to parts of the Roc AST like `When`, `If` and `LetRec`. But Mono IR converts this to jumps and join points, which are more of a Control Flow Graph than a tree. We need to map back from graph to a tree again in the Wasm backend.

Our solution is to wrap all joinpoint/jump graphs in an outer `loop`, with nested `block`s inside it.

### Possible future optimisations

There are other algorithms available that may result in more optimised control flow. We are not focusing on that for our development backend, but here are some notes for future reference.

The WebAssembly compiler toolkit `binaryen` has an [API for control-flow graphs][cfg-api]. We're not using `binaryen` right now. It's a C++ library, though it does have a (very thin and somewhat hard-to-use) [Rust wrapper][binaryen-rs]. Binaryen's control-flow graph API implements the "Relooper" algorithm developed by the Emscripten project and described in [this paper](https://github.com/emscripten-core/emscripten/blob/main/docs/paper.pdf).

> By the way, apparently "binaryen" rhymes with "Targaryen", the family name from the "Game of Thrones" TV series

There is also an improvement on Relooper called ["Stackifier"](https://medium.com/leaningtech/solving-the-structured-control-flow-problem-once-and-for-all-5123117b1ee2). It can reorder the joinpoints and jumps to make code more efficient. (It is also has things Roc wouldn't need but C++ does, like support for "irreducible" graphs that include `goto`).

[cfg-api]: https://github.com/WebAssembly/binaryen/wiki/Compiling-to-WebAssembly-with-Binaryen#cfg-api
[binaryen-rs]: https://crates.io/crates/binaryen

## Stack machine vs register machine

Wasm's instruction set is based on a stack-machine VM. Whereas CPU instructions have named registers that they operate on, Wasm has no named registers at all. The instructions don't contain register names. Instructions can oly operate on whatever data is at the top of the stack.

For example the instruction `i64.add` takes two operands. It pops the top two arguments off the VM stack and pushes the result back.

In the [spec][spec-instructions], every instruction has a type signature! This is not something you would see for CPU instructions. The type signature for i64.add is `[i64 i64] → [i64]` because it pushes two i64's and pops an i64.

[spec-instructions]: https://webassembly.github.io/spec/core/appendix/index-instructions.html

This means that WebAssembly has a concept of type checking. When you load a .wasm file as a chunk of bytes into a Wasm runtime (like a browser or [wasmer](https://wasmer.io/)), the runtime will first _validate_ those bytes. They have some fast way of checking whether the types being pushed and popped are consistent. So if you try to do the i64.add instruction when you have floats on the stack, it will fail validation.

Note that the instruction makes no mention of any source or destination registers, because there is no such thing. It just pops two values and pushes one. (This architecture choice helps to keep WebAssembly programs quite compact. There are no extra bytes specifying source and destination registers.)

Implications of the stack machine for Roc:

- There is no such thing as register allocation, since there are no registers! There is no reason to maintain hashmaps of what registers are free or not. And there is no need to do a pass over the IR to find the "last seen" occurrence of a symbol in the IR. That means we don't need the `Backend` methods `scan_ast`, `scan_ast_call`, `set_last_seen`, `last_seen_map`, `free_map`, `free_symbols`, `free_symbol`, `set_free_map`.

- There is no random access to the stack. All instructions operate on the data at the _top_ of the stack. There is no instruction that says "get the value at index 17 in the stack". If such an instruction did exist, it wouldn't be a stack machine. And there is no way to "free up some of the slots in the stack". You have to consume the stuff at the top, then the stuff further down. However Wasm has a concept of local variables, which do allow random access. See below.

## Local variables

WebAssembly functions can have any number of local variables. They are declared at the beginning of the function, along with their types (just like C). WebAssembly has 4 value types: `i32`, `i64`, `f32`, `f64`.

In this backend, each symbol in the Mono IR gets one WebAssembly local. To illustrate, let's translate a simple Roc example to WebAssembly text format.
The WebAssembly code below is completely unoptimised and uses far more locals than necessary. But that does help to illustrate the concept of locals.

```
app "test" provides [ main ] to "./platform"

main =
    1 + 2 + 4
```

The Mono IR contains two functions, `Num.add` and `main`, so we generate two corresponding WebAssembly functions.

```
  (func (;0;) (param i64 i64) (result i64)   ; declare function index 0 (Num.add) with two i64 parameters and an i64 result
    local.get 0              ; load param 0                                    stack=[param0]
    local.get 1              ; load param 1                                    stack=[param0, param1]
    i64.add                  ; pop two values, add, and push result            stack=[param0 + param1]
    return)                  ; return the value at the top of the stack

  (func (;1;) (result i64)   ; declare function index 1 (main) with no parameters and an i64 result
    (local i64 i64 i64 i64)  ; declare 4 local variables, all with type i64, one for each symbol in the Mono IR
    i64.const 1              ; stack=[1]
    local.set 0              ; stack=[]     local0=1
    i64.const 2              ; stack=[2]    local0=1
    local.set 1              ; stack=[]     local0=1  local1=2
    local.get 0              ; stack=[1]    local0=1  local1=2
    local.get 1              ; stack=[1,2]  local0=1  local1=2
    call 0                   ; stack=[3]    local0=1  local1=2
    local.set 2              ; stack=[]     local0=1  local1=2  local2=3
    i64.const 4              ; stack=[4]    local0=1  local1=2  local2=3
    local.set 3              ; stack=[]     local0=1  local1=2  local2=3  local3=4
    local.get 2              ; stack=[3]    local0=1  local1=2  local2=3  local3=4
    local.get 3              ; stack=[3,4]  local0=1  local1=2  local2=3  local3=4
    call 0                   ; stack=[7]    local0=1  local1=2  local2=3  local3=4
    return)                  ; return the value at the top of the stack
```

If we run this code through the `wasm-opt` tool from the [binaryen toolkit](https://github.com/WebAssembly/binaryen#tools), the unnecessary locals get optimised away (which is all of them in this example!). The command line below runs the minimum number of passes to achieve this (`--simplify-locals` must come first).

```
$ wasm-opt --simplify-locals --reorder-locals --vacuum example.wasm > opt.wasm
```

The optimised functions have no local variables at all for this example. (Of course, this is an oversimplified toy example! It might not be so extreme in a real program.)

```
  (func (;0;) (param i64 i64) (result i64)
    local.get 0
    local.get 1
    i64.add)
  (func (;1;) (result i64)
    i64.const 1
    i64.const 2
    call 0
    i64.const 4
    call 0)
```

### Reducing sets and gets

It would be nice to find some cheap optimisation to reduce the number of `local.set` and `local.get` instructions.

We don't need a `local` if the value we want is already at the top of the VM stack. In fact, for our example above, it just so happens that if we simply skip generating the `local.set` instructions, everything _does_ appear on the VM stack in the right order, which means we can skip the `local.get` too. It ends up being very close to the fully optimised version! I assume this is because the Mono IR within the function is in dependency order, but I'm not sure...

Of course the trick is to do this reliably for more complex dependency graphs. I am investigating whether we can do it by optimistically assuming it's OK not to create a local, and then keeping track of which symbols are at which positions in the VM stack after every instruction. Then when we need to use a symbol we can first check if it's on the VM stack and only create a local if it's not. In cases where we _do_ need to create a local, we need to go back and insert a `local.set` instruction at an earlier point in the program. We can make this fast by waiting to do all of the insertions in one batch when we're finalising the procedure.

For a while we thought it would be very helpful to reuse the same local for multiple symbols at different points in the program. And we already have similar code in the CPU backends for register allocation. But on further examination, it doesn't actually buy us much! In our example above, we would still have the same number of `local.set` and `local.get` instructions - they'd just be operating on two locals instead of four! That doesn't shrink much code. Only the declaration at the top of the function would shrink from `(local i64 i64 i64 i64)` to `(local i64 i64)`... and in fact that's only smaller in the text format, it's the same size in the binary format! So the `scan_ast` pass doesn't seem worthwhile for Wasm.

## Memory

WebAssembly programs have a "linear memory" for storing data, which is a block of memory assigned to it by the host. You can assign a min and max size to the memory, and the WebAssembly program can request 64kB pages from the host, just like a "normal" program would request pages from the OS. Addresses start at zero and go up to whatever the current size is. Zero is a perfectly normal address like any other, and dereferencing it is not a segfault. But addresses beyond the current memory size are out of bounds and dereferencing them will cause a panic.

The program has full read/write access to the memory and can divide it into whatever sections it wants. Most programs will want to do the traditional split of static memory, stack memory and heap memory.

The WebAssembly module structure includes a data section that will be copied into the linear memory at a specified offset on initialisation, so you can use that for string literals etc. But the division of the rest of memory into "stack" and "heap" areas is not a first-class concept. It is up to the compiler to generate instructions to do whatever it wants with that memory.

## Stack machine vs stack memory

**There are two entirely different meanings of the word "stack" that are relevant to the WebAssembly backend.** It's unfortunate that the word "stack" is so overloaded. I guess it's just a useful data structure. The worst thing is that both of them tend to be referred to as just "the stack"! We need more precise terms.

When we are talking about the instruction set, I'll use the term _machine stack_ or _VM stack_. This is the implicit data structure that WebAssembly instructions operate on. In the examples above, it's where `i64.add` gets its arguments and stores its result. I think of it as an abstraction over CPU registers, that WebAssembly uses in order to be portable and compact.

When we are talking about how we store values in _memory_, I'll use the term _stack memory_ rather than just "the stack". It feels clunky but it's the best I can think of.

Of course our program can use another area of memory as a heap as well. WebAssembly doesn't mind how you divide up your memory. It just gives you some memory and some instructions for loading and storing.

## Calling conventions & stack memory

In WebAssembly you call a function by pushing arguments to the stack and then issuing a `call` instruction, which specifies a function index. The VM knows how many values to pop off the stack by examining the _type_ of the function. In our example earlier, `Num.add` had the type `[i64 i64] → [i64]` so it expects to find two i64's on the stack and pushes one i64 back as the result. Remember, the runtime engine will validate the module before running it, and if your generated code is trying to call a function at a point in the program where the wrong value types are on the stack, it will fail validation.

Function arguments are restricted to the four value types, `i32`, `i64`, `f32` and `f64`. If those are all we need, then there is _no need for any stack memory_, stack pointer, etc. We saw this in our example earlier. We just said `call 0`. We didn't need any instructions to create a stack frame with a return address, and there was no "jump" instruction. Essentially, WebAssembly has a first-class concept of function calls, so you don't build it up from lower-level primitives. You could think of this as an abstraction over calling conventions.

That's all great for primitive values but what happens when we want to pass more complex data structures between functions?

Well, remember, "stack memory" is not a special kind of memory in WebAssembly, and is separate from the VM stack. It's just an area of our memory where we implement a stack data structure. But there are some conventions that it makes sense to follow so that we can easily link to Wasm code generated from Zig or other languages.

### Observations from compiled C code

- `global 0` is used as the stack pointer, and its value is normally copied to a `local` as well (presumably because locals tend to be assigned to CPU registers)
- Stack memory grows downwards
- If a C function returns a struct, the compiled WebAssembly function has no return value, but instead has an extra _argument_. The argument is an `i32` pointer to space allocated in the caller's stack, that the called function can write to.
- There is no maximum number of arguments for a WebAssembly function, and arguments are not passed via _stack memory_. This makes sense because the _VM stack_ has no size limit. It's like having a CPU with an unlimited number of registers.
- Stack memory is only used for allocating local variables, not for passing arguments. And it's only used for values that cannot be stored in one of WebAssembly's primitive values (`i32`, `i64`, `f32`, `f64`).

These observations are based on experiments compiling C to WebAssembly via the Emscripten toolchain (which is built on top of clang). It's also in line with what the WebAssembly project describes [here](https://github.com/WebAssembly/design/blob/main/Rationale.md#locals).

## Modules vs Instances

What's the difference between a Module and an Instance in WebAssembly?

Well, if I compare it to running a program on Linux, it's like the difference between an ELF binary and the executable image in memory that you get when you _load_ that ELF file. The ELF file is essentially a _specification_ for how to create the executable image. In order to start executing the program, the OS has to actually allocate a stack and a heap, and load the text and data. If you run multiple copies of the same program, they will each have their own memory and their own execution state. (More detail [here](https://wiki.osdev.org/ELF#Loading_ELF_Binaries)).

The Module is like the ELF file, and the Instance is like the executable image.

The Module is a _specification_ for how to create an Instance of the program. The Module says how much memory the program needs, but the Instance actually _contains_ that memory. In order to run the Wasm program, the VM needs to create an instance, allocate some memory for it, and copy the data section into that memory. If you run many copies of the same Wasm program, you will have one Module but many Instances. Each instance will have its own separate area of memory, and its own execution state.

## Modules, object files, and linking

A WebAssembly module is equivalent to an executable file. It doesn't normally need relocations since at the WebAssembly layer, there is no Address Space Layout Randomisation. If it has relocations then it's an object file.

The [official spec](https://webassembly.github.io/spec/core/binary/modules.html#sections) lists the sections that are part of the final module. It doesn't mention any sections for relocations or symbol names, but it has room for "custom sections" that in practice seem to be used for that.

The WebAssembly `tool-conventions` repo has a document on [linking](https://github.com/WebAssembly/tool-conventions/blob/main/Linking.md), and the `parity_wasm` crate supports "name" and "relocation" [sections](https://docs.rs/parity-wasm/0.42.2/parity_wasm/elements/enum.Section.html).