# JAJ - Just Another JAPL
Repository for the v2 implementation of [the japl programming language](http://github.com/japl-lang).

Once most of the changes from this version are merged (or well, replace) the current JAPL implementation
this repository will be yeeted into the void. This is just a way for me to start from scratch without any of the messy legacies of the current codebase, which lacks modularity and inherits way too much from the C code of Lox (see Credits)

## Credits

JAPL was inspired by Bob Nystrom's amazing [Crafting Interpreters](https://craftinginterpreters.com) book

## Additions and changes from clox

Clox is fine as a toy language, but its limitations are a little too strict for a language that aims to be usable in the real world (which is fine, because clox was meant to be simple). To make JAPL a complete language, then, I did some significant modifications to the design of clox. A non-exhaustine list of said changes follows (note that not all of these changes are implemented yet. They are all scheduled though):

- The compiler is now triple-pass: parsing, optimization and compilation. This allows us to do neat things like constant folding (meaning compiling expressions such as `2 * 2;` as a single constant instruction pushing `4` on the stack), emitting warnings for code after a return statement in a function, and much more. It also makes it easier to implement closures (no need for upvalues like Bob does in his book) because the compiler knows which names are closed over and in which functions they are used _before_ compiling their declarations
- Any declaration now has the ability to be made module-local or public (`private` or `public`, with the former being the default) and statically or dynamically bound (`static` or `dynamic`, again with the former being the default behavior), _including globals!_. This makes most of the name resolution as fast as indexing in our stack, except for the cases when the user decides otherwise. For statically resolved globals to work though, they _must_ have stack semantics, meaning they cannot be used before definition. Mutually recursive functions and closures are prime examples of cases when globals don't have stack semantics and should be declared as `dynamic` instead. Some other notes about the new declaration system is the correct syntax is `public static var x = nil;`, meaning the visibility modifier goes first, then goes the name resolution modifier and finally the declaration as normal (`var x;` is desugared to `private static var x;` automatically). Since names are now bound to the module they're declared into, attempting to access one declared as `private` from outside the module itself will raise an error (at compile time if the name is declared as `static` and at runtime otherwise).

    __Note__: Some languages support so-called "static methods", basically methods that are bound to a _class_ instead of an _instance_ and therefore take no self parameter. It might cause confusion for some people to see a `static fun` inside a class take a self argument, but that's normal: JAPL doesn't support static methods (because they're worthless). The keyword `static` in JAPL has no other meaning than "resolve this name's location at compile time": it is merely a compile-time optimization to speed up name lookup since hash tables are slow

- Loops can be interrupted via `break` and an iteration can be skipped using `continue`. Although JAPL has a triple-pass compiler, we still use the book's method of patching jump offsets because otherwise we'd have to predict how many instructions a given AST node compiles into (something that JAPL might do in the future, though)
- Classes support multiple inheritance. Methods are resolved starting from the first parent until the last one, in the order in which they are listed in the class declaration
- Proper exception support, with `try`/`except` handlers (with `finally` and `else` support as well), and obviously `raise`, has been added. The VM will keep a list of active exception hooks which are created with the `BeginTry` instruction. When an exception is raised, the VM will traverse this list backwards to find any matching exception hook according to a set of rules (i.e. taking superclasses and multiple catching into account) and execute their associated code if they do. If the VM can't find any matching handler and it gets to the top of the list, it then writes the message to stderr and exits. Some niceties like `except (exc1, exc2, exc2)` and `exc SomeExc as excName` (`except (SomeExc, Exc2) as name` is also valid) have also been blatantly stolen from python's exception handling system
- JAPL will have an iterator protocol, hence a `foreach` loop has been added to iterate over collections and sequence types
- Closures are now different from regular functions. JAPL will compile a function to be a closure only if it makes use of stack variables that are outside its own scope (which means that most functions _won't_ be closures). In this case, as the book rightly suggests: _"[...] The next easiest approach, then, would be to take any local variable that gets closed over and have it always live on the heap. When the local variable declaration in the surrounding function is executed, the VM would allocate memory for it dynamically. That way it could live as long as needed."_
- Builtin collections similar to Python's have been added: lists, tuples, sets and dictionaries. A notable quirk though is that since brackets are used both for set and dictionary literals and for block statements, and due to the latter's precedence being higher, a bare `{};` creates an empty block scope and leaves a dangling semicolon instead of creating a dictionary object and discarding it immediately. Set and dictionary literals can only be defined where an expression is expected (which is fine, because they _are_ expressions), so something like `var a = {1, 2, 3};` is perfectly fine and not ambiguous because the parser only expects expression as values for variable declarations, and block statements are, well, statements
- JAPL supports `yield` statements and expressions, allowing on-the-fly value generation for improved iteration performance and highly-efficient O(1) algorithms, the most basic being infinite counters
- Some handy operators have been added: `is` (and its opppsite `isnot`) checks if two objects refer to the same value, `A of B` returns `true` if A is a subclass of B and `A as B` will call `B(A)` allowing for simple casting, like `"55" as Integer;`, which pushes the Integer `55` onto the stack
- The keyword `this` has been removed and instead JAPL passes the instance's value as the first argument to a bound method (commonly named `self`)
- JAPL's type system is much more flexible than lox's: everything is an object, including builtins, hence no more `only instances have properties` errors, because all entities in JAPL are essentially an instance of some type, all the way up to `BaseObject` which is the base of itself and is merely an implementation detail
- When optimizations are enabled (i.e. always unless explicitly disabled, if you like slow code I guess) the compiler will emit optimized instructions for a few edge cases. For example, it will emit a specialized `PopN` instruction that pops n values off the stack when compiling local scopes because those usually pop a lot of values when discarding local variables. Another small optimization is used when compiling if statements, which will cause the compiler to emit a `JumpIfFalsePop` instruction instead of separate `JumpIfFalse` and a `Pop` instructions. JAPL will also reuse existing entries in the constant table by default, so don't be surprised if two separate `LoadConstant` instructions point to the same constant index: it just means the compiler had already seen that constant and just reused it
- JAPL now has long jump instructions, which use 24-bit operands instead of 16-bit ones to allow to jump even further if one wants to jump more than 65535 instructions into the code (which is not unlikely in real-world scenarios)
- Private attributes are not very useful without an import system and modules, a topic that clox doesn't touch at all (probably because it's not that interesting implementation-wise and is only a recipe for trouble in a beginner's book), so JAPL fixes that by having a pretty Python-esque import system with things like `import name;`, `import module.submodule;`, `from module import someComponent;`, `import someModule as someName;`, `import a, b, c;` and basically all variations of the above. Imports in JAPL are "proper", i.e. they don't just copy-paste code like C's `#include` or some other lox implementations: they create a separate namespace and populate it with whatever the imported module decides to export (i.e. all declarations marked as `public`)
- Inline comments in JAPL start with an hashtag and that's the only kind of comment that exists. This is because `//` is used as the binary operator for integer (aka floor) division
- Support for in-place operations has been added (`+=`, `-=`, etc)
- Multithreading and multiprocessing support has been (or more like will be) added. Multiprocessing is easy: just fork() (or CreateProcess on windows) and let the new VM do its thing. Since processes have entirely separate address spaces, there's no race conditions to handle. Multithreading is a bit trickier, requiring a global VM lock. But wait! Unlike Python's GIL, which locks all but one thread from running bytecode at a time, JAPL's lock is only acquired during a garbage collection cycle, which minimizes interference with other threads and lets JAPL achieve true concurrency. Also, due to how JAPL's garbage collector is implemented, collection cycles become rarer and rarer as more and more memory is allocated, which further minimizes the pauses the GC has to issue while it reclaims memory. Since we're on the subject of memory management and we mentioned the fact that JAPL has an exception system, it's worth adding that the GC will try to raise an `OutOfMemoryException` when it runs out of memory (assuming there's enough memory for the exception itself to be allocated, which requires just a few dozen bytes. If it can't do even that, it will just shut down the VM entirely, free every object that it is managing, and print an error message on stderr)
- JAPL will support native `async` functions using the coroutines model (like Python does) which can be called via `await`. This also allows me to experiment with writing an asynchronous scheduler, if [my other project](https://github.com/giambio) wasn't enough
- JAPL supports `const` declarations, which emit simple `LoadConstant` instructions. For this reason, name resolution specifiers do not apply to constant declarations and they have to be assigned at declaration time using a constant type (i.e. a number or a string). The compiler statically checks assignment to constants and spits out compile errors if it finds an attempt to modify a constant's value


__Note__: This list is likely to be expanded/modified a lot as I iron down some of the quirks and kinks of designing a language that people would actually want to use that is also decently fast (being _any_ amount faster than Python, ideally 2x or more, is the first goal of the project) while also being entirely type safe, so don't rely on it too much nor be sad if some feature is missing: I might've just forgotten I want to add it or just not gotten around to laying down its design yet. Also, if you haven't noticed by now, I'm a huge python fan: most of the design for JAPL in fact comes from Python, while some other features come from Nim (the _amazing_ language JAPL is written in)
