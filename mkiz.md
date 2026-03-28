<center><h1>Modern Kiz Reference</h1></center>

Modern Kiz (Kiz 2.0.x, hereinafter MKiz) is a major refactor of Traditional Kiz[^1] (hereinafter TKiz).
MKiz introduces many radical new features while removing outdated ones from TKiz.

Notably, TKiz uses Python-style `fn foo: ...` syntax, as opposed to the original `fn foo ... end` syntax in earlier versions of TKiz.

> [!IMPORTANT]
> This document is not a formal specification on par with standards such as the C++ Standard, as time constraints limit the level of detail and formality in its writing 🙏

# Arithmetic Expressions

### String Literal
`"asdfghjklqwertyyuiopzxcvbbnm"`: Basic string literal
`r"asdfghjklqwertyyuiopzxcvbbnm"`: Raw string literal
`t|SqlChecker|"select {pattern name} from a"`: Template string literal, which is validated by a designated checker to prevent XSS/SQL injection vulnerabilities

`"""abc\bdef"""`: Multi-line string literal, delimited by triple `"""`

### Time Literal
- `3Ms`: 3 milliseconds
- `3S`: 3 seconds
- `3Min`: 3 minutes
- `3Hour`: 3 hours

### Number Literal
`111`: Basic integer literal
`111_111`: Integer literal with visual digit separators
`111.222`: Basic decimal literal
`111.222f`: Explicit single-precision floating-point literal
`111.222e-10`: Literal in scientific notation

### List Literal
`[1,2,"y"]`: Basic list literal

### Tuple Literal
`(1,0.2,"y")`: Basic tuple literal

### Record Literal
`{key1=1, key2=2, key3="y"}`: Basic list literal

### Unary Operator
- `-a`: Arithmetic negation operator

### Binary Operator
- `a + b`: Addition operator
- `a - b`: Subtraction operator
- `a * b`: Multiplication operator
- `a / b`: Division operator
- `a ** b`: Exponentiation operator
- `a % b`: Modulo operator

### Call Expression
`a(param1=arg1, param2=arg2)`: Function invocation using keyword arguments (MKiz supports keyword arguments exclusively)

### Member Access
`a.b`: Accesses a member via the dot operator. MKiz does not support traditional direct member assignment syntax; use a setter method instead.
# Variables & Assignment
### Variable Assignment & Definition
MKiz uses Python-like syntax, requiring clear distinction between **assignment** and **definition**.

`::Type` denotes a type annotation (in contrast to Rust’s `: Type`).

A variable’s type can be inferred in nearly all cases, so explicit type annotations are unnecessary.

```mkiz
name::Type = pattern expression
```

```mkiz
content::String = move io.read(path="demo.txt")
content_view = view content
content_duplicate = clone content
```

### Pattern
Patterns are **mandatory** in MKiz.
The language supports the following patterns:
- `view`: A lightweight view of data; changes to the original data are reflected synchronously in the view.
- `frozenview`: A read-only, immutable view of data; the view is invalidated if the original data is mutated.
- `inout`: A mutable view allowing modification of the underlying data.
- `move`: Transfers ownership from the source data.
- `clone`: Creates an independent copy of the data.

# Effects
MKiz supports **Algebraic Effects**, where an effect represents a side effect.

```mkiz
effect $EffectName
```

Example:
```mkiz
effect $IOEff
```

Effect annotations mark the side effects a procedure may produce.
Foreign tasks **must** be annotated with side effects[^2].

```mkiz
extern "stdio" // a C library
    task read(path::String) -> String \ $IOEff

task foo() -> String \ $IOEff
    return move io.read(path="demo.txt")?
```

# Functions & Tasks
### Function
A function represents a mathematical function and is **pure**.

```mkiz
fn foo(param::Type) -> Type
    statements
```

### Task
The only difference between a task and a function is that a task explicitly declares side effects.

```mkiz
fn foo(param::Type) -> Type \ $Effect1, $Effect2
    statements 
```

```mkiz
task print(
    ...objects::Any,
    end::String
) -> () \ $io.ConsoleEff
    statements
```

### Return
```mkiz
return pattern expression
```

### Declaration-Time Annotations
Only one annotation is allowed at declaration.

```
#annotation(param=arg)
fn foo(param::Type) -> Type \ $Effect1, $Effect2
    statements 
```

### Call-Time Annotations
Multiple annotations are supported at call site.
```
foo()
@annotation1(a=x)
@annotation2(b=y)
```

# Time
*"Time is a first-class citizen!"*
### Time-Limited Types
Values of such types are only valid for a fixed duration, after which they become invalid.

```
a::Int!3Ms = 0
print(a.must()) // Panics if a has expired
```

### Assignment & Arithmetic
```
a::Time = 3Ms
b::Time = a*2 // 6Ms
```

### Temporal Contracts
```
// Promises to return within 100 milliseconds
#within(time=100Ms)
task quick_read(f::File) -> String \ $IOEff, $TimeEff
    ...

// May only execute during a specified time window
#onlyduring(start=2am, end=4pm)
task daily_job() -> () \ $IOEff
    ...
```

### Time-Limited Operations
```
content = move io.read(path="demo.txt")
    @mustin(3ms)
```
# Permission
Permission is useful when someone want to call your core function.🥳[^3]
### Basic Permissions
Syntax under discussion
### Sandbox
Syntax under discussion

# Data & Capabilities

### Trait
Use the `trait` keyword to define a trait.
```
trait TraitName
```

### Data
```
data DataClass(Trait1, Trait2) =
    * field1::Type
    * field2::Type
    * field3::Type
```

```
data Dog =
    * name::String
    * age::Int
```

