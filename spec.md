# Introduction

Val is a research language based on the principles of mutable value semantics (MVS) (Racordon et al. 2022) for safety and efficiency. It is designed to help developers write and maintain correct programs using powerful abstractions without loss of efficiency.

Val is strongly related to the [Swift programming language](https://www.swift.org) and borrows liberally from its syntax and semantics. Nonetheless, it exposes a more elaborated memory model to the user for a tighter control over memory, blending ideas found in other languages such as Rust, C++, and Go.

On a theoretical front, Val owes greatly to linear types [(Wadler 1990)](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.31.5002&rep=rep1&type=pdf), ownership types [(Clarke et al. 2013)](https://doi.org/10.1007/978-3-642-36946-9_3), and capability-based type systems [(Smith et al. 2000)](https://doi.org/10.1007/3-540-46425-5_24), while striving to hide the complexity inherent to these approaches. It does so by excluding first-class references from the user model and using continuations to mitigate the loss of expressiveness.

# Lexical conventions

## Program text

1. A Val program is written in text format. The text of a program is written using the [Unicode](https://home.unicode.org) character set and kept in units called source files. A source file is a sequence of Unicode Extended Grapheme clusters.

2. This document refers to individual Unicode characters with the notation `U+n` where `n` is a hexadecimal value representing a Unicode code point, and refers to the Unicode general categories to identify groups of Unicode characters.

## Lexical translations

1. A sequence of Unicode characters is translated into a sequence of tokens. The following rules apply during translation:

    1. Comments are treated as though they are a single space `U+20`.

    2. Spaces are ignored unless they appear between the opening and closing delimiters of a character or string literal. Unicode characters with the "White_Space" property are recognized as spaces.

    3. New-line delimiters are ignored unless they appear between the opening and closing delimiters of a character or string literal. Unicode characters with the "Line_Break" property are recognized as new-line delimiters.


## Comments

1. The character sequence `//` starts a single-line comment, which terminates immediately before the next new-line delimiter.

2. The character sequences `/*` and `*/` are multiline comment opening and closing delimiters, respectively. A multiline comment opening  delimiter starts a comment that terminates immediately after a matching closing delimiter. Each opening delimiter must have a matching closing delimiter. Multiline comments may nest, and need not contain any new-line characters.

3. The character sequences `//` have no special meaning in a multiline comment. The character sequences `/*` and `*/` have no special meaning in a single-line comment. The character sequences `//` and `/*` have no special meaning in a string literal. String and character literal delimiters have no special meaning in a comment.

## Tokens

1. A token is a terminal symbol of the syntactic grammar. It falls into one of five categories: scalar literals, keywords, identifiers, raw operators, and punctuators.

2. A token is associated with a three-valued flag specifying whether it was followed by a raw character, an inline space (including an inline space substituted for a comment), or a new-line delimiter in the source file.

3. (Example)

    The input "a << b" is translated to a sequence of 4 tokens: __identifier__, __raw-operator__, __raw-operator__, __identifier__. The first and third tokens are known to be followed by an inline space.

4. Unless otherwise specified, the token recognized at a given lexical position is the one having the longest possible sequence of characters.

### Scalar literals

#### Boolean literals

1. The Boolean literals are `true` and `false`:

    ```ebnf
    boolean-literal ::= (one of)
      true false
    ```

#### Integer literals

1. Integer literals have the form:

    ```ebnf
    integer-literal ::=
      binary-literal
      octal-literal
      decimal-literal
      hexadecimal-literal

    binary-digit ::= (one of)
      0 1 _

    binary-literal ::= (token)
      '0b' binary-digit+

    octal-digit ::= (one of)
      0 1 2 3 4 5 6 7 _

    octal-literal ::= (token)
      '0o' octal-digit+

    decimal-digit ::= (one of)
      0 1 2 3 4 5 6 7 8 9 _

    decimal-literal ::= (token)
      decimal-digit+

    hexadecimal-digit ::= (one of)
      0 1 2 3 4 5 6 7 8 9 A B C D E F a b c d e f _

    hexadecimal-literal ::= (token)
      '0x' hexadecimal-digit+
    ```

2. The sequence of digits of a literal are interpreted as follows, ignoring all occurrences of `_`:

    1. In a binary literal, as a base 2 integer.

    2. In an octal literal, as a base 8 integer.

    3. In a decimal literal, as a base 10 integer.

    4. In a hexadecimal literal, as a base 16 integer, where the characters `a` through `f` and `A` through `F` have decimal values ten through fifteen.

3. (Example)

    The integer literal `0o12_34_5__` is interpreted as the integer `5349` in base 10.

4. The default inferred type of an integer literal is the Val standard library `Int`, which represents a 64-bit signed integer. If the interpreted value of an integer literal is not in the range of representable values for its type, the program is ill-formed.

#### Floating-point literals

1. Floating-point literals have the form:

    ```ebnf
    floating-point-literal ::=
      decimal-floating-point-literal

    decimal-floating-point-literal ::= (token)
      decimal-fractional-constant exponent?
      decimal-literal exponent

    decimal-fractional-constant ::= (token)
      decimal-literal . decimal-literal

    exponent := (token)
      'e' exponent-sign? decimal-literal
      'E' exponent-sign? decimal-literal

    exponent-sign ::= (one of)
      + -
    ```

2. The significand of a floating-point literal is the __decimal-fractional-constant__ or the __decimal-literal__ preceding the __exponent__. In the significand, the digits and optional period are interpreted as a base `N` real number `s`, where `N` is 10, ignoring all occurrences of `_`. If __exponent__ is present, the exponent `e` of the floating-point-literal is the result of interpreting the sequence of an optional `sign` and the digits as a base 10 integer. Otherwise, the exponent `e` is 0. The scaled value of the literal is `s × 10e`.

3. The default inferred type of an integer literal is the Val standard library `Double`, which represents a 64-bit floating point number. If the interpreted value of a floating-point literal is not in the range of representable values for its type, the program is ill-formed. Otherwise, the value of a floating-point literal is the interpreted value if representable, else the larger or smaller representable value nearest the interpreted value, chosen in an implementation-defined manner.

#### Unicode scalar literals

1. Unicode scalar literals have the form:

    ```ebnf
    unicode-scalar-literal ::= (token)
      single-quote escape-char single-quote
      single-quote c-char single-quote

    escape-char ::=
      simple-escape
      unicode-escape

    simple-escape ::= (one of)
      \0 \t \n \r \' \"

    single-quote ::= (one of)
      '

    unicode-escape ::= (token)
      '\u' hexadecimal-digit+

    c-char ::= (regexp)
      [^']
    ```

2. The __hexadecimal-digit__  of a __unicode-escape__ represents a Unicode scalar value.

#### String literals

1. String literals have the form:

    ```ebnf
    string-literal ::=
      simple-string
      multiline-string

    simple-string ::= (token)
      '"' simple-quoted-text? '"'

    simple-quoted-text ::= (token)
      simple-quoted-text-item
      simple-quoted-text simple-quoted-text-item

    simple-quoted-text-item ::=
      escape-char
      s-char

    s-char ::= (regexp)
      [^"\x0a\x0d]
      
    multiline-string ::= (token)
      """ multiline-quoted-text """

    multiline-quoted-text ::= (token)
      multiline-quoted-text-item
      multiline-quoted-text multiline-quoted-text-item

    multiline-quoted-text-item ::=
      escape-char
      m-char

    m-char ::= (regexp)
      [^"]|"(?!"")
    ```

2. The first new-line delimiter in a multiline string literal is not part of the value of that literal if it immediately succeeds the opening delimiter. The last new-line delimiter that is succeeded by a contiguous sequence of inline spaces followed by the closing delimiter is called the indentation marker. The indentation marker and the succeeding inline spaces specify the indentation pattern of the literal and are not part of its value. The pattern is defined as the sequence of inline spaces between the indentation marker and the closing delimiter. That sequence must be homogeneous. If the literal has no indentation marker, its indentation pattern is an empty sequence. Each line of a multiline string literal must begin with the indentation pattern of that literal. That prefix is not part of the value of the literal.

3. (Example)

    ```val
    let s = """
        Hello,
        World!
      """
    print(s) // "  Hello,\n  World!"
    ```

### Keywords

1. Keywords are reserved identifiers. They have the form:

    ```ebnf
    keyword ::= (one of)
      Any Never as as! _as!! async await break catch conformance continue deinit else extension
      false for fun if import in infix init let match namespace nil postfix prefix private public
      return sink static true try type typealias var where while yield
    ```

### Identifiers

1. Identifiers are case-sensitive sequences of letters and digits. They have the form:

    ```ebnf
    identifier ::= (token)
      identifier-head identifier-tail*
      '`' bq-char+ '`'
      contextual-keyword

    identifier-head ::= (regexp)
      [_\p{Lu}\p{Ll}\p{Lt}\p{Lm}\p{Lo}\p{Nl}]

    identifier-tail ::= (regexp)
      [\p{Lu}\p{Ll}\p{Lt}\p{Lm}\p{Lo}\p{Mn}\p{Mc}\p{Nl}\p{Nd}\p{Pc}]
    
    bq-char ::= (regexp)
      [^`\x0a\x0d]
      
    contextual-keyword ::= (one of)
      mutating size any
    ```

2. Contextual keywords are identifiers that have a special meaning when appearing in a certain context. When referred to in the grammar, these identifiers are used explicitly rather than using the identifier grammar production.

### Raw operators

1. Raw operators have the form:

    ```ebnf
    raw-operator ::= (regexp)
      [-*/^%&!?\p{Sm}]
    ```

    [Note: The Unicode category Sm includes +, =, <, >, |, and ~.]

# General concepts

## Preamble

1. An *entity* is an object, projection, function, subscript, trait, type, namespace, or module.

2. A *name* denotes an entity. A name is composed of a stem identifier, and, optionally argument labels and/or an operator notation and/or an method implementation introducer. A name that is only composed of a stem identifier is called a *bare name*. A name that contains labels is called a *function name*. A name that contains an operator notation is called an *operator name*. An operator name may not have argument labels. A name, function name, or operator name that contains a method implementation introducer is called a *method name*. Every name is introduced by a __decl__.

3. (Example) `foo` and `+` are bare names. `foo(bar:ham:)` is a function name. `infix+` is an operator name. `foo.let` and `foo(bar:ham:).let`. are method names.

4. An __ident-expr__ is said to be a *use* of the name that it denotes. An expression is said to be a use of the all the uses of its sub-expressions.

5. A *binding* is a name that denotes an object or a projection. The value of a binding is the denoted object or the value of the denoted projection. The value of a binding may be mutable or immutable. A mutable binding can modify its value; an immutable binding cannot. A binding is *dead* at a given program point if it denotes an object that has been consumed. A mutable binding can be *resurrected* by consuming an object; an immutable binding cannot.

## Scopes and declaration spaces

1. A *lexical scope* is a region of the program represented by a syntactic element.

2. The lexical scope of a module or namespace declaration is called a *global scope*. The lexical scope of a type, extension, trait, or conformance declaration is called a *type scope*. Unless specified otherwise, the lexical scope of any other syntactic element is called a *local scope*.

3. A lexical scope `l1` *contains* a lexical scope `l2` if the region of the program text delimited by `l1` includes that delimited by `l2`. The innermost lexical scope that contains a lexical scope `l1` and that is not `l1` is called the *parent* of `l1`.

4. A lexical scope `l1` is a *sibling* of a lexical scope `l2` if `l1` is the lexical scope of a trait, nominal product type, or type alias declaration `d` and `l2` is the lexical scope of an extension or conformance declaration for the entity declared by `d`, or if `l2` is a sibling of `l1`.

5. The declaration space of a scope is the set of names introduced in that scope and its siblings. The declaration space of a declaration is the declaration space of its lexical scope.

6. (Example)

    ```val
    type A {
      fun foo() {}
    }
    extension A {
      fun bar() {}
    }
    ```

    The declarations space of the type declaration includes `foo` and `bar`.

7. A declaration may introduce one or more names in the declaration space of the innermost lexical scope that contains it. The same name may not be introduced more than once in a declaration space.

    ```val
    type A {
      fun foo() {}
    }
    extension A {
      fun foo() {} // error: invalid redeclaration of 'foo'
    }
    ```

## Name lookup

1. The procedure that identifies the entity denoted by a name is called *name lookup*. The rules of name lookup apply uniformly to all names.

2. Name lookup succeeds when it finds a single entity or a set of functions, which is called an *overload set*. Overload resolution takes place after name lookup and identifies a single entity from an overload set using the context in which the name appears.

3. Access rules are considered only once name lookup and overload resolution have succeeded.

### Unqualified name lookup

1. Unqualified name lookup `ulookup(n, s)` for a name `n` from a lexical scope `s` is described as follows:

    - If `qlookup(n, s) = E` and:

        - if `E` contains an entity `e` that is a local binding and `n` occurs after the declaration of `e` in the program text, `ulookup(n, s) = E`; or

        - if `E` is not empty, `ulookup(n, s) = E`;

    - otherwise:

        - if `ls` is not the lexical scope of a module declaration, `ulookup(n, s) = ulookup(n, s')` where `s'` is the innermost scope that contains `s`; or

        - if `s` is the lexical scope of a module declaration other than the Val standard library,  `ulookup(n, s) = ulookup(n, s')` where `s'` is the module declaration of the Val standard library; or

        - if `s` is the lexical scope of the Val standard library, `ulookup(n, s) = {}`.

### Qualified name lookup

1. *Qualified stem name lookup* `qslookup(n, s)` for a name `n` and a lexical scope `s` searches for the entity or entities denoted by a name `m` whose stem identifiers are equal to that of `n` in the context of the entity associated with `s`.

2. *Qualified name lookup* `qlookup(n, s)` for a name `n` and a lexical scope `s` searches for the entity or entities denoted by `n` in the context of the entity associated with `s`. The procedure is described as follows:

    - If `qslookup(n, s) = E` and:

      - if `n` is a bare name, `qlookup(n, s) = E`; or

      - if `n` is a method name, `qlookup(n, s) = E'` where `E'` contains the entities in `E` identified by a method name `m` whose method introducer is equal to that of `n`; or

      - `qlookup(n, s) = E'` where `E'` contains the entities in `E` identified by `n`;

    - otherwise, `qslookup(n, s) = {}`.

3. (Example)

    ```val
    type A {
      var a: Int
      fun foo(a: Int) { a.copy() }
      fun foo(b: Int) -> Int {
        let   { a + b }
        inout { b += a }
      }
    }
    ```

    Qualified name lookup for `foo(a:)` in the lexical scope `s` of the declaration of `A` starts with a qualified stem name lookup for `foo(a:)` in `s`. That search returns a set with three entities named `foo(a:)`, `foo(b:).let`, and `foo(b:).inout`. Since `foo(a:)` is a method name, the two last entities are discarded and the seach concludes with a singleton.

#### In a global or local scope

1. Qualified stem name lookup `qslookup(n, s)` for a name `n` in a local or global scope `s` is the set containing the entities such that there exists a name in the declaration space of `s` whose stem identifier is equal to the stem identifier of `n`.

#### In a type scope

1. The names introduced in the declaration space of the declaration of a trait, nominal product type, or type alias are members of the declared entity.

2. The members of a type are members of its aliases.

3. The members of a trait `T` are members of all the traits that `T` refines and all the types that conform to `T`.

4. The members of a generic type parameter are members of all the generic type parameters and associated types that are part of its equivalence class.

5. Qualified stem name lookup `qslookup(n, s)` for a name `n` in a type scope `s` is described as follows:

    - If `s` is the lexical scope of a nominal product type declaration, trait declaration, type alias declaration, generic type parameter declaration, or abstract type declaration, `qslookup(n, s)` is the set containing the entities such that there exists a member of the declared type such whose stem identifier is equal to the stem identifier of `n`.

    - If `s` is the lexical scope of an extension or conformance declaration, `qslookup(n, s) = qslookup(n, s')` where `s'` is the lexical scope of the declaration of the extended type.

## Program and linkage

1. A program consists of one or more module declarations linked together.

2. A name is said to have linkage when it might denote the same entity a name introduced by a declaration in another scope.

    1. When a name has *external linkage*, the entity it denotes can be referred to by names from any scope contained in other module declarations or from other scopes of the same module declaration.

    2. When a name has *module linkage*, the entity it denotes can be referred to by names from any scope in the same module declaration.

    3. When a name has *internal linkage*, the entity it denotes can be referred to by names from any scope contained in the scope that contains its declaration.

## Memory model

1. The fundamental storage unit in the Val memory model is the *byte*. A byte is a contiguous sequence of 8 bits. The memory available to a Val program consists of one or more sequences of contiguous bytes. Every byte has a unique address.

2. A *memory location* is a contiguous region of storage that has been allocated during a program's execution. A memory location has a size. A non-empty memory location has an address, determined as the address of the first byte in that location.

3. Two memory locations are *disjoint* if and only if they denote disjoint regions of storage. A memory location `l1` *contains* another memory location `l2` if and only if the region of storage denoted by `l1` contains the region denoted by `l2`. Two memory locations shall be disjoint or one shall contain the other.

4. A memory location may contain other memory locations, called sub-locations. A memory location that is not a sub-location of any other memory location is called a *root location*. A memory location `l1` must be disjoint from another memory location `l2` unless unless `l1`

5. A memory location may be *bound* to a type `A`. Such a memory location shall have the size and alignment suitable to store an object of type `A` and shall not contain any other memory location. Otherwise, the behavior is undefined. The memory locations contained in a memory location bound to `A` are bound according to the memory layout of `A`.

6. A memory location bound to `A` may be used as storage for an object of type `A`. A bound memory location shall not be rebound to another type or deallocated if it is used as storage for an object whose lifetime has not yet ended, or if it is contained in a bound memory location. Otherwise, the behavior is undefined.

### Memory location lifetime

1. The lifetime of a memory location starts when it is allocated and ends when it is deallocated. The lifetime of a memory location falls in one of three categories: *static*, *automatic*, or *dynamic*. The lifetime category is determined by the construct used to allocate the memory location.

    1. A memory location allocated for an object declared by a global binding declaration has *static lifetime* and is called a *static memory location*.

    2. A memory location allocated for an object declared by a local binding has *automatic lifetime* and is called an *automatic memory location*.

    3. A memory location allocated by a call to `Builtin.aligned_alloc(alignment:byte_count:)` has *dynamic lifetime* and is called a *dynamic memory location*.

2. A program shall terminate the lifetime of a dynamic memory location by calling `Builtin.dealloc`. Behavior is undefined if a program calls `Builtin.dealloc` to deallocate a static or automatic memory location.

3. A memory location shall not be occupied by an initializing, alive, or deinitializing object when it reaches the end of its lifetime.

4. When the end of the lifetime a memory location is reached, all names denoting objects stored within that location become invalid. Deallocating a memory location that has already reached the end of its lifetime has undefined behavior.

## Objects

1. The constructs in a Val program create, destroy, project, access, and modify *objects*. An object is the result of a scalar literal expression, aggregate literal expression, function call, call to a `sink` accessor, or it is the value of an escapable binding. [Note: A function is not an object but a lambda is.]

2. An object has a type determined at compile-time. That type might be polymorphic; in that case, the implementation generates information associated with the object that makes it possible to determine its concrete dynamic type at runtime.

3. An object occupies a memory location in its period of construction, throughout its lifetime, and in its period of destruction. That memory location must be bound to the type of the object.

4. An object may contain other objects, called sub-objects. A sub-object can be a member sub-object or a buffer element. An object that is not a sub-object of any other object is called a *root object*.

5. An object `o1` is nested within another object `o2` if and only if:

    1. `o1` is a sub-object of `o2`; or

    2. there exists an object `o3` such that `o1` is nested within `o3` and `o3` is nested within `o1`.

6. For every object `o`, there exists a single root object `o`, determined as follows:

    1. if `o` is a root object, then `o` is its own root;

    2. otherwise, the root object of `o` is the root object of the object that contains `o`.

7. An object has non-zero size if and only if its type has non-zero size. An object that has non-zero size occupies one or more bytes of contiguous storage, including every byte that is occupied in full or in part by any of its sub-objects.

8. Unless an object is a sub-object of zero size, the address of that object is the address of the memory location it occupies. Two objects with overlapping lifetimes may have the same address if one is nested within the other, or if at least one is a sub-object that has zero size; otherwise, they have distinct addresses and occupy disjoint memory locations.

### Object lifetime

1. The lifetime of an object of type `A` begins when:

    1. a memory location has been allocated and bound to `A`, and
    2. its initialization is complete.

2. The lifetime of an object of type `A ` ends when:

    1. its destructor has returned, and
    2. the memory location it occupies is rebound, resued for another object, or deallocated.

3. The properties ascribed to objects throughout this document apply for a given object only during its lifetime. [Note: The behavior of an object under construction and destruction might not be the same as the behavior of an object whose lifetime has started and not ended.]

4. An object is said to be *initializing* in the period after its memory location has been bound abd before its lifetime has started, *alive* throughout its lifetime, *deinitializing* during a bound call of a consuming method of that object, and *dead* in the period after its lifetime has ended and before its storage has been rebound, reused, or deallocated.

5. Unless specified otherwise, a program shall not manipulate an object outside of its lifetime. Behavior is undefined if the program:

    1. accesses the object or its sub-objects; or

    2. projects the object or its sub-objects; or

    3. performs a bound call of a method of the object.

6. If, after the lifetime of an object has ended and before the memory location which the object occupied is reused or deallocated, a new object is stored at the memory location which the original object occupied, the name of the original object automatically denotes the new object and, once the lifetime of the new object has started, can be used to manipulate the new object.

### Object alignment

1. Object types have alignment requirements which place restrictions on the addresses of the memory locations which an object of that type may occupy. An alignment is an platform-specific integer value representing the number of bytes between successive addresses at which a given object can be allocated. An object type imposes an alignment requirement on every object of that type.

2. Alignments are represented as values of the type `Int`. Valid alignments include only those values of `MemoryLayout<A>.alignnment` for any type `A` plus the value of `Pointer.universal_alignment`. Every alignment value shall be a non-negative integral power of two.

3. Alignments have an order from *weaker* to *stricter* alignments. Stricter alignments have larger alignment values. An address that satisfies an alignment requirement also satisfies any weaker valid alignment requirement.

4. Comparing alignments is meaningful and provides the obvious results:

    1. Two alignments are equal when their numeric values are equal.

    2. Two alignments are different when their numeric values are not equal.

    3. When an alignment is larger than another it represents a stricter alignment.

### Sinkability

1. An object is said to be *consumed* if it is passed as argument to a `sink` parameter, returned from a function, or assigned to a mutable, escapable, or member binding.

2. An *unsinkable* may not be consumed by any operation except its destructor. An object of type `[let A]` or `[inout A]` is unsinkable. A closure that captures projections is unsinkable. An object initialized with an unsinkable object is unsinkable.

3. (Example)

    ```val
    type Wrapper<A> { var wrapped: A }

    fun main() {
      let w = 2
      let x = Wrapper(wrapped: w.copy()) // the value of 'x' is sinkable
      let y = Wrapper(wrapped: [let w])  // the value of 'y' is unsinkable
      let z = Wrapper(wrapped: y)        // the value of 'z' is unsinkable
    }
    ```

### Escapability

1. A sinkable object evaluated by a non-consuming expression is escapable at a given program point if it is not projected by any other object at that program point.

2. A sinkable object bound to an escapable binding is escapable at a given program point if it is not projected by any other object at that program point.

3. (Example)

    ```val
    fun print_and_return_forty_two() -> Int {
      sink let x = 42 // the value of 'x' is escapable here
      let y = x       // the value of 'x' is not escapable here
      print(y)        // the value of 'x' is escapable afterward
      return x        // the value of 'x' escapes here
    }
    ```

    The value of `y` is never escapable because `y` is not an escapable binding.

4. An escapable object may be consumed.

## Projections

1. A projection exposes an object.

2. An object `o1` projects an object `o2` if and only if:

    1. `o1` is a projection of `o2` or one of `o2`'s sub-objects; or

    2. a sub-object of `o1` projects `o2`.

3. When an object `o1` projects an object `o2`, `o2` is said to be projected by o1`. [Note: Copying creates a new object that is not a projection.]

4. (Example)

    ```val
    type A { fun zero: Int { 0 } }

    fun main() {
      let x = A()
      let y = x.zero
        // the value of 'y' projects that of 'x'
        // corollary: the value of 'x' is projected by that of 'y'
    }
    ```

5. The object yielded by a subscript projects the arguments bound to the subscript's out parameters.

6. (Example)

    ```val
    subscript min<T, E>(
      _ a: out T, _ b: out T, by less_than: [E] (T, T) -> Bool
    ): T {
      let { if less_than(a, b) { yield a } else { yield b } }
    }

    fun main() {
      let comparator = Int.<
      let (x, y) = (2, 3)
      let z = min[x, y, by: comparator]
        // the value of 'z' projects both the values of 'x' and 'y',
        // but not that of 'comparator'
      print(z)
    }
    ```

    The expression `min[x, y, by: comparator]` is a call to the `let` implementation of `min`, which projects the values of its first and second arguments, but not that of its third argument.

7. A closure projects the objects it captures by projection.

8. An object projects the objects captured by the stored projections with which it has been initialized.

9.  If an object `o1` projects an object `o2` immutably, `o2` is immutable for the duration of `o1`'s lifetime. If an object `o1` projects an object `o2` mutably, `o2` is inaccessible for the duration of `o1`'s lifetime.

10. (Example) Mutable projection:

    ```val
    fun main() {
      var x = 42
      var y = x // mutable projection begins here
      print(x)  // error: the value of 'x' is projected mutably
      x += 1    // error: the value of 'x' is projected mutably
      print(y)  // mutable projection ends afterward
    }
    ```

11. (Example) Immutable projection:

    ```val
    fun main() {
      var x = 42
      let y = y // immutable projection begins here
      print(x)  // OK
      x += 1    // error: the value of 'x' is projected immutably
      print(y)  // immutable projection ends afterward
    }
    ```

# Types

## General

1. The Val language is statically typed. Every entity has a type that is known at compile time. The type of an entity is immutable.

2. A type has a canonical form. Two types are equivalent if and only if they have the same canonical form. A type is either *structural* or *nominal*. A nominal type has a name and is defined by a type declaration, unless it is a built-in type. [Note: The types `Any` and `Never` are not considered nominal types.]

3. A type is *instantiable* if it can be the type of an object. An instantiable type `A` has a memory layout that defines the *size* and *alignment* requirements to bind a memory location to `A`.

4. A type has a *type representation* if it can be represented at compile-time. [Note: type representation can be understood as the object representation of a type expression in the compile-time interpreter.]

5. The *object representation* of an object is the set of bytes that it occupies in storage. The *value representation* of an object of type `A` is the set of bits that participate in representing a value of type `A`. Bits in the object representation that are not part of the value representation are called *padding bits*.

## Built-in types

1. A built-in type is an instantiable nominal type representing a value on the execution machine. A built-in type may be referred to only in Val standard library and shall not conform to any trait.

2. A built-in type has non-zero size. The value representation of a built-in type determines its *value*.

3. A built-in integer types denotes a bit pattern and does not specify signedness. There are six built-in integer types. Five are named `Builtin.I{n}` where `n` is `1`, `8`, `16`, `32`, or `64` and denotes the number of bits in the value representation of the type; the sixth is named `Builtin.Word` and has the same value representation as either `Builtin.I32` or `Builtin.I64` depending on the execution machine. The object representation of a built-in integer type is the minimum number of bytes necessary to store its value representation.

4. There are four built-in floating-point types named `Builtin.F{n}` where `n` is `16`, `32`, `64`, or `80` and denotes the number of bits in the value representation of the type. The object representation of a built-in integer type is the minimum number of bytes necessary to store its value representation.

5. There is a unique built-in pointer type `Builtin.Pointer` that denotes an untyped pointer to a memory location. The object and value representation of `Builtin.Pointer` are the same as that of `Builtin.Word`.

## Subtyping relation

1. The types form a lattice, partially ordered by a subtyping relation `<:`, for which the least (and only) upper bound is `Any` and the greatest (and only) lower bound is `Never`.

2. If a type `A1` is equivalent to a type `A2`, then `A1 <: A2`.

3. If the canonical form of a type `A1` is subtype of the canonical form of a type `A2`, then `A1 <: A2`.

4. If `A1` and `A2` are union types, `A1 <: A2` if all operands of `A1` are subtype of `A2`. If `A2` is a union type and `A1` is not, `A1 <: A2` if `A1` is subtype of at least one operand of `A2`.

5. If `A1` and `A2` are function types, `A1 <: A2` if:

    1. `A1` and `A2` have the same number of parameters and labels; and

    2. the type of each parameter of `A2` is subtype of the type of the corresponding parameter of `A1`; and

    3. the return type of `A1` is subtype of the return type of `A2`.

6. If `A1` and `A2` are lambda types, `A1 <: A2` if:

    1. the function type of `A1` is subtype of the function type of `A2`; and

    2. the environment of `A1` and `A2` are both equivalent or the environnment of `A2` is erased.

7. If `A1`and `A2` are existential types, `A1 <: A2` if the traits of `A2` are coarser than the traits of `A1` and the associated types of `A1` are equivalent to the associated types of `A2`.

8. If `A1` and `A2` are bound generic types, `A1 <: A2` if each type arguments of `A1` is equivalent to corresponding type argument of `A2`.

9.  (Example)

    The subtyping relation holds for the following pairs of types:

      - `(A | B) <: (A | B | C)`

      - `[E] ((A2) -> B1) <: ((A1) -> B2)` (where `A1 <: A2` and `B1 <: B2`)

      - `(T & U & V) <: (T & U)`

      - `(any T & U where ::T.Element == Int) <: (any T where ::T.Element == Int)`

## Generic types

# Declarations

## General

1. A declaration may introduce one or more entities.

2. A declaration may be composed other declarations, called sub-declarations. The entities of introduced by the sub-declaration of a declaration `d` are also said to be introduced by `d`. [Note: a sub-declaration is part of a declaration itself, unlike a member declaration, which is a separate construct contained in the lexical scope of a declaration.]

## Modifiers

### Access modifiers

1. A name is *exposed* to a lexical scope if it can be referred to from that scope. When a name is exposed to a lexical scope, it is exposed to all scopes contained in that scope. An access modifier specifies how a declaration exposes the names it introduces. Access modifiers have the form:

    ```ebnf
    access-modifier ::=
      'public'
    ```

2. A declaration is *private* if it does not have any access modifier. A private declaration *exposes* the names that it introduces to the innermost scope that contains it. The static and non-static members introduced in the lexical scope of a type declaration are additionally exposed to the lexical scopes of the conformance declarations of the declared type.

3. A declaration is *public* if it has a `public` access modifier. A public declaration exposes the names it introduces the parent scope of the innermost scope that contains it. A public declaration that appears in the lexical scope of a module is *external*.

4. (Example)

    ```val
    type A {
      var x: Int
      public fun foo() -> Int { x.copy() }
    }

    conformance A: Copyable {
      public fun copy() -> Self {
        // 'x' is exposed to the conformance declarations of 'A'
        A(x: x.copy())
      }
    }

    fun main() {
      let A(x: 1)
      print(a.foo()) // OK
      print(a.x)     // error: 'a.x' is not exposed here
    }
    ```

### Member modifiers

1. Member modifiers have the form:

    ```ebnf
    member-modifier ::=
      receiver-modifier
      static-modifier

    receiver-modifier ::= (one of)
      sink inout out

    static-modifier ::=
      'static'
    ```

2. A member modifier may appear at most once in a declaration.

3. The `static` may only apply to a binding declaration, a function declaration, or a subscript declaration at type scope.

4. A receiver modifier may only appear in a function declaration or a subscript declaration at type scope.

5. The `out` modifier may only appear in a subscript declaration.

## Conformance lists

1. Conformance lists have the form:

    ```ebnf
    conformance-list ::=
      ':' type-identifier (',' type-identifier)*
    ```

## Generic clauses

1. Generic clauses have the form:

    ```ebnf
    generic-clause ::=
      '<' generic-param (',' generic-param)* where-clause? '>'

    generic-param ::=
      generic-type-param
      generic-size-param

    generic-type-param ::=
      identifier '...'? trait-annotation?

    trait-annotation ::=
      ':' trait-composition

    generic-size-param ::=
      identifier ':' 'size'
    ```

2. When a generic type parameter is followed by a trait annotation, that annotation is interpreted as a conformance constraint as though it as been written in the where clause.

3. (Example)

    The generic clause `<X, Y: Copyable & Equatable>` is sugar for `<X, Y where Y: Copyable & Equatable>`.

4. A generic type parameter whose identifier is directly suffixed by an ellipsis is said to be *variadic*. Within the generic environment in which it is introduced, a variadic type parameter denotes a list of skolems whose length has been existentially quantified. A generic clause can define at most one variadic type parameter, which must appear last.

## Where clauses

1. Where clauses have the form:

    ```ebnf
    where-clause ::=
      'where' where-clause-constraint

    where-clause-constraint ::=
      equality-constraint
      conformance-constraint
      size-constraint-expr

    equality-constraint ::=
      type-name '==' type-expr

    conformance-constraint ::=
      type-name ':' trait-composition
    ```

2. A where clause specifies constraints on the generic parameters introduced by a generic signature, or the associated type requirements introduced by a trait declaration.

    1. An equality constraint specifies that the types denoted by either side of `==` must be equivalent.

    2. A conformance constraint specifies that the type denoted by the left hand side of `:` be conforming to the traits specified in __trait-composition__.

    3. A size constraint is an expression denoting a predicate over one or more size parameters. It must be an expression of type `Bool` and shall only refer to size parameters or names introduced in global scopes.

## Trait declarations

### General

1. A trait is a collection of requirements on a type. [Note: A trait is not a type but it may form a type if it is part of an existential type.]

2. Trait declarations have the form:

    ```ebnf
    trait-decl ::=
      trait-head trait-body

    trait-head ::=
      access-modifier? 'trait' identifier trait-refinement-list

    trait-refinement-list ::=
      ':' type-name (',' type-name)*

    trait-body ::=
      '{' (trait-requirement-decl | ';')* '}'

    trait-requirement-decl ::=
      associated-type-decl
      function-decl
      subscript-decl
    ```

3. (Example)

    ```val
    trait Shape {
      static fun name() -> String
      fun draw(to: inout Canvas)
    }
    ```

4. A trait declaration may only appear at global scope. It introduces __identifier__ as a name denoting the declared trait.

5. The associated type, function, and subscript declarations that appear in the body of a trait specify the *requirements* of that trait. Such declarations may not have access levels. [Note: The access level of a requirement implementation depends on the visibility of the conformances requiring that implementation.]

### Associated type requirements

1. An associated type declaration defines an *associated type requirement*. As associated type is a placeholder for a type that relates to the trait and must be specified by conforming types. Associated type declarations have the form:

    ```ebnf
    associated-type-decl ::=
      associated-type-head associated-type-constraints? associated-type-default?

    associated-type-head ::=
      static-modifier? identifier

    associated-type-constraints ::=
      conformance-list
      conformance-list? where-clause

    associated-type-default ::=
      '=' type-expr
    ```

2. The type constraints of an associated type declaration impose constraints on the types that may satisfy an associated type requirement.

3. (Example)

    ```val
    trait Generator {
      type Element: Copyable
      inout fun next() -> (element: Element, done: Bool)
    }
    ```

    The trait `Generator` declares an associated type requirement `Element`, constrained to types that conform to another trait `Copyable`. `Element` appears as in the return type of the method requirement `next`.

### Method requirements

1. A function declaration that appears in the body of a trait declaration is a *method requirement declaration* that defines a one or more *method requirements*. A method requirement is the specification of a method that must be implemented in conforming types.

2. When a method requirement declaration has a __function-bundle-body__, each method implementation in that body denotes a method requirement. When a method requirement declaration is bodiless, it defines denotes a single method requirement whose kind depends on the receiver modifier of the method requirement declaration.

3. A method requirement may have one or several default implementations. A default implementation may be defined as the body of a function declaration requirement, as the body of a method implementation in the __function-bundle-body__ of a function requirement declaration, or via a default requirement implementation declared in a trait extension.

4. (Example)

    ```val
    trait State {
      subscript x: Int { let inout }
    }

    trait Counter {
      fun current() -> Int { 0 }
      fun offset(by: Int) -> Int { let inout }
    }

    extension Counter {
      fun offset(by value: Int) -> Int {
        let { current() + value }
      }
    }

    extension Counter where Self: State {
      fun current() -> Int { x.copy() }
      fun offset(by value: Int) -> Int {
        inout { x += value }
      }
    }
    ```

    The method requirement `Counter.current` has two default implementations: one is defined in the declaration of `Counter` and the other in the conditional trait extension. The method requirement `Counter.foo.let` has one default implementation, defined in the unconditional trait extension. The method requirement `Counter.foo.inout` has one default implementation, defined in the conditional trait extension.

### Subscript requirements

1. A subscript declaration that appears in the body of a trait declaration is a *subscript requirement declaration* that defines a one or more *subscript requirements*. A subscript requirement is the specification of a subscript that must be implemented in conforming types.

2. A subscript requirement declaration must have a __subscript-bundle-body__. Each subscript implementation in that body denotes a subscript requirement.

3. A subscript requirement may have one or several default implementations. A default implementation may be defined as the body of a subscript implementation in the __subscript-bundle-body__ of a subscript requirement requirement, or via a default requirement implementation declared in a trait extension.

### Trait refinement

1. A trait `T1` is said to *refine* another trait `T2` if it declares conformance to `T2` and its set of requirements includes all requirements of `T2`. Conformance of the trait `T1` to `T2` shall be declared in the conformance list of the declaration of `T1`.

2. Refinement introduces an ordering between the two traits. A trait `T1` is *finer* than a trait `T2` if `T1` refines `T2` or if there exists a trait `T3` such that `T1` refines `T3` and `T3` is finer than `T2`. Trait refinement shall not introduce cycles. If `T1` is finer than `T2`, `T2` is said to be *coarser* than `T1`.

3. (Example)

    ```val
    trait A: C {} // error: refinement introduces a cycle
    trait B: A {}
    trait C: B {}
    ```

### Trait conformance

1. A type `A` *conforms* a trait `T1` in a lexical scope if a conformance of `A` to `T1` is exposed to that scope and if `T1` and satisfies all the requirements of `T1`, or if `A` conforms to a trait `T2` such that `T2` refines `T1`.

2. (Example)

    ```val
    trait T { fun foo() }
    trait U { fun bar() }

    // declares conformance to 'T'
    type A: T {}

    // satisfies conformance to 'T'
    extension A {
      fun foo() {}
    }

    // declares and satisfies conformance to 'U'
    conformance A: U {
      publc fun bar() {}
    }
    ```

3. A *source* of conformance denotes a declaration defining the conformance. A type or conformance declaration is a source of conformance for all the traits that appear in its inheritance list. A source of conformance is conditional if it is a conformance declaration with a where clause. A type may have at most one source of conformance to a specific trait. A type that conforms to a trait `T1` shall not have a source of conformance to a trait `T2` if `T2` refines `T1` and the source of conformance to `T1` is conditional.

4. The conformance of a type `A` to `T` is *exposed* to a lexical scope `l` if and only if `A` is exposed to `l` and the source of the conformance is:

    1. the declaration of `A`; or

    2. a conformance declaration declared in `l` or a lexical scope that contains `l`; or

    3. an external conformance declaration imported from another module.

5. The conformance of a type `A` to a trait `T` shall not be exposed outside of the lexical scope of a module `m` unless at least `A` or `T` is declared in `m`.

6. (Example)

    ```val
    import M

    public type A {}
    conformance A: M.T    // OK: conformance is private

    type B: M.T {}        // OK: 'B' is not exposed outside of the module

    public type C: M.T {} // error: cannot expose conformance to imported trait 'M.T'
    ```

7. A method requirement `r` is satisfied if by a type `A` if `A` has a single method `m` with the same name, type, and kind. `m` may be defined in the type declaration, extension declaration, or a conformance declaration of `A`. If `r` has default implementations, it may be satisfied by `A` if there exists a unique default implementation whose conditions are satisfied by `A`.

## Product type declarations

### General

1. Nominal product type declarations have the form:

    ```ebnf
    product-type-decl ::=
      product-type-head product-type-body

    product-type-head ::=
      access-modifier? identifier generic-clause? conformance-list

    product-type-body ::=
      '{' product-type-member-decl-list '}'

    product-type-member-decl-list ::=
      (product-type-member-decl | ';')*

    product-type-member-decl ::=
      function-decl
      subscript-decl
      product-type-decl
      type-alias-decl
    ```

## Type alias declarations

### General

1. Type alias declarations have the form:

    ```ebnf
    type-alias-decl ::=
      type-alias-head type-alias-body

    type-alias-head ::=
      access-modifier? 'typealias' identifier generic-clause?

    type-alias-body ::=
      '=' type-expr
      '=' union-decl

    union-decl ::=
      product-type-decl ('|' product-type-decl)*
    ```

## Extension declarations

### General

1. Extension declarations have the form:

    ```ebnf
    extension-decl ::=
      extension-head extension-body

    extension-head ::=
      access-modifier? 'extension' type-expr where-clause?

    extension-body ::=
      '{' (extension-member-decl | ';')* '}'

    extension-member-decl ::=
      function-decl
      subscript-decl
      product-type-decl
      type-alias-decl
    ```

2. An extension declaration may not appear in the lexical scope of a conformance or extension declaration.

3. A `public` access modifier may only appear in extension declarations defined in the lexical scope of a module. An public extension declaration exposes its public members outside of the module in which it is declared.

4. When a subscript or method `e` is defined in an extension declaration, the constraints of the __where-clause__ of that declarations are called the conditions of `e`.

## Conformance declarations

### General

1. Conformance declarations have the form:

    ```ebnf
    conformance-decl ::=
      conformance-head conformance-body

    conformance-head ::=
      access-modifier? 'conformance' type-expr ':' conformance-list where-clause?

    conformance-body ::=
      '{' (conformance-member-decl | ';')* '}'

    conformance-member-decl ::=
      function-decl
      subscript-decl
      product-type-decl
      type-alias-decl
    ```

2. A conformance declaration may not appear in the lexical scope of a conformance or extension declaration.

3. A `public` access modifier may only appear in conformance declarations defined in the lexical scope of a module. An public conformance declaration exposes new conformances outside of the module in which it is declared.

4. (Example)

    ```val
    // In 'MyModule.module.val'
    public namespace Foo {
      public trait T {}
      public conformance Int: T {} // error
    }
    public conformance String: T {} // OK: `String` conforms to `T` in importing modules.
    ```

## Binding declarations

### General

1. Binding declarations have the form:

    ```ebnf
    binding-decl ::=
      binding-head binding-type-annotation? binding-initializer?

    binding-head ::=
      access-modifier? member-modifier* binding-introducer pattern

    binding-introducer ::= 
      'let'
      'var'
      'sink' 'let'
      'sink' 'var'
      'inout'

    binding-type-annotation ::=
      ':' type-expr

    binding-initializer ::=
      '=' expr
    ```

2. (Example)

    ```val
    let (name, age): (String, Int) = ("Thomas", 3)
    ```

    This binding declaration defines two new immutable bindings: `name` and `age`.

3. A binding declaration defines a new binding for each name pattern in pattern. All new bindings are defined with the same capabilities.

    1. A binding declaration introduced with `let` defines an immutable binding. The value of a live immutable binding may be projected immutably. The value of an immutable binding may not be projected mutably for the duration of that binding's lifetime.

    2. A binding declaration introduced with `var` or `inout` defines a mutable binding. The value of a live mutable binding may be projected mutably or immutably.

    3. A binding declaration introduced with `sink` defines an escapable binding. An escapable binding may only be assigned to sinkable objects. The value of an escapable binding may be consumed at a given program point if is escapable at that program point.

  A mutable binding can appear on the left side of an assignment, on the right side of an assignment to a non-escapable mutable binding, as the initializer of non-escapable mutable binding, as argument to an `inout` parameter, or as argument to an `assign` parameter. An escapable binding can appear as the

4. The pattern of a binding declaration may not contain any binding patterns and, unless it appears in the condition of a loop statement or in the condition of a selection expression, it may not contain any expression patterns.

5. A binding declaration may be defined at module scope, namespace scope, type scope, or function scope.

    1. A binding declaration at module scope or namespace scope is called a global binding declaration. It introduces one or more global bindings. A global binding declaration must be introduced with `let`.

    2. A binding declaration at type scope is called a static member binding declaration if it contains a `static` modifier. Otherwise, it is called a member binding declaration. A static member binding declaration introduces one or more global bindings. A member binding declaration introduces one or more member bindings.

    3. A binding declaration at function scope is called a local binding declaration. It introduces one or more local bindings.

6. The `sink` capability may only appear in a local binding declaration introduced with `let` or `var`.

### Initialization

1. A static member binding declaration, or global binding declaration must contain an initializer. Initialization occurs at the first dynamic use of one of the declared bindings.

2. A local binding declaration must contain an initializer unless it appears in the condition of a match expression. If the binding declaration is a sub-statement in a brace statement, initialization occurs immediately after declaration. If the binding declaration is a condition in a loop statement or a condition in a selection expression, initialization occurs immediately after the a successful pattern matching.

3. A member binding declaration may not have an initializer. Initialization occurs in the constructor of the object of which the introduced bindings are members (see Type initialization).

4. The initialization of an escapable binding or a binding whose declaration is introduced with `var` consumes the value of its initializer.

5. The initialization of a non-escapable binding whose declaration is introduced with `let` or `inout` projects the object to which its initializer evaluates. The projection is immutable if the binding is immutable. Otherwise, it is mutable. The projection is for the duration of the binding's lifetime.

6. (Example)

    ```val
    fun main() {
      var fruits = ["apple", "mango", "orange"]
      inout first = fruits[0] // mutable projection of 'fruits[0]'
      first = "strawberry"
      print(first)            // projection ends afterward
    }
    ```

### Lifetime

1. Binding lifetimes are not bound to lexical scopes.

2. A binding is said to be alive at a given program point if that program point falls within one of its lifetimes. Otherwise, it is said to be dead.

3. The lifetime of a global binding begins with its initialization is complete and ends when the program terminates.

4. The lifetime of a member binging `b` begins when the initialization of the object `o` of which it is a sub-object is complete and ends when `o` is consumed, or when `b` is consumed in a `sink` method of `o`.

5. The lifetime of a local binding begins when its initialization is complete and ends after its last use in an operation, or after any consuming operation.

6. (Example) Lifetime ending at last use.

    ```val
    fun main() {
      let count = 1 // lifetime of 'count' begins here
      count += 1    // a use of 'count'
      print(count)  // last use of 'count', lifetime ends afterward
      print("done")
    }
    ```

7. (Example) Lifetime ending because of a consuming operation.

    ```val
    fun main() {
      sink let count = 1 // lifetime of 'count' begins here
      sink _ = count     // lifetime of 'count' ends here
      print(count)       // error: 'count' has been consumed
    }
    ```

8. A dead immediate local mutable binding can be re-initialized, starting a new lifetime.

9. (Example) A binding with two lifetimes.

    ```val
    fun borrow<A>(_ thing: inout T) {
      let a = [thing]         // lifetime of 'thing' ends here
      print(a)
      thing = a.remove_last() // new lifetime of 'thing' starts here.
    }
    ```

    The lifetime of `thing` ends when `a` is initialized because construction an array literal consumes the literal's elements. A new lifetime starts before `borrow` returns as the call to `Array.remove_last` produces an independent value.

## Function declarations

### General

1. Function declarations have the form:

    ```ebnf
    function-decl ::=
      'default' 'init'
      function-head function-signature function-body?

    function-head ::=
      access-modifier? member-modifier* function-ident generic-clause? capture-list?

    function-ident ::=
      'init'
      'deinit'
      'fun' identifier
      oper-notation 'fun' operator

    function-body ::=
      function-bundle-body
      brace-stmt

    function-bundle-body ::=
      '{' method-impl+ '}'

    operator ::= (token)
      raw-operator+
    ```

2. (Example)

    ```val
    fun gcd(_ a: Int, _ b: Int) -> Int {
      if b == 0 { a } else { gcd(b, a % b) }
    }
    ```

3. A function declaration introduces one or more function objects.

4. A function declaration may be defined at module scope, namespace scope, type scope, or function scope.

    1. A function declaration at module scope or namespace scope is called a global function declaration. It introduces one global function.

    2. A function declaration at type scope that contains a `static` modifier is called a static method declaration; it is also a global function declaration. A static method declaration introduces one global function.

    3. A function declaration at type scope that does not contain a `static` modifier is called a method declaration. It introduces one or more methods.

    4. A function declaration at type scope declared with `init` is called a constructor declaration. It introduces one global function.

    5. A function declaration at type scope declared with `deinit` is called a destructor declaration. It introduces one sink method.

    6. A function declaration at function scope is called a local function declaration. It introduces a local function.

5. The `init` introducer and the `deinit` introducer may only appear in a function declaration at type scope.

6. A method implementation may only appear in a method declaration.

7. A function declaration of the form 'default init' is an *explicit default constructor* and may only appear in a product type declaration.

8.  An operator notation specifier defines an operator member function; it may only appear in a function declaration at type scope.

9.  A capture list may only appear in a function declaration at local scope.

### Function signatures

1. Function signatures have the form:

    ```ebnf
    function-signature ::=
      '(' param-list? ')' ('->' type-expr)?
    ```

2. The default value of a parameter declaration may not refer to another parameter in a function signature.

3. The output type of a function signture defines the output type of the containing declaration. If that type is omitted, the output type of the declaration is interpreted as `()`.

### Function implementations

1. The brace statement in the body of a global or local function declaration defines its implementation. A global or local  function declaration must have a function implementation, unless it is static method declaration.

2. The parameters declared in the signature of the function declaration containing a function implementation define the parameters of that implementation.

3. All parameters of a function implementation are treated as immediate local bindings in that implementation, defined before its first statement. All parameters except `assign` parameters are alive before the first statement. Further:

    1. `sink` parameters are immutable and escapable; and

    2. `inout` parameters are mutable and escapable, and they must be alive at the end of every terminating execution path; and

    3. `assign` parameters are dead and mutable, and they must be alive at the end of every terminating execution path.

4. The output type of a function implementation is the output type of the containing function declaration, unless it is an explicit `inout` method implementation (see Method implementations). A function implementation must have a return statement on every terminating execution path, unless the output type of the containing declaration is `()`. In that case, explicit return statements may be omitted. A function implementation must return an escapable object whose type is a subtype of the output type of the containing method declaration.

5. The `return` keyword may be omitted if the body of the function implementation consists of a single expression.

6. (Example) Expression-bodied function

    ```val
    fun factorial(_ n: Int) -> Int {
      if n > 0 { n * factorial(n - 1) else { 1 } }
    }
    ```

### Method declarations

1. A bodiless method declaration or a method declaration that contains a bodiless method implementation defines a method requirement and may only appear in a trait declaration. A bodiless static method declaration defines a static method requirement and may only appear in a trait declaration.

2. A method declaration may contain at most one method implementation of each kind. A method declaration that contains one or more explicit method implementations may not have a receiver modifier.

3. A non-bodiless method declaration must contain an explicit `let` method implementation, or its body must be a brace statement. In that case, that brace statement is interpreted as an implicit method implementation whose kind depends on the receiver modifier of the method declaration.

4. The declaration of a method `m` that contains an explicit `inout` method implementation is automatically provided with a synthesized `sink` method implementation, defined as follows:

    ```val
    sink {
      sink var this = self
      this.=m(arg1, ..., argn)
      return this
    }
    ```

5. The declaration of a method `m` that contains an explicit `sink` method implementation is automatically provided with a synthesized `inout` method implementation, defined as follows:

    ```val
    inout {
      self = m(arg1, ..., argn)
    }
    ```

6. (Example)

    ```val
    type Vector2 {

      var x: Double
      var y: Double

      fun scaled(by factor: Double) -> Self {
        let   { Vector2(x: x * factor, y: y * factor) }
        inout { x *= factor; y *= factor }
      }

      fun dot(_ other: Vector2) -> Double {
        x * other.x + y * other.y
      }

      inout fun transpose() {
        swap(&x, &y)
      }

    }
    ```

    The method `Vector2.scaled(by:)` has an explicit `let` implementation, an explicit `inout` implementation and a synthesized `sink` implementation. The method `Vector2.dot(_:)` has an implicit `let` implementation. The method `Vector2.transpose` has an implicit `inout` implementation.

### Method implementations

1. A method implementation is a function implementation defined in a method. It may be defined implicitly or explicitly (see Method declarations). Explicit method implementations have the form:

    ```ebnf
    method-impl ::=
      method-introducer brace-stmt?

    method-introducer ::= (one of)
      let sink inout
    ```

2. An explicit method implementation introduced with `let` is called a `let` method implementation; one introduced with `sink` is called a `sink` method implementation; one introduced with `inout` is called an `inout` method implementation.

3. The parameters declared in the signature of the method declaration containing a method implementation define the parameters of that implementation. An additional implicit parameter, called the receiver parameter, and named `self`, represents the method receiver.

4. In an explicit method implementation, the passing convention of the receiver parameter corresponds to the kind of the method implementation: it is `let` in a `let` implementation; it is `sink` in a `sink` implementation; it is `inout` in a `inout` implementation. In an implicit method implementation, the passing convention of the receiver parameter is defined by the receiver modifier of the containing method declaration.

5. The output type of an explicit `inout` method implementation is `()`.

## Subscript declarations

### General

1. Subscript declarations have the form:

    ```ebnf
    subscript-decl ::=
      subscript-head subscript-signature subscript-body

    subscript-head ::=
      member-modifier* subscript-ident? generic-clause? capture-list?

    subscript-ident ::=
      'subscript' identifier
      oper-notation 'subscript' operator

    subscript-body ::=
      brace-stmt
      '{' subscript-impl+ '}'
    ```

2. (Example)

    ```val
    subscript min<T, E>(_ a: T, _ b: T, by comparator: [E](T, T) -> Bool): Int {
      let    { if comparator(a, b) { yield a } else { yield b } }
      inout  { if comparator(a, b) { yield &a } else { yield &b } }
      assign { if comparator(a, b) { yield &a } else { yield &b } }
      sink   { if comparator(a, b) { return a } else { return b } }
    }
    ```

3. A subscript declaration may be defined at module scope, namespace scope, type scope, or function scope.

    1. A subscript declaration at module scope or namespace scope introduces a global subscript declaration. It introduces a global subscript.

    2. A subscript declaration at type scope that contains a `static` modifier is called a static subscript declaration; it is also a global subscript declaration. A static member subscript declaration introduces a global susbcript.

    3. A subscript declaration at type scope that does not contain a `static` modifier is called a member subscript declaration. It introduces a member subscript. A member subscript declaration may not have a receiver modifier.

    4. A subscript declaration at function scope is a local subscript declaration. It introduces a local subscript.

4. A member subscript declaration or a static member subscript declaration without an identifier is called a nameless subscript declaration. It introduces a nameless subscript. A nameless subscript declaration must have an explicit parameter list.

5. A member subscript declaration or a static member subscript declaration may be defined without an explicit parameter list. Such a declaration is called a property subscript declaration. It introduces a property subscript.

6. A bodiless subscript declaration or a subscript declaration that contains a bodiless subscript implementation defines a subscript requirement and may only appear in a trait declaration. A bodiless static subscript declaration defines a static subscript requirement and may only appear in a trait declaration.

7. A subscript declaration may contain at most one subscript implementation of each kind.

8. If the body of a subscript declaration is a brace statement, it is interpreted as the body of a `let` subscript implementation.

9. An operator notation specifier defines an operator member subscript; it may only appear in a subscript declaration at type scope.

10.  A capture list may only appear in a subscript declaration at local scope.

### Subscript signatures

1. Subscript signatures have the form:

    ```ebnf
    subscript-signature ::=
      explicit-subscript-param-list? ':' 'var'? type-expr

    explicit-subscript-param-list ::=
      '(' param-list? ')'
    ```

2. The default value of a parameter declaration may not refer to another parameter in a subscript signature.

3. The output type of a subscript signture defines the output type of the containing declaration. If that type is prefixed by `var`, all projections produced by the subscript are mutable. Otherwise, only the projections produced by the `inout` implementation of the subscript are mutable.

### Subscript implementations

1. A subscript implementation may be defined ikmplicitly or explicitly. Explicit subscript implementations have the form:

    ```ebnf
    subscript-impl ::=
      subscript-introducer brace-stmt?

    subscript-introducer ::= (one of)
      let sink inout assign
    ```

2. Am explicit subscript implementation introduced with `let` is called a `let` subscript implementation; one introduced with `sink` is called a `sink` subscript implementation; one introduced with `inout` is called an `inout` subscript implementation; one introduced with `assign` is called an `assign` subscript implementation.

3. The parameters declared in the signature of the subscript declaration containing a subscript implementation define the parameters of that implementation. In a member subscript declaration, an additional implicit `out` parameter, called the receiver parameter, named `self`, and representing the subscript receiver.

4. The passing convention of an `out` parameter dependends on the kind of the sibscript implementation: it is a `let` parameter in a `let` subscript implementation; or it is a `sink` parameter in an `sink` subscript implementation; or it is an `inout` parameter in an `inout` subscript implementation; or it is an `assign` parameter in an `assign` subscript implementation.

5. All parameters of a subscript implementation are treated as immediate local bindings in that implementation, defined before its first statement. All parameters except `assign` parameters are alive before the first statement. Further:

    1. `sink` parameters are immutable and escapable.

    2. `inout` parameters are mutable and escapable. They must be alive at the end of every terminating execution path.

    3. `assign` parameters are dead and mutable. They must be alive at the end of every terminating execution path.

6.  A `let` subscript implementation or an `inout` subscript implementation must have exactly one a yield statement on every terminating execution path. Given a subscript declaration with an output type `A`, a `let` subscript implementation must yield an immutable projection of an object whose type is subtype of `A`, unless the output signature of the subscript declaration is prefixed by `var`. In that case it must yield an mutable projection of an object of type `A`. An `inout` subscript implementation must yield a mutable projection of an object of type `A`.

7. A `sink` subscript implementation must have a return statement on every terminating execution path. It must return an escapable object whose type is subtype of the output type of the containing subscript declaration.

let+assign = inout
    ```

## Parameter declarations

1. Parameter declarations have the form:

    ```ebnf
    param-list ::=
      param-decl (',' param-decl)?

    param-decl ::=
      (identifier | '_') identifier? (':' param-type-expr)? default-value?

    default-value ::=
      '=' expr
    ```

1. If the declaration contains two identifiers, the first is used as an argument label and the second is used as a parameter name. Otherwise, the same identifier is used as both an argument label and as a parameter name. If the declaration contains a wildcard followed by an identifier, it defines a positional parameter. The identifier is used as a parameter name.

2. Parameter declarations define the passing convention of the arguments to the declared parameters in a function call:

    1. A parameter declared without any explicit convention is called a `let` parameter.

    2. A parameter declared with the `sink` convention is called a `sink` parameter.

    3. A parameter declared with the `inout` convention is called an `inout` parameter.

    4. A parameter declared with the `assign` convention is called an `assign` parameter.

3. A `let` parameter of a `sink` parameter may have a default value.

4. A default value must be a non-consuming expression. A default value to a `sink` parameter must evlauate to an escapable object.

## Capture lists

1. Capture lists have the form:

    ```ebnf
    capture-list ::=
      '[' binding-decl (',' binding-decl)* ']'

2. The bindings of a capture list may not have access or member modifiers.

# Statements

## General

1. Statements of the form:

    ```ebnf
    stmt ::=
      brace-stmt
      loop-stmt
      jump-stmt
      decl
      expr
    ```

2. Except as indicated, statements are executed in sequence.

3. Statements do not require explicit statement delimiter. Semicolons can be used to separate statements explicitly, for legibility or to disambiguate exceptional situations. [Note: A common practice is to write each statement on a new line.]

## Brace statements (a.k.a. code blocks)

1. Brace statements are sequences of statements executed in a lexical scope.

2. Brace statements have the form:

    ```ebnf
    brace-stmt ::=
      '{' (stmt | ';')* '}'
    ```

3. The statements contained in a brace statements are called its sub-statements.

4. Control enters the lexical scope of a brace statement before executing any sub-statements and exits that lexical scope when it reaches the end of the brace statement.

## Loop statments

### General

1. Loop statements describe iteration. They have the form:

    ```ebnf
    loop-stmt ::=
      do-while-stmt
      while-stmt
      for-stmt
    ```

2. A loop statement introduces a lexical scope. The body of a loop is a brace statement lexically nested inside the loop's scope.

3. A loop has three control entry points: a head, a body, and a tail. Control enters a loop from its head. The head belongs to the loop scope and the tail belongs to the body's scope. When a loop statement is executed, control is transferred to its head, entering the loop's scope. If control reaches the end of a loop's body, it is unconditionally transferred to its tail. The body's scope and then the loop's scope are exited when control exits the tail, whether or not it is transferred back to the head.

4. A continuation test is a procedure that determines whether an additional iteration should take place, or whether control should exit the loop. A continuation test may take in the head or in the body of a loop.

### Do-while statments

1. `do-while` statements have the form:

    ```ebnf
    do-while-stmt ::=
      'do' brace-stmt 'while' expr
    ```

2. The condition of a `do-while` statement belongs to the tail of the loop. It must be an expression of type `Bool`, which is evaluated by each continuation test. The test succeeds if the condition evaluates to `true`. That value is not consumed.

3. (Example)

    ```val
    var counter = 0
    do {
      counter += 1
      let x = counter
    } while x < 3
    ```

    The binding `x` that occurring in the condition of the `do-while` statement is declared it its body.

4. The head of a `do-while` statement unconditionally transfers control to the body of the loop. The tail performs a continuation test. If it succeeds, control is transferred back to the head. Otherwise, it exits the loop.

### While statments

1. `while` statements have the form:

    ```ebnf
    while-stmt ::=
      'while' while-condition-list brace-stmt

    while-condition-list ::=
      while-condition-item (',' while-condition)*

    while-condition-item ::=
      binding-decl
      expr
    ```

2. The condition of a `while` belongs to the head of the loop. It is a non-empty sequence of condition items. A condition item that is a binding declaration is considered satisfied if and only if the value of the initializer matches the pattern and can initialize its new bindings. A condition item that is an expression must be of type `Bool` and is considierd satisfied if and only if it evaluates to `true`. That value is not consumed. If the condition contains more than a single item, the n+1th item is evaluated if and only if the nth item is satisfied. The continuation test succeeds if and only if all items are satisfied.

3. The head of a `while` statement performs a continuation test. If it succeeds, control is transferred to the body of the loop. Otherwise, it exits the loop. The tail unconditionally transfers control back to the head.

### For statements

1. `for` statements have the form:

    ```ebnf
    for-stmt ::=
      'for' for-binding-decl for-range loop-filter? brace-stmt

    for-binding-decl ::=
      binding-head binding-type-annotation?

    for-range ::=
      'in' expr

    loop-filter ::=
      'where' expr
    ```

2. (Example)

    ```val
    fun main() {
      var things: Array<Any> = [1, "abc", 3, 2]
      for inout x: Int in things where x < 3 {
        x += 1
      }
      print(things) // [2, "abc", 3, 3]
    }
    ```

3. The binding declaration and the filter expression of a `for` statement belong to the head of the loop. The range of a `for` statement belongs to the lexical scope in which that statement is defined.

4. (Example)

    ```val
    fun main() {
      for let x in 0 until x { print(x) }
    }
    ```

    This program is ill-typed. The range is referring to a binding introduced in the loop binding.

5. The range must be an expression of a type that conforms to the `Iterable` trait. When a `for` statement is executed, an iterator object `it` is produced by calling `make_iterator` on the object evaluated by the loop range. The continuation test consists of verifying that `it.next()` does not result in a `nil` object. The loop iterates over all elements produced by the iterator, unless it is exited early via a break or a return statement.

6. The head of a `for` statement performs a continuation test. If it fails, control exits the loop. Otherwise, it tests whether the latest object projected by the loop's iterator matches the pattern of the binding declaration and can initialize its new bindings. If and only if it does, the loop filter is evaluated. If and only if the filter evaluates to `true`, control enters the body of the loop. The value of the filter is not consumed. If either the pattern of the binding declaration does not match, or the filter is not satisfied, control is transferred back the head.

7.  (Example)

    ```val
    fun main() {
      var things: Array<Any> = [1, "abc", 3, 2]
      for inout x: Int in things where x < 3 {
        x += 1
      }
      print(things) // [2, "abc", 3, 3]
    }
    ```

    This program can be understood as though it had be written:

    ```val
    fun main() {
      var things: Array<Any> = [1, "abc", 3, 2]
      var iterator = things.make_iterator()
      while true {
        if inout x: Int = iterator.next() where x < 3 {
          x += 1
        } else {
          break
        }
      }
      print(things) // [2, "abc", 3, 3]
    }
    ```

## Jump statements

### General

1. Jump statements unconditionally transfer control. They have the form:

    ```ebnf
    jump-stmt ::=
      'return' expr?
      'yield' expr
      'break'
      'continue'
    ```

2. `break` and `continue` statements are called loop jump statements. A loop jump statement applies to the innermost loop.

### Return statments

1. Return statements return an object from a function, terminating the execution path and transferring control back to the function's caller.

2.  The expression in a return statement is called its operand. If the operand is omitted, it is interpreted as `()`. A return statement consumes the value of the operand to initialize an escapable object as result of the call to the containing function.

### Yield statments

1. Yield statements project an object out of a subscript, suspending the execution path and temporarily transferring control to the subscript's caller. Control comes back to the subscript once after the last use of the yielded projection at the call site, resuming execution at the statement that directly follows the yield statement.

2. (Example)

    ```val
    subscript element<A>(at: index, in array: Array<A>): T {
      print("will yield")
      yield array[position]
      if a > b { yield a } else { yield b }
      print("did yield")
    }

    fun main() {
      let fruits = ["apple", "mango", "orange"]
      let f = element(at: 1, in: fruits) // "will yield"
      print("foo")                       // "foo"
      print(f)                           // "mango"
                                         // "did yield"
      print("bar")                       // "bar"
    }
    ```

3. The expression in a yield statement is called its operand. A yield statement projects the value of its operand as the result of the call to the containing subscript. The yielded object is projected immutably in a `let` subscript implementation and mutably in an `inout` subscript implementation. The mutability marker `&` must prefix a mutable projection.

### Break statements

1. Break statements exit a loop. Control is transferred to the statement immediately following the loop, if any.

### Continue statements

1. Continue statements skip the remainder of a loop body. Control is transferred to the begin of the loop.

# Value expressions

## General

1. An expression is a sequence of operators and operands that specifies a computation. An expression results in a value and may cause side effects.

2. If during the evaluation of an expression, the result is not mathematically defined or not in the range of representable values for its type, the behavior is undefined.

## Properties of expressions

### Consuming expressions

1. An expression is consuming if and only if its evaluation may end the lifetime of one or objects not created by the expression's evaluation.

## Primary expressions

### General

1. Primary expressions have the form:

    ```ebnf
    primary-expr ::=
      scalar-literal
      compound-literal
      primary-decl-ref
      implicit-member-ref
      capture-expr
      lambda-expr
      async-expr
      await-expr
      selection-expr
      tuple-expr
      'nil'
      '_'
    ```

### Scalar literals

1. Scalar literals have the form:

    ```ebnf
    scalar-literal ::=
      boolean-literal
      integer-literal
      floating-point-literal
      string-literal
    ```

### Compound literals

#### General

1. Compound literals have the form:

    ```ebnf
    compound-literal ::=
      buffer-literal
      map-literal
    ```

2. A compound literal consumes the value of each of its components.

#### Buffer literals

1. Buffer literals have the form:

    ```ebnf
    buffer-literal ::=
      '[' buffer-component-list? ']'

    buffer-component-list ::=
      expr (',' expr)* ','?
    ```

2. The type of a buffer literal is `T[n]`, where `n` is the number of components in the literal and `A` the type of all components. If a buffer literal appears in a typed context, `A` may be inferred from that context. Otherwise, the literal may not be empty, the type of `A` is inferred from the first component, and all other components must have the same type. The implementation may issue a warning if the literal has no component.

3. (Example)

    ```val
    let a = []               // error: cannot infer empty buffer type without context
    let b = [1, 2]           // OK, 'b' has type 'Int[2]'
    let c = [1, 2.0]         // error: '2.0' does not have type 'Int'
    let d: Double[] = [1, 2] // OK, 'd' has type 'Double[2]'
    let e: Int[] = []        // warning: zero-lenght buffer
    ```

#### Map literal

1. Map literals have the form:

    ```ebnf
    map-literal ::=
      '[' map-literal-component-list ']'
      '[' ':' ']'

    map-component-list ::=
      map-component (',' map-component)* ','?

    map-component ::=
      expr ':' expr
    ```

2. A map literal is said to be empty if it has the form `[:]`.

3. The type of a map literal is `Map<Key, Value>`, where `Key` is the type of the map's keys and `Value` is the type of the map's values. If a map literal appears in a typed context, `Key` and `Value` may be inferred from that context. Otherwise, the literal may not be empty, the types of `Key` and `Value` are inferred from the first component, and all other components must have the same type.

4. (Example)

    ```val
    let a = [:]                // error: cannot infer empty map type without context
    let b = [1: "a", 2: "b"]   // OK, 'b' has type 'Map<Int, String>'
    let c = [1: "a", 2.0: "b"] // error: '2.0' does not have type 'Int'
    let d: Map<Double, String> = [1: "a", 2.0: "b"] // OK
    let e: Map<Double, String> = [:] // OK
    ```

### Primary declaration references

1. Primary declaration references have the form:

    ```ebnf
    primary-decl-ref ::=
      ident-expr type-argument-list?
    ```

### Identifiers

1. Identifiers have the form:

    ```ebnf
    ident-expr ::=
      entity-ident impl-ident?

    entity-ident ::=
      identifier
      function-entity-ident
      oper-notation operator

    function-entity-ident ::=
      identifier '(' argument-label+ ')'

    argument-label ::=
      (identifier | '_') ':'

    impl-ident ::=
      '.' method-introducer
    ```

2. An identifier may not have any whitespace between its constituent tokens.

3. An identifier that denotes a non-static binding member or a non-static method may only be used as part of a value member access. An identifier that denotes a static binding member or a static method may only be used as part of a type member access.

4. (Example)

    ``val
    type A {
      let m: Int
      static let n = 0
    }

    let foo = A(m: 0)
    let i = foo.m   // OK
    let j = A.m     // OK
    let k = foo.n   // error
    let l = A.n     // OK
    ```

5. An identifier may contain be suffixed by a method introducer if and only if its entity identifier refers to a method declaration for which an explicit or synthesized method implementation for the same method introducer exists.

6. (Example)

    ```val
    type Vector2 {

      var x: Double
      var y: Double

      fun scaled(by factor: Double) -> Self {
        let   { Vector2(x: x * factor, y: y * factor) }
        inout { x *= factor; y *= factor }
      }

    }

    let f = Vector2.scaled(by:).let  // OK
    let g = Vector2.scaled(by:).sink // OK
    ```

## Compound expressions

### Member accesses

#### General

1. Member accesses have the form:

    ```ebnf
    value-member-expr ::=
      expr '.' primary-decl-ref
      type-expr '.' primary-decl-ref
    ```

2. A member access whose declaration reference denotes a non-static member binding or a non-static method is called a value member access. A member access whose declaration reference denotes a a static binding member or a static method is called a type member access.

3. A value member access whose base is an expression is called an unbound value member access.

### Function calls

#### General

1. Function calls have the form:

    ```ebnf
    call-expr ::=
      expr '(' call-argument-list? ')'

    call-argument-list ::=
      call-argument ( ',' call-argument )*

    call-argument :=
      (identifier ':')? expr
    ```

    The opening parenthesis preceding the call argument list must be on the same line as the callee.

2. The kind of the call expression depends on its callee:

    1. if the callee is a type expression, the call expression is an initializer call; or

    2. if the callee is a value member expression, the call expression is a method call; or

    3. if the callee is a type member expression referring to a function declaration, the call expression is a static method call; or

    4. otherwise, the call expression is a function call.

3. Arguments to `sink` parameters are consumed. Arguments to `let` parameters are projected immutably in the entire call expression. Arguments to `inout` and `set` parameters are projected mutably in the entire call expression.

## Operators

### Operator notations

1. Operator notations have the form:

    ```ebnf
    oper-notation ::= (one of)
      infix prefix postfix
    ```

# Type expressions

## Tuple types

1. A tuple type is a structural type composed of zero or more ordered *operands*. An operand is a type together with an optional label.

## Function and lambda types

## Union types

## Existential types

## Type aliases
