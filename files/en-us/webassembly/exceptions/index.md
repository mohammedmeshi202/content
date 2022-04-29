---
title: WebAssembly exception handling
slug: WebAssembly/Exceptions
tags:
  - C++
  - Compiling
  - Emscripten
  - WebAssembly
  - Exceptions
---
{{WebAssemblySidebar}}

WebAssembly (WASM) supports exception handling, allowing WASM modules to throw and handle exceptions internally. WASM modules can also choose to interoperate with JavaScript code by sharing "tags" that allow the values of exceptions to be inspected.
This allows WASM code to handle WASM exceptions thrown by imported JavaScript functions, or to propagate exceptions for handling by JavaScript.

WASM and JavaScript code can also catch WASM exceptions (and JavaScript errors) for which doesn't have a tag.
However in this case the catching code will not be able to inspect the exception values and may simply rethrow it. 

WASM added support for exceptions in Firefox 100/Chrome 95 (for other platforms see the [`WebAssembly:Exception` browser compatibility table](/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Exception#browser_compatibility)).

> **Note:** Prior to these versions there was no exception handling within WASM modules.
> JavaScript could however catch errors thrown when importing WASM modules ({{jsxref("WebAssembly.CompileError()")}}, {{jsxref("WebAssembly.LinkError()")}}), and errors thrown when WASM code encountered an unrecoverable error, referred to as a "trap" ({{jsxref("WebAssembly.RuntimeError()")}}).


## What's a WASM Exception?

The heart of WASM exceptions is a _tag_, which is either



- `(tag)` definition or import - create or import a `WA.Tag` into the module
- `throw $tag` - construct a `WA.Exception` with the provided `WA.Tag`. This instruction takes as inputs the values that the tag requires.
- `catch $tag` - This catch will handle any thrown `WA.Exception` with tag matching `$tag`.
   The `WA.Exception` can be from a wasm `throw $tag` instruction, or a from JS constructing and throwing a `WA.Exception` manually with the correct tag.
   This catch will unwrap the values contained in the `WA.Exception` as based off of the tag.
- `catch_all` - This catch will handle any thrown exception. It can be a `WA.Exception`, a `JS Error`, or any value that JS may throw.
  This catch does not provide any value for the caught exception, as it might be something that wasm doesn't understand.
So (1, 2, 3) all operate on `WA.Exception` and `WA.Tag`. (4) may handle general JS values that are thrown.

prior to this ou 

This article explain how exception handling works in WebAssembly code;
how exceptions are defined, thrown and caught, and how they propagate across boundaries into the embedding environment.


<!-- Exception handling is a mechanism that separates code that detects and handles exceptional circumstances from the rest of your program. Note that an exceptional circumstance is not necessarily an error. When a function detects an exceptional situation, you represent this with an object. -->

Exception handling allows code to break control flow when an exception is thrown.
The exception can be any exception known by the WebAssembly module, or it may an unknown exception that was thrown by a called imported function.

To access exceptions, WebAssembly provides instructions to check if the exception is one that WebAssembly understands.
If so, the data of the WebAssembly exception is extracted and copied onto the stack, allowing succeeding instructions to process the data.

A WebAssembly exception is created when you throw it with the `throw` instruction.
Thrown exceptions are handled as follows:

1. They can be caught by one of `catch`/`catch_all` blocks in an enclosing try block of a function body.
1. Throws not caught within a function body continue up the call stack, popping call frames, until an enclosing try block is found.
1. If the call stack is exhausted without any enclosing try blocks, the embedder defines how to handle the uncaught exception.


-----

With exception handling we can now have exceptions in WASM. These are "generic" and are thrown to JS as WebAssembly:Exception.
These can be trapped in WASM code and rethrown.
These can be thrown in JS code called from WASM and caught in WASM.

Yes. The core of wasm exception-handling is four components:

- `(tag)` definition or import - create or import a `WA.Tag` into the module
- `throw $tag` - construct a `WA.Exception` with the provided `WA.Tag`. This instruction takes as inputs the values that the tag requires.
- `catch $tag` - This catch will handle any thrown `WA.Exception` with tag matching `$tag`.
   The `WA.Exception` can be from a wasm `throw $tag` instruction, or a from JS constructing and throwing a `WA.Exception` manually with the correct tag.
   This catch will unwrap the values contained in the `WA.Exception` as based off of the tag.
- `catch_all` - This catch will handle any thrown exception. It can be a `WA.Exception`, a `JS Error`, or any value that JS may throw.
  This catch does not provide any value for the caught exception, as it might be something that wasm doesn't understand.
So (1, 2, 3) all operate on `WA.Exception` and `WA.Tag`. (4) may handle general JS values that are thrown.

You can think of it as wasm really only understanding native `WA.Exception`'s and `WA.Tag`'s in it's throw-catch instructions, but being able to interop with general JS values being thrown via catch_all. And JS (via the WebAssembly JS-API) can use WA.Exception/Tag to interop if it wants.

> before exception handling WebAssembly code would throw specific errors: WebAssembly.CompileError, WebAssembly.LinkError, WebAssembly.RuntimeError.


`WebAssembly.CompileError`/`LinkError` are normal JS errors (instanceof Error) thrown when constructing wasm module or instance. There is nothing special with them. As they are not a `WA.Exception`, they cannot be caught in a wasm tagged catch, but may be caught in a wasm catch_all.

`WebAssembly.RuntimeError` is the error thrown when 'wasm traps'. A wasm 'trap' is generated when a wasm instructions goes 'wrong' such as with divide by zero or memory out of bounds. Wasm traps are specified to not be catchable by wasm tagged catch or catch_all. They may be caught by JS try-catches.

So when a `WA.RuntimeError` is thrown due to a wasm trap, a special flag is attached to the `WA.RuntimeError` to indicate it's from a wasm trap. This disqualifies it from being caught in a wasm try-catch_all. If instead the `WA.RuntimeError` is created and thrown from JS code, it's catchable in a wasm try-catch_all.

This is the weirdest part of wasm exception-handling.

> a tag defines the type or structure of exceptions in WASM. It just consists of a sequence of data types and an attribute. Its telling you that at some point an exception might be thrown that has data fields of these data-types.

Yes, a WA.Tag is the 'type' of a WA.Exception. The most important thing is the sequence of data types the WA.Exception will contain. A WA.Tag also has an identity. Two WA.Tag objects constructed with the same sequence of data types are not interchangeable.

This allows for a wasm module to define it's own private (not exported) WA.Tag for its use. This prevents any code outside of that module from being able to inspect the contents of a WA.Exception thrown from that module as that always requires providing the tag it was constructed with.

The attribute can currently just indicate "an exception thrown from javascript (or not)", but might indicate something else in future.

> I'm a bit confused by this because I would have thought the "where is this thrown from" would belong to the runtime exception not the tag.

What attribute is this? WA.Tags only have 'identity' and 'sequence of data types'.

The only place where 'exception thrown for JS or not' matters is with the weirdness around WA.RuntimeError being from a wasm trap or from JS code.

The tag is represented in JavaScript by WebAssembly:Tag, so when you create a tag you're creating a type of exception. You have to pass this to the module so it can be actually used by WASM.

A tag can be created in JS via:

let tag = new WebAssembly.Tag(...)
Or by being defined in a wasm module via:

(module (tag $tagName))
A tag defined in JS may be given to a wasm module by being imported ((module (import (tag i32)))).
A tag defined in wasm may be given to JS by being exported ((module (export $tagName))).

As tags have identity, sometimes a tag needs to be defined in JS and imported into a module so that they can share access to created WA.Exceptions.

> I guess my problem is that "type of exception" is a bit abstract. It seems that the wasm and the javascript that calls it will have to have an intimate knowledge of each other to understand how to use the types of exceptions that they throw.

Yes, that's roughly right.

If JS wants to inspect WA.Exceptions thrown from a wasm module, it needs the tags used by the module.
If wasm wants to inspect exceptions thrown from JS code, the JS code needs to use WA.Exceptions with a tag that the module has access to.

Wasm/JS can always catch and rethrow any of each other's exceptions even if they are not WA.Exceptions or don't have the WA.Tag. But to access the contents of the exception, they need to both be using WA.Exception with shared tags.


> 

So to get back and try to clarify this. A WA.Exception can be thrown in two ways:

In wasm via throw instruction ((func throw $tag))
In JS via throw new WebAssembly.Exception(tag, params, options)
Any WA.Exception created/thrown via (1) will not have a .stack property attached to the object.
A WA.Exception created/thrown via (2) will have a .stack property if options contains traceStack = true.

This was a compromise solution to allow toolchains targeting wasm to opt-in to stack trace collection (which can be expensive) by using the JS-API to throw their WA.Exceptions instead of using the wasm throw instruction.

Again, as WA.Tags have identities, the wasm module and JS code would need to share a tag in order for this to work. But it's expected that this isn't a burden for toolchain generated code.


-----

# Exception handling

This explainer reflects the up-to-date version of the exception handling
proposal agreed on [the CG meeting on
09-15-2020](https://github.com/WebAssembly/meetings/blob/master/main/2020/CG-09-15.md).

---

## Overview



### Exception handling

This proposal adds exception handling to WebAssembly. Part of this proposal is
to define a new section to declare exceptions. However, rather than limiting
this new section to just defining exceptions, it defines a more general format
`tag` that allows the declaration of other forms of typed tags in future.

WebAssembly tags are defined in a new `tag` section of a WebAssembly module. The
tag section is a list of declared tags that are created fresh each time the
module is instantiated.

Each tag has an `attribute` and a `type`. Currently, the attribute can only
specify that the tag is for an exception. In the future, additional attribute
values may be added when other kinds of tags are added to WebAssembly.

To allow for such a future extension possibility, we reserve a byte in the
binary format of an exception definition, set to 0 to denote an exception
attribute.

### Exceptions

An `exception tag` is a value to distinguish different exceptions, while an
`exception tag index` is a numeric name to refer to an (imported or defined)
exception tag within a module (see [tag index space](#tag-index-space) for
details). Exception tags are declared in the tag and import sections of a
module.

An `exception` is an internal construct in WebAssembly that represents a runtime
object that can be thrown. A WebAssembly exception consists of an exception tag
and its runtime arguments.

The type of an exception tag is denoted by an index to a function signature
defined in the `type` section. The parameters of the function signature define
the list of argument values associated with the tag. The result type must be
empty.

Exception tag indices are used by:

1. The `throw` instruction which creates a WebAssembly exception with the
   corresponding exception tag, and then throws it.

2. The `catch` clause uses the tag to identify if the thrown exception is one it
   can catch. If true it pushes the corresponding argument values of the
   exception onto the stack.

### Try-catch blocks

A _try block_ defines a list of instructions that may need to process exceptions and/or clean up state when an exception is thrown.
Like other higher-level constructs, a try block begins with a `try` instruction, and ends with an `end` instruction. That is, a try-catch block is sequence of instructions having the following form:

```
try blocktype
  instruction*
catch i
  instruction*
catch j
  instruction*
...
catch_all
  instruction*
end
```

A try-catch block contains zero or more `catch` blocks and zero or one
`catch_all` block. All `catch` blocks must precede the `catch_all` block, if
any. The `catch`/`catch_all` instructions (within the try construct) are called
the _catching_ instructions. There may not be any `catch` or `catch_all` blocks
after a `try`, in which case the `try` block does not catch any exceptions.

The _body_ of the try block is the list of instructions before the first
catching instruction. The _body_ of each catch block is the sequence of
instructions following the corresponding catching instruction before the next
catching instruction (or the `end` instruction if it is the last catching
block).

The `catch` instruction has an exception tag associated with it. The tag
identifies what exceptions it can catch. That is, any exception created with the
corresponding exception tag. Catch blocks that begin with a `catch` instruction
are considered _tagged_ catch blocks.

The last catching instruction of a try-catch block can be the `catch_all`
instruction. If it begins with the `catch_all` instruction, it defines the
_default_ catch block. The default catch block has no tag index, and is used to
catch all exceptions not caught by any of the tagged catch blocks. The term
'catch block' refers to both `catch` and `catch_all` blocks.

When the program runs `br` within `catch` or `catch_all` blocks, the rest of
the catching blocks will not run and the program control will branch to the
destination, as in normal blocks.

Try blocks, like control-flow blocks, have a _block type_. The block type of a
try block defines the values yielded by evaluating the try block when either no
exception is thrown, or the exception is successfully caught by the catch block.
Because `try` and `end` instructions define a control-flow block, they can be
targets for branches (`br` and `br_if`) as well.

### Throwing an exception

The `throw` instruction takes an exception tag index as an immediate argument.
That index is used to identify the exception tag to use to create and throw the
corresponding exception.

The values on top of the stack must correspond to the type associated with the
exception tag. These values are popped off the stack and are used (along with
the corresponding exception tag) to create the corresponding exception. That
exception is then thrown.

When an exception is thrown, the embedder searches for the nearest enclosing try
block body that execution is in. That try block is called the _catching_ try
block.

If the throw appears within the body of a try block, it is the catching try
block.

If a throw occurs within a function body, and it doesn't appear inside the body
of a try block, the throw continues up the call stack until it is in the body of
an an enclosing try block, or the call stack is flushed. If the call stack is
flushed, the embedder defines how to handle uncaught exceptions. Otherwise, the
found enclosing try block is the catching try block.

A throw inside the body of a catch block is never caught by the corresponding
try block of the catch block, since instructions in the body of the catch block
are not in the body of the try block.

Once a catching try block is found for the thrown exception, the operand stack
is popped back to the size the operand stack had when the try block was entered
after possible block parameters were popped.

Then in case of a try-catch block, tagged catch blocks are tried in the order
they appear in the catching try block, until one matches. If a matched tagged
catch block is found, control is transferred to the body of the catch block, and
the arguments of the exception are pushed back onto the stack.

Otherwise, control is transferred to the body of the `catch_all` block, if any.
However, unlike tagged catch blocks, the constructor arguments are not copied
back onto the operand stack.

If no tagged catch blocks were matched, and the catching try block doesn't have
a `catch_all` block, the exception is rethrown.

If control is transferred to the body of a catch block, and the last instruction
in the body is executed, control then exits the try block.

If the selected catch block does not throw an exception, it must yield the
value(s) specified by the type annotation on the corresponding catching try
block.

Note that a caught exception can be rethrown using the `rethrow` instruction.

### Rethrowing an exception

The `rethrow` instruction can only appear in the body of a catch/catch_all
block. It always re-throws the exception caught by an enclosing catch block.

Associated with the `rethrow` instruction is a _label_. The label is used to
disambiguate which exception is to be rethrown, when inside nested catch blocks.
The label is the relative block depth to the corresponding try block for which
the catching block appears.

For example consider the following:

```
try $l1
  ...
catch  ;; $l1
  ...
  block
    ...
    try $l2
      ...
    catch  ;; $l2
      ...
      try $l3
        ...
      catch  ;; $l3
        ...
        rethrow N  ;; (or label)
      end
    end
  end
  ...
end
```

In this example, `N` is used to disambiguate which caught exception is being
rethrown. It could rethrow any of the three caught expceptions. Hence, `rethrow
0` corresponds to the exception caught by `catch 3`, `rethrow 1` corresponds to
the exception caught by `catch 2`, and `rethrow 3` corresponds to the exception
caught by `catch 1`. In wat format, the argument for the `rethrow` instructions
can also be written as a label, like branches. So `rethrow 0` in the example
above can also be written as `rethrow $l3`.

Note that `rethrow 2` is not allowed because it does not refer to a `try` label
from within its catch block. Rather, it references a `block` instruction, so it
will result in a validation failure.

Note that the example below is a validation failure:
```
try $l1
  try $l2
    rethrow $l2  ;; (= rethrow 0)
  catch
  end
catch
end
```
The `rethrow` here references `try $l2`, but the `rethrow` is not within its
`catch` block.

### Try-delegate blocks

Try blocks can also be used with the `delegate` instruction. A try-delegate
block contains a `delegate` instruction with the following form:

```
try blocktype
  instruction*
delegate label
```

The `delegate` clause does not have an associated body, so try-delegate blocks
don't have an `end` instruction at the end. The `delegate` instruction takes a
try label and delegates exception handling to a `catch`/`catch_all`/`delegate`
specified by the `try` label. For example, consider this code:

```
try $l0
  try
    call $foo
  delegate $l0  ;; (= delegate 0)
catch
  ...
catch_all
  ...
end
```

If `call $foo` throws, searching for a catching block first finds `delegate`,
and because it delegates exception handling to catching instructions associated
with `$l1`, it will be next checked by the outer `catch` and then `catch_all`
instructions.

`delegate` can also target `catch`-less `try`s or non-`try` block constructs
like `block`s or `loop`s, in which case the delegated exception is assumed to
propagate to the outer scope and will be caught by the next matching
try-catches, or rethrown to the caller if there is no outer try block. In the
examples, catches are annotated with `($label_name)` to clarify which `try` it
belongs to for clarification; it is not the official syntax.
```
try $l0
  block $l1
    try
      call $foo
    delegate $l1  ;; delegates to 'catch ($l0)'
  end
catch ($l0)
end
```

Like branches, `delegate` can only target outer blocks, and effectively
rethrows the exception in that block. Consequently, delegating to a specific
`catch` or `catch_all` handler requires targeting the respective label from
within the associated `try` block. Delegating to a label from within a `catch`
block does delegate the exception to the next enclosing handler -- analogous to
performing a `throw` within a `catch` block, that handler is no longer active
at that point. Here is another example:

```
try $l0
  try $l1
  catch ($l1)
    try
      call $foo
    delegate $l1  ;; delegates to 'catch ($l0)'
  catch_all
    ...
  end
catch ($l0)
```

Here the `delegate` is targeting `catch ($l1)`, which exists before the
`delegate`. So in case an exception occurs, it propagates out and ends up
targetting `catch ($l0)`, if the catch has a matching tag. If not, it will
propagate further out. Even if the `catch_all` is below the `delegate`,
`delegate` targets catches of a `try` as a whole and does not target an
individual `catch`/`catch_all`, so it doesn't apply.

If `delegate` targets the implicit function body block, then in effect it
delegates the exception to the caller of the current function. For example:
```
(func $test
  try
    try
      call $foo
    delegate 1  ;; delegates to the caller
  catch
    ...
  catch_all
    ...
  end
)
```
In case `foo` throws, `delegate 1` here delegates the exception handling to the
caller, i.e., the exception escapes the current function. If the immediate is
greater than or equal to the number of block nesting including the implicit
function-level block, it is a validation failure. In this example, any number
equal to or greater than 2 is not allowed.

The below is an example that includes all the cases explained. The numbers
within `()` after `delegate`s are the label operands in immediate values.
```
(func $test
  try $lA
    block $lB
      try $lC
        try
        delegate $lC (0)  ;; delegates to 'catch ($lC)'
        try
        delegate $lB (1)  ;; $lB is a block, so delegates to 'catch ($lA)'
        try
        delegate $lA (2)  ;; delegates to 'catch ($lA)'
        try
        delegate 3        ;; propagates to the caller
        try
        delegate 4        ;; validation failure
      catch ($lC)
        try
        delegate $lC (0)  ;; 'catch ($lC)' is above this instruction,
                          ;; so delegates to 'catch ($lA)'
        try
        delegate $lB (1)  ;; $lB is a block, so delegates to 'catch ($lA)'
        try
        delegate $lA (2)  ;; delegates to 'catch ($lA)'
        try
        delegate 3        ;; propagates to the caller
        try
        delegate 4        ;; validation failure
      end  ;; try $lC
    end  ;; block $lB
  catch ($lA)
  end  ;; try $lA
)
```

### JS API

#### Traps

The `catch`/`catch_all` instruction catches exceptions generated by the `throw`
instruction, but does not catch traps. The rationale for this is that in general
traps are not locally recoverable and are not needed to be handled in local
scopes like try-catch.

The `catch` instruction catches foreign exceptions generated from calls to
function imports as well, including JavaScript exceptions, with a few
exceptions:
1. In order to be consistent before and after a trap reaches a JavaScript frame,
   the `catch` instruction does not catch exceptions generated from traps.
1. The `catch` instruction does not catch JavaScript exceptions generated from
   stack overflow and out of memory.

Filtering these exceptions should be based on a predicate that is not observable
by JavaScript. Traps currently generate instances of
[`WebAssembly.RuntimeError`](https://webassembly.github.io/reference-types/js-api/#exceptiondef-runtimeerror),
but this detail is not used to decide type. Implementations are supposed to
specially mark non-catchable exceptions.
([`instanceof`](https://tc39.es/ecma262/#sec-instanceofoperator) predicate can
be intercepted in JS, and types of exceptions generated from stack overflow and
out of memory are implementation-defined.)

#### API additions

The following additional classes are added to the JS API in order to allow
JavaScript to interact with WebAssembly exceptions:

  * `WebAssembly.Tag`
  * `WebAssembly.Exception`.

The `WebAssembly.Tag` class represents a typed tag defined in the tag section
and exported from a WebAssembly module. It allows querying the type of a tag
following the [JS type reflection
proposal](https://github.com/WebAssembly/js-types/blob/master/proposals/js-types/Overview.md).
Constructing an instance of `Tag` creates a fresh tag, and the new tag can be
passed to a WebAssembly module as a tag import.

In the future, `WebAssembly.Tag` may be used for other proposals that require a
typed tag and its constructor may be extended to accept other types and/or a tag
attribute to differentiate them from tags used for exceptions.

The `WebAssembly.Exception` class represents an exception thrown from
WebAssembly, or an exception that is constructed in JavaScript and is to be
thrown to a WebAssembly exception handler. The `Exception` constructor accepts a
`Tag` argument and a sequence of arguments for the exception's data fields. The
`Tag` argument determines the exception tag to use. The data field arguments
must match the types specified by the `Tag`'s type. The `is` method can be used
to query if the `Exception` matches a given tag. The `getArg` method allows
access to the data fields of a `Exception` if a matching tag is given. This last
check ensures that without access to a WebAssembly module's exported exception
tag, the associated data fields cannot be read.

The `Exception` constructor can take an optional `ExceptionOptions` argument,
which can optionally contain `traceStack` entry. When `traceStack` is `true`,
web VMs can attach a stack trace string to `Exception.stack` field, as in
JavaScript's `Error` class. While `Exception` is not a subclass of JavaScript's
`Error` and it can be used to represent normal control flow constructs,
`traceStack` field can be set when we use it to represent errors. The format of
stack trace strings conform to the [WebAssembly stack trace
conventions](https://webassembly.github.io/spec/web-api/index.html#conventions).
When `ExceptionOption` is not provided or it does not contain `traceStack`
entry, `traceStack` is considered `false` by default.

To preserve stack trace info when crossing the JS to Wasm boundary, exceptions
can internally contain a stack trace, which is propagated when caught by `catch`
and rethrown by `rethrow`.

More formally, the added interfaces look like the following:

```WebIDL
dictionary TagType {
  required sequence<ValueType> parameters;
};

[LegacyNamespace=WebAssembly, Exposed=(Window,Worker,Worklet)]
interface Tag {
  constructor(TagType type);
  TagType type();
};

dictionary ExceptionOptions {
  boolean traceStack = false;
};

[LegacyNamespace=WebAssembly, Exposed=(Window,Worker,Worklet)]
interface Exception {
  constructor(Tag tag, sequence<any> payload, optional ExceptionOptions options);
  any getArg(Tag tag, unsigned long index);
  boolean is(Tag tag);
  readonly attribute (DOMString or undefined) stack;
};
```

`TagType` corresponds to a `FunctionType` in [the type reflection
proposal](https://github.com/WebAssembly/js-types/blob/main/proposals/js-types/Overview.md),
without a `results` property). `TagType` could be extended in the future for
other proposals that require a richer type specification.

## Changes to the text format

This section describes change in the [instruction syntax
document](https://github.com/WebAssembly/spec/blob/master/document/core/text/instructions.rst).

### New instructions

The following rules are added to *instructions*:

```
  try blocktype instruction* (catch tag_index instruction*)* (catch_all instruction*)? end |
  try blocktype instruction* delegate label |
  throw tag_index argument* |
  rethrow label |
```

Like the `block`, `loop`, and `if` instructions, the `try` instruction is
*structured* control flow instruction, and can be labeled. This allows branch
instructions to exit try blocks.

The `tag_index` of the `throw` and `catch` instructions denotes the exception
tag to use when creating/extract from an exception. See [tag index
space](#tag-index-space) for further clarification of exception tags.

## Changes to Modules document

This section describes change in the [Modules
document](https://github.com/WebAssembly/design/blob/master/Modules.md).

### Tag index space

The `tag index space` indexes all imported and internally-defined exception
tags, assigning monotonically-increasing indices based on the order defined in
the import and tag sections. Thus, the index space starts at zero with imported
tags, followed by internally-defined tags in the [tag section](#tag-section).

For tag indices that are not imported/exported, the corresponding exception tag
is guaranteed to be unique over all loaded modules. Exceptions that are imported
or exported alias the respective exceptions defined elsewhere, and use the same
tag.

## Changes to the binary model

This section describes changes in the [binary encoding design
document](https://github.com/WebAssembly/design/blob/master/BinaryEncoding.md).

#### Other Types

##### tag_type

We reserve a bit to denote the exception attribute:

| Name      | Value |
|-----------|-------|
| Exception | 0     |

Each tag type has the fields:

| Field | Type | Description |
|-------|------|-------------|
| `attribute` | `uint8` | The attribute of a tag. |
| `type` | `varuint32` | The type index for its corresponding type signature |

##### external_kind

A single-byte unsigned integer indicating the kind of definition being imported
or defined:

* `4` indicating a `Tag`
[import](https://github.com/WebAssembly/design/blob/main/BinaryEncoding.md#import-section) or
[definition](#tag-section)

### Module structure

#### High-level structure

A new `tag` section is introduced.

##### Tag section

The `tag` section comes after the [memory
section](https://github.com/WebAssembly/design/blob/master/BinaryEncoding.md#memory-section)
and before the [global
section](https://github.com/WebAssembly/design/blob/master/BinaryEncoding.md#global-section).
So the list of all sections will be:

| Section Name | Code | Description |
| ------------ | ---- | ----------- |
| Type | `1` | Function signature declarations |
| Import | `2` | Import declarations |
| Function | `3` | Function declarations |
| Table | `4` | Indirect function table and other tables |
| Memory | `5` | Memory attributes |
| Tag | `13` | Tag declarations |
| Global | `6` | Global declarations |
| Export | `7` | Exports |
| Start | `8` | Start function declaration |
| Element | `9` | Elements section |
| Code | `10` | Function bodies (code) |
| Data | `11` | Data segments |

The tag section declares a list of tag types as follows:

| Field | Type | Description |
|-------|------|-------------|
| count | `varuint32` | count of the number of tags to follow |
| type | `tag_type*` | The definitions of the tag types |

##### Import section

The import section is extended to include tag definitions by extending an
`import_entry` as follows:

If the `kind` is `Tag`:

| Field | Type | Description |
|-------|------|-------------|
| `type` | `tag_type` | the tag being imported |

##### Export section

The export section is extended to reference tag types by extending an
`export_entry` as follows:

If the `kind` is `Tag`:

| Field | Type | Description |
|-------|------|-------------|
| `index` | `varuint32` | the index into the corresponding tag index space |

##### Name section

The set of known values for `name_type` of a name section is extended as
follows:

| Name Type | Code | Description |
| --------- | ---- | ----------- |
| [Function](#function-names) | `1` | Assigns names to functions |
| [Local](#local-names) | `2` | Assigns names to locals in functions |
| [Tag](#tag-names) | `3` | Assigns names to tag types |

###### Tag names

The tag names subsection is a `name_map` which assigns names to a subset of
the tag indices (Used for both imports and module-defined).

### Control flow operators

The control flow operators are extended to define try blocks, catch blocks,
throws, and rethrows as follows:

| Name | Opcode | Immediates | Description |
| ---- | ---- | ---- | ---- |
| `try` | `0x06` | sig : `blocktype` | begins a block which can handle thrown exceptions |
| `catch` | `0x07` | index : `varint32` | begins the catch block of the try block |
| `catch_all` | `0x19` | | begins the catch_all block of the try block |
| `delegate` | `0x18` | relative_depth : `varuint32` | begins the delegate block of the try block |
| `throw` | `0x08` | index : `varint32` | Creates an exception defined by the tag and then throws it |
| `rethrow` | `0x09` | relative_depth : `varuint32` | Pops the `exnref` on top of the stack and throws it |

The *sig* fields of `block`, `if`, and `try` operators are block signatures
which describe their use of the operand stack.