Create a data instance:
```
a::data~Dog = move Dog(name="Tom", age=10)
```

### Ability
Use the `ability` keyword to define an ability.
Abilities are pure.

# Type

### Protocol
Use the `protocol` keyword to define a protocol.
```
protocol ProtocolName
    ability name1(param1::Type) -> Type
    ability name2(param1::Type) -> Type
```

### Type
*Type is the set of abilities.*

```
type TypeName : Protocol1, Protocol2
    data AssociatedData =
      * name::Type
      * age::Type

    ability name1(param1::Type) -> Type
        statements

    // Allow mutation of associated data
    #mutdata
    ability name2(param1::Type) -> Type
        statements
```

Use `type(var)` to infer a type of a expression.
```
a = 1
b::type(a) = 1
```

Access a field in an ability:
```
data.name
```

Set a field in an ability:
```
data.name set = expression
```

### Duck Type System
*If an animal has a duck’s abilities, it is a duck.*

When invoking an ability (called a method in other languages), the concrete type of `atype` is irrelevant.
```
atype.bark()
```

### Constrained Types
MKiz does not provide traditional generics, but supports **type agreements**.
```
// Accepts any type implementing add and mul protocols
fn foo(a::Any[add+mul]) -> type(a)
    return a+a*a
```

# Error
`%%n%%` denotes a placeholder to be substituted.
```
error ErrName: ParentError(
"for padding string"
)
```

```
error FileNotFoundError: IoError(
"file %%fname%% not found"
)
```

Works with the Result type (`T?ErrT`):
```
task foo() -> String?FileNotFoundError
    return error FileNotFoundError(fname="demo.txt")
```

Use `?` to propagate errors upward:
```
task bar() -> String?FileNotFoundError
    return foo()?
```

# Variant
In MKiz, a variant combines a type tag and a union (similar to Rust `enum`).
```
variant VariantName = | SubName1::Type
                      | SubName2::Type
                      | SubName3::Type
```

Create a variant instance:
```
instance = move VariantName.SubName1(value)

// Variant name can be omitted when type is unambiguous
instance::VariantName = .SubName1(value)

fn foo(m::VariantName) -> ()
    return

foo(m=.SubName1(value))
```

# Actor
```
actor ActorName
    // MessageKind is a conventional variant for messages
    variant MessageKind = MsgKind1 | MsgKind2
        
    // handle message
    on name::Type | MsgKind1
        statements
    
    // Pure
    fn name1(param1::Type) -> Type
        statements
    
    // Impure
    task name1(param1::Type) -> Type \ $Eff
        statements
```

Use `send` to send a message to another actor:
```
send Actor2(msg=expression, kind=.MsgKind)
```

# Control Flow

### Walrus Operator `:=`
```
if name := pattern expression
    statements
```

- `:=` requires a **name** on the left and an **expression** on the right.
- Only two forms are allowed:
  `name := pattern expression` or `expression`.
- Unlike Python, **chained assignments** are prohibited.
- May only be used in conditional expressions of `if`, `elif`, and `while`; invalid in all other expression contexts.

**Invalid:**
```
if name := pattern name := expression
    statements
```

**Supported:**
```
if name1, name2 := pattern expression
    statements
```

### If & Elif & Else
MKiz provides a traditional `if` construct.

```
if expression
   statements
elif name := pattern expression
    statements
elif expression
    statements
else
   statements
```

### While Loop
MKiz uses a traditional `while` loop.
```
while expression
    statements
```

An `if break` clause is used for cleanup logic after loop termination.
```
while expression
    statements
if break
   statements
else
   statements
```

### For Loop
MKiz provides a traditional `for` loop.
```
for name in view iterable
    statements
```

A trailing `if` guard can be used to filter elements (similar to list comprehensions in Python).
```
for name in view iterable if expression
    statements
```

The `if break` clause is also supported for post-loop cleanup logic.
```
for name in view iterable
    statements
if break
   statements
else
   statements
```

### Data Matcher
The `when` statements performs pattern matching on variants, data structures, or values.

Matching a variant or tagged data:
```
when for_check
    .SubName1(assign_on_name1) => 
        statements
    .SubName2(assign_on_name2) =>
        statements
```

Using brace `{}` syntax to match a structured data type:
```
when for_check
    DataClass1{assign_on_name1, assign_on_name2} => 
        statements
    DataClass2{assign_on_name1} =>
        statements
```

Direct matching against literal values:
```
when for_check
    1 =>
       statements
    "6" =>
        statements
```

# Module System
MKiz features a significantly improved module system compared to TKiz.[^4]
In MKiz, modules are first-class citizens and can be passed, returned, and manipulated as values.

### Module Declaration
A module declaration at the top of a file scopes the entire file to that module.
```
module module_name
```

### Import
Use the `use` keyword to import modules. MKiz supports the following forms:
- Basic import
```
use module_name
```

- Restricted import (dependencies included)
```
use module_name (foo1, foo2)
```

- Open import
```
use module_name*
```

- Open import with exclusions
```
use module_name* no (foo1, foo2)
```

Access module members via the dot `.` operator:
```
module_name.member
```

### Visibility
The default visibility is private. Use the `pub` keyword to mark items as public.
```
pub task
pub fn
pub data
pub trait
pub type
```

---
**Thank you for reading.**

[^1]: Traditional Kiz refers to Kiz 0.x.x and 1.x.x, with syntax inspired by Lua and Ruby. It represents the first experimental iteration of the Kiz language.

[^2]: Well, if you intend to extern a pure external function, why not define it directly in MKiz instead? 😋

[^3]: That’s a sarcastic “Yay 🥳”, in case you couldn’t tell.

[^4]: This “Yay 🥳” is genuine.

author: *azhz1107cat*