# LCCC - Intro to XLang Design and Examples

[![Creative Commons License](https://i.creativecommons.org/l/by/4.0/80x15.png)](http://creativecommons.org/licenses/by/4.0/)

All content on this blog is licensed under the CC-BY 4.0 License. 

## Intro to XLang

[XLang](https://lccc.lcdev.xyz/xlang) is the intermediate architectuer and intermediate representation language used by the [LCCC](https://github.com/lccc-project/lccc) project for lowering and optimizing programs from a source language to machine code.

It is a stack based language that was originally designed in early 2020 by myself as part of the LCCC project, though it has undergone some revisions since.

## General Concepts

XLang is a strict stack-base architecture - within a function, all instructions, also referred to as expressions, will pop a certain number of input values from a stack, perform some operation, and push each output value back to the same stack. There is no random access to the stack (though the `pivot` instruction offers the ability to move values up by popping off two groups then pushing them in the opposite order), and the stack cannot be addressed.
You also cannot modify an entry on the stack, but would rather have to pop the value and then push the new value.

Local variables are offered when mutable, addressible memory is required.

There are two kinds of values on the stack: rvalues and lvalues. An rvalue is a simple value, such as `0`, `512`, `2.71828`, a pointer, or a composite value like a struct.
An lvalue is a synthetic value that designates some part of memory, known as an object.

Computations are performed on rvalues, while lvalues must be read or written to.

## The xlang crate graph

Interacting with xlang programmatically will typically require using the xlang crate graph. 

These crates can be obtained by using a `git` dependency on the lccc repository (`https://github.com/lccc-project/lccc`) or cloning it as a submodule.

You can also interact with these using the ABI structure the crates expose, but this is not presently supported.

The main crate graph starts with the `xlang` crate, which depends on the rest of the main crate graph, and reexpotrs them. It also links the `xlang_interface` system library, which provides a minimal api surface for ensuring different [plugins](#xlang-plugins) perform tasks like allocation and deallocation consistently. The remaining crates are:
* `xlang_host`, which exposes primitive abstractions over the host environment (exported as `xlang::host`)
* `xlang_abi`, which exposes ABI-safe data structures and primitives, many which match or closely follow the interface of standard library data structures (exported as `xlang::abi`)
* `xlang_target`, which exposes structures that describe the properties of targets which code may be generated for by lccc and a mechanism to query target properties (exported as `xlang::target`)
* `xlang_struct`, which exposes a [structural representation](#building-ir) of xlang ir

The `xlang` crate then is dependend upon by a few optional crates for writing [plugins](#xlang-plugins):
* `xlang_backend` exposes interfaces that abstract on backend plugins
* `xlang_frontend` exposes interfaces that abstract on language frontends
* `xlang_opt` exposes interfaces that abstract on optimizers

These optional crates are not required, but may improve the experience of writing these kinds of plugins.

## XLang Plugins

Plugins are used by drivers to perform tasks relevant to the translation from source code to machine code. They are system dylibs (IE. a shared object or dll) with a special entry point that is called by the driver.

There are 3 main kinds of plugins:
1. Frontends, which read source files and produces xlang ir,
2. Transformers, which accept and modify xlang ir in place (most transformer plugins will be optimization passes), and
3. Backends, which consumes xlang ir and produce something else, such as machine code, or a lower-level IR (like wasm).


Each main plugin has a different entry point, exported as an unmangled symbol using the lcrust-v0 calling convention (via the `rustcall!()` macro from `xlang_host`), that exposes a trait object that satisfies the appropriate trait in the `xlang::plugin` module. Trait objects are captured accross the abi boundary using types in `xlang_abi::traits`.


Frontend plugins expose `xlang_frontend_main`, which returns a trait object satisfying `XLangFrontend`. Backend plugins expose `xlang_backend_main`, which returns a trait object satisfying `XLangBackend`.  Transformer plugins expose `xlang_plugin_main`, which returns a trait object satisfying `XLangPlugin`.

As plugins share data structures accross an ABI boundary, ABI-safe types provided by `xlang_abi` are used throughout much of `xlang`, and the lccc infrastructure.

## Building IR

IR Building occurs using the `xlang_struct` crate, which is also reexported via the `xlang::ir` module.

The IR is represented as a number of structs, which can be constructed and modified by [plugins](#xlang-plugins). The structs contain abi-safe data structures provided primarily by the `xlang_abi` crate.

Each plugin is passed a mutable reference to the top level `File` by the `accept_ir` method of the `XLangPlugin` trait. Frontends would then build the IR and write it to the reference, backends would read the IR and build it's output, and transformers modify the `File` in place. 

## Stack Operations in XIR, or how to do interesting things


Many operations in XIR are typed, and perform some transformation on the values taken from the stack before pushing new ones. However, three expressions manipulate the stack in more generalized ways. The expressions are `pivot`, `pop`, and `dup`, and are collectively called the stack operations.

`pop` and `dup` are both relatively straightforward for anyone who's worked with a stack, but has a few interesting features. Both are provided with a single integer value we'll call `n`. Both instructions, `pop n` and `dup n`, pop `n` values from the stack. With `pop n` these values are discarded, and `dup n` pushs the values in the reverse order they were popped, then repeats the process. As a net result, `dup n` duplicates a group of `n` values from the stack. In the XIR parser, these expressions may be written simply as `pop` or `dup`, in which case an `n` of `1` is inferred.

`pivot` is an interesting expression. It is provided with 2 values, which we'll call `m` and `n`. Strictly speaking, `pivot m n` pops `n` values from the stack in a group we'll call `S`, and then pops `m` values from the stack in a group we'll call `T`. It then pushes the values in `S` in the opposite order that they were popped (or, the same order they appeared on the stack in the first case), and then pushes the values in `T`.
This is a powerful operation in XIR, as it can provide effectively random access to the value stack in a proper stack operation.

There notable cases are particularily useful for lowering source languages, though others may be useful in niche cases, and show up in optimized IR that makes heavier use of the stack compared to local variables (which are stored primarily in memory).
* `pivot 1 n`, moves 1 value to the top of the stack from `n` values down.
* `pivot n 1` moves 1 value from the top of the stack to `n` values down (not including that value itself),
* `pivot n n` swaps two equally sized groups of `n` values.

Like `pop` and `dup`, `pivot` may appear in two special-cased forms. A single input like `pivot n` means `pivot n n` (performs the swap operation), and no inputs like simply `pivot` means `pivot 1 1` (swaps 1 value). 


## An Example, the Tiny language lowered to XLang

The [Tiny Language](https://github.com/chorman0773/TinyCompiler/blob/main/lang-spec/README.md) is a simple language that was introduced in a compilers course at my university.
While the course only referred to the lexical and syntatic grammar of the language, I have personally expanded it to include semantic analysis and codegen (to JVM bytecode), and even added a few extensions.

The grammar of the language is, in ABNF:

```abnf

QUOTE := %x22

STRING_CHAR := %x01-09 \ %x0B-0C \ %x0E-21 \  %x23-7F \ %x80-D7FF \ %xE000-10FFFF

string-literal := <QUOTE> [*<STRING_CHAR>] <QUOTE>

DIGIT := %x30-39

number-literal := *<digit>["." *<digit>]

literal := <number-literal> / <string-literal>

identifier := <XID_Start>[*<XID_Part>]

file := *<method-decl>

method-decl := <type> ["MAIN"] <identifier> "(" <param-list> ")" <block>

type := "INT" / "STRING" / "REAL"

param-list := [<param> / (<param> "," <param-list>)]

param := <type> <identifier>

block := "BEGIN" [*<statement>] "END"

statement := <declaration> / <assignment> / <read> / <write> / <if> / <block> / <return>

declaration := <type> <identifier> [":=" <expression>] ";"
assignment := <identifier> ":=" <expression> ";"
read := "READ" "(" <identifier> "," <string-literal> ")" ";"
write := "WRITE" "(" <expression> "," <string-literal> ")" ";"
if := "IF" "(" <bool-expression> ")" <statement> ["ELSE" <statement>]
return := "RETURN" <expression> ";"

bool-expression := (<expression> ("==" / "!=") <expression>)
```

The semantics of Tiny are fairly simple: READ reads from a named file, and WRITE writes to it. `IF` takes one branch if a value evaluates to true, and another if it evaluates to false.
The function marked `MAIN` is run at startup.

Using this, we can write a few test programs, taken from a [full compiler](https://github.com/chorman0773/TinyCompiler), and explore how they could be translated to XLang IR.

For the compilation of these tests, I have elected to use a basic variation of the [Itanium Name Mangling](https://itanium-cxx-abi.github.io/cxx-abi/abi.html#mangling) scheme. `INT`, being a 32-bit integer is encoded as `i`, `REAL` as `d`, and `STRING` as u6string

### test1 - A fairly basic example

```
INT f0(REAL x)
BEGIN
    RETURN x;
END
```


In a basic xlang lowering, we can translate the above to:

```
public functon _ZN5test12f0Ed(float(64)) -> int(32){
       // [float(64)]
       convert weak int(32) // [int(32)]
       exit 1
}
```

A very simple program with a very simple translation - convert the input value to a 32-bit integer, then return it.

Function parameters are placed on the stack in their appropriate order and are already there when the function begins execution.
`convert weak int(32)` then pops a value off the stack, in this case `float(64)`, performs a conversion, and pushes the result, in this case `int(32)`.

`exit 1` then pops 1 value off the stack and "exits" (that is, returns from) the current function with that popped value, which is the return value.

To show the behaviour of the program, in loose terms, to validate that the lowering is correct, stack type annotations were added to show the state of the stack after each statement. These aren't interpreted by the XIR frontend (as they appear in comments), but are provided as a visual guide to show the world as understood by xlang.



### test2 - A simple executable program


```
INT MAIN main() BEGIN
   INT x;
   READ(x,"test-stdin");
   WRITE(x,"test-stdout");
END
```

This needs some more work, and we need some dependencies for the `READ` and `WRITE` statements, as well as for creating strings (an extension allows us to ). We're also going to lower the `MAIN` function in two steps: one step compiling the function directly, and a second generating a call stub that does some initialization, calls the main function, and then returns.
The definitions of the `__tiny_read_INT`, `__tiny_write_INT`, `__tiny_const_string`, `__tiny_rt_init`, and `__tiny_rt_cleanup` are left as an implementation-detail and are largely irrelevant for this program:

```
public function __tiny_read_INT(*void()) -> int(32);
public function __tiny_write_INT(int(32), *void());

public function __tiny_const_string(*const readonly nonnull char(8)) -> *void();
public function __tiny_rt_init();
public function __tiny_rt_cleanup();

public function _ZN5test24mainEv() -> int(32){
    // []
    const global_address function(*void())->int(32) __tiny_read_INT // [*function(*void())->int(32)]
    const global_address function(*const readonly nonnull char(8)) -> *void() __tiny_const_string // [*function(*void())->int(32), *function(*const readonly nonnull char(8))->void()]
    const *char(8) "test-stdin\n" // [*function(*void())->int(32), *function(*const readonly nonnull char(8))->void(), *char(8)]
    derive *const readonly nonnull char(8) // [*function(*void())->int(32), *unction(*const readonly nonnull char(8))->void(), *const readonly nonnull char(8)]
    call function(*const readonly nonnull char(8))->*void() // [*function(*void())->int(32), *void()]
    call function(*void())->int(32) // [int(32)]
    const global_address function(int(32),*void()) __tiny_write_INT // [int(32), *function(int(32),*void())]
    pivot 1 1 // [*function(int(32),*void()), int(32)]
    const global_address function(*const readonly nonnull char(8)) -> *void() __tiny_const_string // [*function(int(32),*void()), int(32), *function(*const readonly nonnull char(8)) -> *void()]
    const *char(8) "test-stdin\n" // [*function(int(32),*void()), int(32), *function(*const readonly nonnull char(8)) -> *void(), *char(8)]
    derive *const readonly nonnull char(8) // [*function(int(32),*void()), int(32), *function(*const readonly nonnull char(8)) -> *void(), *const readonly nonnull char(8)]
    call function(*const readonly nonnull char(8))->*void() // [*function(int(32),*void()), int(32), *void()]
    call function(int(32),*void()) // []
    const int(32) 0 // [int(32)]
    exit 1
}

public function main() -> int(32){
    const global_address function() __tiny_rt_init // [*function()]
    call function() // []
    const global_address function()->int(32) _ZN5test24mainEv // [*function() -> int(32)]
    call function()->int(32) // [int(32)]
    const global_address function() __tiny_rt_cleanup // [int(32), *function()]
    call function() // [int(32)]
    exit 1
}
```
External functions can be declared by giving the signature without a body, ending the line with `;`.

We use the `const global_address` expresion to push the address of a global, in this case a function. 
`const` in general pushes a sole constant value to the stack (such as an scalar value, uninitialized value, or, as in this case, the address of some function to cal). 
`const *char(8) <string-literal>` is another example of this, which pushes a string literal with the given type (in this case, a pointer, so the string goes into rodata somewhere else in memory rather than appear directly on the stack). 

The `derive` expression following the string literal push is an type manipulation instruction for pointers. It changes attributes on a pointer type used by the optimizer. In many cases it also forms a new edge in the provenance[^1]. The Pointer model of xlang is not discussed in this post.

The `call` expression, intuitively, calls a function. It pops off the parameter list specified by the signature it is given, then pops the function to call (usually a function pointer). It then performs a call by pushing the arguments in the callee frame, and beginning execution with the first statement in the function. After that function returns (via `exit`), if the function doesn't return `void()`, the `call` expression pushes the return value from the function back in the callers frame. 

We've examined the [`pivot` expression](#stack-operations-in-xir-or-how-to-do-interesting-things) in detail in a previous section, so it won't be explained again here.

### test3 - Adding some Control (Flow)

Our third, and final example, is another program that this time includes the use of control flow, through `IF`. 

```
INT MAIN main() BEGIN
   INT x;
   INT y;
   READ(x,"test-stdin");
   IF (x==1)
       y := 0;
   ELSE
       y := x;
   WRITE(y,"test-stdout");
END
```

Here, we read a value from `test-stdin` into `x`, then compare `x` and `1`. If they are equal, then set `y` to `0` otherwise set `y` to `x`. 
We then write the value of `y` to `test-stdout`.

For this example, the entry point `main` is omitted, and is exactly the same as the one from `test2`. We'll only show the actual written code translated

```
public function __tiny_read_INT(*void()) -> int(32);
public function __tiny_write_INT(int(32), *void());

public function __tiny_const_string(*const readonly nonnull char(8)) -> *void();

public function _ZN5test34mainEv() -> int(32){
    // []
    const global_address function(*void())->int(32) __tiny_read_INT // [*function(*void())->int(32)]
    const global_address function(*const readonly nonnull char(8)) -> *void() __tiny_const_string // [*function(*void())->int(32), *function(*const readonly nonnull char(8))->void()]
    const *char(8) "test-stdin\n" // [*function(*void())->int(32), *function(*const readonly nonnull char(8))->void(), *char(8)]
    derive *const readonly nonnull char(8) // [*function(*void())->int(32), *unction(*const readonly nonnull char(8))->void(), *const readonly nonnull char(8)]
    call function(*const readonly nonnull char(8))->*void() // [*function(*void())->int(32), *void()]
    call function(*void())->int(32) // [int(32)]
    dup // [int(32),int(32)]
    const int(32) 1 // [int(32),int(32),int(32)]
    cmp_eq // [int(32),uint(1)]
    branch not_equal @0 // [int(32)]
    pop // []
    const int(32) 0 // [int(32)]
    target @0 [int(32)] // [int(32)]
    const global_address function(int(32),*void()) __tiny_write_INT // [int(32), *function(int(32),*void())]
    pivot 1 1 // [*function(int(32),*void()), int(32)]
    const global_address function(*const readonly nonnull char(8)) -> *void() __tiny_const_string // [*function(int(32),*void()), int(32), *function(*const readonly nonnull char(8)) -> *void()]
    const *char(8) "test-stdin\n" // [*function(int(32),*void()), int(32), *function(*const readonly nonnull char(8)) -> *void(), *char(8)]
    derive *const readonly nonnull char(8) // [*function(int(32),*void()), int(32), *function(*const readonly nonnull char(8)) -> *void(), *const readonly nonnull char(8)]
    call function(*const readonly nonnull char(8))->*void() // [*function(int(32),*void()), int(32), *void()]
    call function(int(32),*void()) // []
    const int(32) 0 // [int(32)]
    exit 1
}

```

`branch` and `target` are the two main new statements here, along with `cmp_eq`.

`cmp_eq` is the simplest. It compares two values and pushes `0` if they are unequal, and `1` if they are equal. The pushed value is of type `uint(1)` (a 1 bit unsigned integer type), though this rarely matters.

`branch` "branches" to a target (which can be considered like a label in assembly or C), when some condition is satisfied. You can specify either a condition code, such as `not_equal` used here, `always`, or `never`, to indicate when to `branch`. When a condition is specified, it will pop a value from the stack which has an integer type. The relation of that value to `0` is then checked according to the condition code. In the case of `not_equal`, the branch is taken when the value used is not equal to `0`. `branch always` and `branch never` do not pop a value, and will either always (or never) take the branch. 
In all cases, `branch` may perserve a certain set of values from the top of the stack into the target when it takes the branch.

The `target` statement can be used to declare a target of a branch. As part of the statement, an "incoming" stack is specified, which is preserved from branches to the target that are taken. It is also preserved when "falling through" into a target from a previous expression, in this case, the values are also taken from the top of the stack and the bottom values are discarded. 
Certain expressions, such as `exit` or `branch always` are treated as "diverging" (IE. they do not resume execution, so they do not have an output stack). These will typically be followed by a target so execution can be reached from another part of the program.

The remainder of the function is similar to the second example, so it is not explained here.

[^1]: https://www.ralfj.de/blog/2018/07/24/pointers-and-bytes.html