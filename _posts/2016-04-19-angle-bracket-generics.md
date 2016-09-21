---
layout: post
title:  "Problems of Angle Bracket Generics"
date:   2016-04-19 15:00:00 +0200
categories: syntax
---

In the previous blog post, I talked about how using `<>` for Generics was really difficult to design. The main reason
for this are the ambiguities it introduces. In this blog post, I want to describe how other languages resolved this
problem and how it was ultimately implemented in Dyvil. Not because it is an extremely interesting or enlightening
story, but because it took me forever to figure out the dynamics of Angle Bracket Generics. If you are a language
designer or just making a programming language with generics for fun, then buckle up, because this is going to be a
tough ride.

## Tokenization

The first problem you encounter is tokenization. After a source file is loaded from the disk, the Dyvil Lexer processes
it and produces a list of tokens. This list is later  processed by the parser and converted to an Abstract Syntax Tree.
The lexer would convert the following type into the list of tokens:

```swift
Map<int, List<String>>
-> [ 'Map', '<', 'int', ',', 'List', '<', 'String', '>>' ]
```

You might notice that although `<>` work like brackets in types, the lexer treats them as identifiers. This means that,
instead of splitting `>>` into two `CLOSE_ANGLE` tokens, it creates an `IDENTIFIER '>>'` token. This is perfectly valid
in other contexts, where this might be used as an operator. At this point, the Parser has to do some magic to resolve
the issue: When it encounters a token that *starts with* a closing angle in a type context, it splits the token into two
parts: the `head` and the `tail`. The `head` is consumed, while the `tail` will be parsed later, maybe even by another
production. In Dyvil, we certain constructs where the Parser has to *try* to parse a production, and try the next one if
that fails. The most common example is Local Variables. Within a Statement List, a line could either be some expression
or a declaration:

```java
{
int i = 0
int add(int i, int j) = i + j
List<List<int>> myList = []
println i
new FooFactory().withName("test").forString("abc").deploy()
}
```

There is no straight-forward way to tell these expressions apart without parsing multiple tokens (at least two in these
cases). To make the parser less complicated, a so-called `TryParser` is used. It tries to parse a production
(declaration) up to a certain point where it is preferred over the fallback (expression). For variables, this is the `=`
sign; methods require the `(`. If the `TryParser` encounters a syntax error before the critical part, it aborts parsing
the declaration and restarts with the Expression parser.

Back to the token problem: Remember how token splitting changes the token sequence for the entire file? This
imperfection and our laziness to not use a fork system has come back to bite us: If some `>>` tokens were split in a
type context of a declaration, the `TryParser` has to *un-split* them in order to correctly parse an Expression.
Luckily, this un-splitting was merely a matter of keeping track of the original tokens and re-linking their `prev` and
`next` fields.

Implementation-wise, this creates yet another problem: To make the parsing code efficient, Token lists are maintained as
a doubly-linked list, where each token has a `next` and `prev` field. The `split` method changes these fields, mutating
the token chain as if there had been two separate tokens in the source. After figuring out how to handle token
splitting, I realized that it actually was the lesser of two evils spawned by Angle Bracket generics.

# Comparison Ambiguities

The aforementioned tokenization problem was obviously an implementation problem. But in the field of language design and
compiler construction, design is usually much harder than implementation. This is once again shown when looking at how
Angle Bracket Generics interact with Comparisons, which just happen to use the `<` and `>` symbols as well.

Consider the following expression:

```swift
println(Map<int, String>(1))
```

Without knowing that `Map`, `int` and `String` are all types, there are two different ways to parse this:

```swift
println(Map < int, String > 1) // call to println with two args, a less-than
                               // and a greater-than comparison
println(Map<int, String>(1))   // call to println with one arg, a call to
                               // Map.apply(int) with type arguments int
                               // and String
```

If you do not immediately see the difference: Congratulations, you just found the problem. One thing that I learned in
the two years of working with Dyvil is: If it's hard to parse for the compiler, it's probably hard to parse for humans
too. For a fairly long time, I was unsure how to resolve this issue. I knew that other languages like Swift and C# used
the same generic call syntax, but it was extremely hard to find out how *they* resolved the ambiguity.

For C#, I found the answer in the book [**Annotated C# Standard** by Jon Jagger, Nigel Perry and Peter Sestoft, page 78][1].
The resolution is done by examining the token after the closing `>`: If it is one of `(`, `)`, `]`, `:`, `;`, `,`, `.`,
`?`, `==` or `!=`, the expression is parsed as a generic method call. Otherwise, it represents two comparison operations.
This is actually a fairly straightforward solution, because it is *simple*. C# is a language that has to be easily
readable, not easily writable, and a limited set of conditions for a certain production helps readability.

Swift was a bit more tricky. It is a fairly young language, so there are way less resources that could provide insight
to such a small edge case. I found the answer in the [Swift compiler source code][2], nicely formatted in a comment:

```cpp
///   The generic-args case is ambiguous with an expression involving '<'
///   and '>' operators. The operator expression is favored unless a generic
///   argument list can be successfully parsed, and the closing bracket is
///   followed by one of these tokens:
///     lparen_following rparen lsquare_following rsquare lbrace rbrace
///     period_following comma semicolon
```

Once again, the language relies on the token after the closing `>`. However, the set of tokens that may follow a generic
method call is a bit different. Swift and C# both accept `)`, `]`, `,` and `;`. The `(`, `[` and `.` tokens are required
to be placed directly after the `>`, without whitespace separation. That means there is a difference between:

```swift
foo<int, String>()   /* and */ foo<int, String> ()
foo<int, String>[]   /* and */ foo<int, String> []
foo<int, String>.bar /* and */ foo<int, String> .bar
```

Also, Swift allows the token `{` and `}` after a generic call. The same rules are also employed in the Expression Parser
since Dyvil v0.21.0.

## Conclusion

I think at this point you should probably be saying "that's it, no generics with Angle Brackets!". And I wouldn't blame
you. Generics with Angle Brackets can create a lot of problems and ambiguities. But if you look at how big and
successful languages like Java, C# or Swift handle these problems, you realize that it is not impossible to deal with
these problems. After all, it's best to stick to languages and syntax styles that people are used to. Just be sure to
**document the behavior** of your language. Document it in your language specification or guides, not in a comment in
line 1536 of your closed compiler source (at least Swift is open source). And don't rely on others to do your job when
they write a book about it.

[1]: https://books.google.de/books?id=g6axWRRpJZwC&lpg=PA78&dq=c%23%20generic%20argument%20ambiguities&hl=de&pg=PA78#v=onepage&q&f=false
[2]: https://github.com/apple/swift/blob/swift-2.2-RELEASE/lib/Parse/ParseExpr.cpp#L1533
