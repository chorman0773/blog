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
public function _ZN5test12f0Ed(float(64)) -> int(32){
    convert weak int(32)
    exit 1
}
```

A very simple program with a very simple translation - convert the input value to a 32-bit integer, then return it.

To show the behaviour of the program, in loose terms, to validate that the lowering is correct, we can add stack type annotations in comments, placed in square brackets, that show the state of the current value stack that xlang sees after each expression. (The annotations are not processed by the compiler, and are simply added for ease of reading)

```
public functon _ZN5test12f0Ed(float(64)) -> int(32){
       // [float(64)]
       convert weak int(32) // [int(32)]
       exit 1
}
```

Function parameters are placed on the stack in their appropriate order and are already there when the function begins execution.
`convert weak int(32)` then pops a value off the stack, in this case `float(64)`, performs a conversion, and pushes the result, in this case `int(32)`.

`exit 1` then pops 1 value off the stack and "exits" (that is, returns from) the current function with that popped value, which is the return value.

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
    const global_address function(*void())->int(32) __tiny_read_INT
    const global_address function(*const readonly nonnull char(8)) -> *void() __tiny_const_string
    const *char(8) "test-stdin\n"
    derive *const readonly nonnull char(8) nop
    call function(*const readonly nonnull char(8))->*void()
    call function(*void())->int(32)
    const global_address function(int(32),*void()) __tiny_write_INT
    pivot 1 1
    const global_address function(*const readonly nonnull char(8)) -> *void() __tiny_const_string
    const *char(8) "test-stdin\n"
    derive *const readonly nonnull char(8) nop
    call function(*const readonly nonnull char(8))->*void()
    call function(int(32),*void())
    const int(32) 0
    exit 1
}

public function main() -> int(32){
    const global_address function() __tiny_rt_init
    call function()
    const global_address function()->int(32) _ZN5test24mainEv
    call function()->int(32)
    const global_address function() __tiny_rt_cleanup
    call function()
    exit 1
}
```

Adding stack type annotations, we generating

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

