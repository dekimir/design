#Debug Symbol Integration

This is a design proposal for providing debug information within WebAssembly
that would allow source-level debugging of code compiled into WebAssembly.

## Goals and Principles

The immediate goal is to implement enough debugging functionality in WebAssembly
MVP to let users perform at least rudimentary debugging on their source.
Another goal is not to bake in any limitations that would prevent reasonable
future evolution of the standard.

Specifically, we are currently not focused on identifying a language-independent
subset of the debug format -- this proposal bundles language-dependent and
language-independent parts together.

META: We need an evolution strategy that allows new front-end/debugger pairs to
use the format in the future to transfer information currently unanticipated.
Examples: a) dynamic scoping in Lisp; b) full DWARF 4 equivalence.

## Debug Info Goes into Extra Sections

To allow easy stripping of debug info, all of it will go into a separate
section.  In the binary format, this will be an unknown section inserted into
the wasm file.  Because this new info will make the current
[name section](BinaryEncoding.md#name-section) redundant, we propose deprecating
the name section.

META: How do we distinguish the debug-containing unknown section from any other
unknown section?

In the text format, this will be an extra child of the **module** element named
**debug**.

META: If there's a better name than **debug**, we can certainly use it.  Also,
currently undecided about allowing multiple **debug** children, like **func**.

## AST Gets Unique Identifiers

Since all the debug info is in a separate section, there must be a way for it to
reference AST nodes.  For that, we introduce a mechanism to tag any AST node
with a unique identifier: just add an atom to the node's s-expression.  That
atom must begin with the character '@' and be unique across the entire module
(for instance: `(get_local $x @abc123)` or `(param $y f64 @def456)`).

This extension is not visible in the binary format.  In binary, the debug
section can refer to the wasm code by its byte address.

## Debug Functions

The **debug** child of the **module** will contain invocations of special
functions defined in this section.  These functions are intended to convey the
debug info about the AST.

The functions' syntax is given using conventions borrowed from
[here](https://www.gnu.org/software/emacs/manual/html_node/elisp/A-Sample-Function-Description.html#A-Sample-Function-Description).
Any _symbolID_ argument must be unique across the module.

### File

Function: **file** _symbolID_  _stringFilepath_

Describes a file that can be referenced by _symbolID_.  The file contents can be
read by accessing _stringFilepath_ in a manner described elsewhere.  No two
**file** invocations may have the same _stringFilepath_.

META: Specify how exactly the browser may obtain the file content.

### Source Location

Function: **source\_location** _symbolID_  _symbolFile_  _integerLine_  _integerColumn_

Describes a source location that can be referenced by _symbolID_.  _symbolFile_
refers to a file described by the **file** function.  The last two arguments are
line and column in that file.

Function: **has\_source\_location** _symbolLocation_ &rest _@nodes_

Declares that _@nodes_ (an arbitrary number of @ references to the AST)
correspond to the source location _symbolLocation_.

### Lexical Context

Function: **context** _symbolID_  _symbolStart_  _symbolEnd_

Describes a set of consecutive lines in a file, starting at location
_symbolStart_ and endint at location _symbolEnd_.  Both locations must be in the
same file, and the end location must come lexically after the start.

This can be used to describe lexical scopes in C-like program, with the start
location pointing to the opening brace and the end location pointing to the
closing brace.

### Type

Function: **type** _symbolID_  _symbolLocation_

TODO: Devise a convention to represent types.  OK if it's specific to C++.

### Variable

Function: **var** _symbolID_  _stringName_  _symbolContext_  _symbolType_  _integerMemoryAddress_

Describes a source-code variable that can be referenced by _symbolID_.  The
variable name (as used in the source) is captured in _stringName_, its lexical
scope in _symbolContext_ (refers to a _symbolID_ of a **context** invocation),
and its type in _symbolType_ (refers to a _symbolID_ of a **type** invocation).
_integerMemoryAddress_ is the address of module's memory that holds the value of
this source variable.

TODO: Capture the source location where the variable is declared? Isn't the
context redundant then?

### Function

Function: **func** _symbolID_  _stringName_  _symbolType_  _symbolStart_  _symbolEnd_

Describes a source-code function or method whose mangled name is _stringName_,
whose type is the **type** referenced by _symbolType_, and whose start/end
locations are referenced by **source\_location** references _symbolStart_ and
_symbolEnd_.

## How to Perform Debugging Actions

Here's how to perform some basic debugging actions based on this debug-info
format:

### Set a Breakpoint on a Specific Line of a Specific File

1. Find the **file** invocation whose _stringFilepath_ argument matches the
   specified file.  If none exist, abort with an error "no such file".
2. Find all **source\_location** invocations whose _symbolFile_ equals the
   above's _symbolID_.  Among them, find invocations with the minimal
   _integerLine_ equal or larger than the specified line.  If none exist, abort
   with an error "no debuggable code on that line".
3. Find all the **has\_source\_location** invocations whose _symbolLocation_ is
   among the _symbolID_ arguments to the above invocations.  If none exist,
   abort with an error "no debuggable code on that line".
4. Find the earliest (in evaluation order) AST node in the union of the above
   invocations' _@nodes_.  This is where the breakpoint goes.

### Set a Breakpoint on a Specific Function

1. Find the **func** invocation whose _stringName_ matches (META: describe the
   matching process, including mangling) specified function name.  If none
   exist, abort with an error "unknown function".  If multiple invocations
   match, ask the user to choose one among them.
2. Find the **source\_location** invocation whose _symbolID_ equals the
   _symbolStart_ of the above **func**.  If none exists, abort with an error
   "incomplete debug info".
3. Find the **file** invocation whose _symbolID_ equals the _symbolFile_ of the
   above.
4. Proceed with setting a breakpoint on the file and line derived from the
   above's _stringFilepath_ and the source location's _integerLine_.  The
   procedure is described in the previous section.

### Set a Breakpoint Inside a C++ Template Definition

1. Find all instantiations of the template in question.
2. For each instantiation, repeat the above procedure for setting a breakpoint.

### Show which Source Line is Being Executed

1. Find the first **has\_source\_location** invocation whose _@nodes_ contains
   the current AST node's @ identifier. (META: should we disallow multiple
   locations for the same AST node?)  If none exists, the result is nil.
2. Find the **source\_location** invocation whose _symbolID_ equals the
   _symbolLocation_ of the above.  If none exists, the result is nil.
3. Find the **file** invocation whose _symbolID_ equals the _symbolFile_ of the
   above.  If none exists, abort with an error "incomplete debug info".
4. The result is _integerLine_ of the **source\_location** from 2. above, plus
   the _stringFilepath_ of the above.

### Step Into the Next Line (Following Function Calls)

1. Record the current source line as described in the previous section.
2. Execute the current AST node, then consider the next in evaluation order.
   Until the calculated line/file changes, repeat from 1. above.

### Step Over the Next Line (Skipping Function Calls)

Same as the above section, except finish evaluating entire **invoke** nodes
before recalculating the current line and file.

### Show a Specified Variable's Value

1. Record the currently executed source line as described in a prior section.
2. Find all **var** invocations whose _stringName_ matches the (mangled)
   specified variable name.  If none exist, abort with an error "unknown
   variable".
3. For each above invocation, find the **context** whose _symbolID_ equals the
   _symbolContext_ of the **var**.  Keep only those **var** invocations whose
   **context** envelopes the current line (ie, same file and current line is
   between the context's start and end).
4. If multiple **var** invocations remain after the above step, ask the user to
   choose one among them.
5. Find the **type** whose _symbolID_ equals _symbolType_ from the above.
6. Inspect the memory contents at the _integerMemoryAddress_, in accordance with
   the variable's type.

### Show a Specified Object's Field Value

Like the above section, but use the type structure information to find the
field's offset in memory.

### Show a Variable's Type

Like showing the variable's value, but show the **type** info instead.

### Set a Specified Variable's Value

Like showing the variable's value, but set the memory instead.

### Set the Current Function's Return Value

META: Describe this.

### Display Call Stack

META: Describe this.
