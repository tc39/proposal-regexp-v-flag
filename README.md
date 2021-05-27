# ECMAScript proposal: set notation in regular expressions

## Authors

- Markus Scherer
- Mathias Bynens

## Status

This proposal is at stage 2 of [the TC39 process](https://tc39.es/process-document/).

As of the 2021-may-25 TC39 meeting, this proposal officially subsumes the [properties of strings proposal](https://github.com/tc39/proposal-regexp-unicode-sequence-properties).

## Summary

In ECMAScript regex character classes, we propose to add syntax & semantics for the following set operations:

- difference/subtraction (_in A but not in B_)
- intersection (_in both A and B_)
- nested character classes (_needed to enable the above_)

In addition, by merging with the [properties of strings proposal](https://github.com/tc39/proposal-regexp-unicode-sequence-properties), we also propose to add certain Unicode properties of strings, and string literals in character classes.

## Motivation

Many regular expression engines support named character properties, mostly reflecting Unicode character properties, to avoid hardcoding character classes that may require hundreds of ranges and that may change with new versions of Unicode.

However, a character property is often just a starting point. It is common to need additions (union), exceptions (subtraction), and “both this and that” (intersection). See the recommendation to support set operations in [UTS #18: Unicode Regular Expressions](https://www.unicode.org/reports/tr18/#Subtraction_and_Intersection).

ECMAScript regular expression patterns already support one set operation in limited form: one can create a union of characters, ranges, and classes, as long as those classes are `CharacterClassEscape`s like `\s` or `\p{Decimal_Number}`.

A web search for questions about regular expressions with such set operations reveals workarounds such as hardcoding the ranges resulting from set operations (losing the benefits of named properties) and lookahead assertions (which are unintuitive for this purpose and perform less well).

We propose adding syntax & semantics for difference and intersection, as well as nested character classes.

## Proposed solution

We propose to extend the syntax for character classes to add support for set difference/subtraction, set intersection, and nested character classes.

## High-level API

Within regular expression patterns, we propose enabling the following functionality.
We have not yet settled on several issues including single vs. double punctuation,
some distinct prefix to not break existing expressions (see FAQ), etc.

```
// difference/subtraction
[A--B]

// intersection
[A&&B]

// nested character class
[A--[0-9]]
```

Throughout these high-level examples, `A` and `B` can be thought of as placeholders for a character class (e.g. `[a-z]`) or a property escape (e.g. `\p{ASCII}`) and maybe (subject to discussion of specifics) single characters and/or character ranges. See [the illustrative examples section](#illustrative-examples) for concrete real-world use cases.

## Illustrative examples

Real-world usage examples from code using ICU’s `UnicodeSet` which implements a pattern syntax similar to regex character classes (modified here to use `\p{Perl syntax for properties}` rather than `[:POSIX syntax for properties:]` — `UnicodeSet` supports both):

- Code that looks for non-ASCII digits, to convert them to ASCII digits:

    ```
    [\p{Decimal_Number}--[0-9]]
    ```

- Looking for spans of "word/identifier letters" of specific scripts:

    ```
    [\p{Script=Khmer}&&[\p{Letter}\p{Mark}\p{Number}]]
    ```

- Looking for “breaking spaces”:

    ```
    [\p{White_Space}--\p{Line_Break=Glue}]
    ```

    Note that ECMAScript currently doesn’t support `\p{Line_Break=…}` — this is an illustrative example regardless.

- Looking for emoji characters except for the ASCII ones:

    ```
    [\p{Emoji}--[#*0-9]]

    // …or…

    [\p{Emoji}--\p{ASCII}]
    ```

- Looking for non-script-specific combining marks:

    ```
    [\p{Nonspacing_Mark}&&[\p{Script=Inherited}\p{Script=Common}]]
    ```

- Looking for “invisible characters” except for ASCII space:

    ```
    [[\p{Other}\p{Separator}\p{White_Space}\p{Default_Ignorable_Code_Point}]--\x20]
    ```

- Looking for “first letter in each script” starting from:

    ```
    [\P{NFC_Quick_Check=No}--\p{Script=Common}--\p{Script=Inherited}--\p{Script=Unknown}]
    ```

    Note that ECMAScript currently doesn’t support [`\p{NFC_Quick_Check=…}`](https://www.unicode.org/reports/tr15/#Quick_Check_Table) — this is an illustrative example regardless.

- All Greek code points that are either a letter, a mark (e.g. diacritic), or a decimal number:

    ```
    [\p{Greek}&&[\p{Letter}\p{Mark}\p{Decimal_Number}]]
    ```

- All code points, except for those in the “Other” `General_Category`, but add back control characters:

    ```
    [[\p{Any}--\p{Other}]\p{Control}]
    ```

- All assigned code points, except for separators:

    ```
    [\p{Assigned}--\p{Separator}]
    ```

- All right-to-left and Arabic Letter code points, but remove unassigned code points:

    ```
    [[\p{Bidi_Class=R}\p{Bidi_Class=AL}]--\p{Unassigned}]
    ```

    Note that ECMAScript currently doesn’t support [`\p{Bidi_Class=…}`](https://www.unicode.org/reports/tr44/#Bidi_Class) — this is an illustrative example regardless.

- All right-to-left and Arabic Letter code points with `General_Category` “Letter”:

    ```
    [\p{Letter}&&[\p{Bidi_Class=R}\p{Bidi_Class=AL}]]
    ```

    Note that ECMAScript currently doesn’t support [`\p{Bidi_Class=…}`](https://www.unicode.org/reports/tr44/#Bidi_Class) — this is an illustrative example regardless.

- All characters in the “Other” `General_Category` EXCEPT for format and control characters (or, equivalently, all surrogate, private use, and unassigned code points):

    ```
    [\p{Other}--\p{Format}--\p{Control}]
    ```


## FAQ

### Is the new syntax backwards-compatible? Do we need another regular expression flag?

It is an explicit goal of this proposal to not break backwards compatibility. Concretely, we don’t want to change behavior of any regular expression pattern that currently does not throw an exception. There needs to be some way to indicate that the new syntax is in use.

We considered 3 options:
- A new flag outside the expression itself.
- A modifier inside the expression, of the form `(?L)` where `L` is one ASCII letter. (Several regex engines support various modifiers like this.)
- A prefix like `\U...` that is not valid under the current `u` flag (Unicode mode) – but note that `\U` without the `u` flag is just the same as `U` itself.
  - (Banning the use of unknown escape sequences in `u` RegExps was [a conscious choice](https://web.archive.org/web/20141214085510/https://bugs.ecmascript.org/show_bug.cgi?id=3157), made to enable this kind of extension.)

The prefix was suggested in an early TC39 meeting, so we were working with variations of that, for example:
```
UnicodeCharacterClass = '\UniSet{' ClassContents '}'
```

However, we found that this is not very developer-friendly.

In particular, one would have to write the prefix **and** use the ‘u’ flag. Waldemar pointed out that the prefix *looks like* it should be enough, and therefore a developer may well accidentally omit adding the `u` flag.

Also, the use of a backslash-letter prefix would want to enclose the new syntax in `{curly braces}` because other such syntax (`\p{property}`, `\u{12345}`, ...) uses curly braces – but not using `[square brackets]` for the outermost level of a character class looks strange.

Finally, when an expression has several new-syntax character classes, the prefix would have to be used on each one, which is clunky.

An in-expression modifier is an attractive alternative, but ECMAScript does not yet use any such modifiers.

Therefore, a new flag is the simplest, most user-friendly, and syntactically and semantically cleanest way to indicate the new character class syntax. It should **imply and build on** the `u` flag.

We suggest using flag `v` for the next letter after `u`.

We also suggest that the [proposed properties of strings](https://github.com/tc39/proposal-regexp-unicode-sequence-properties) require use of this same new flag.

In other words, the new flag would indicate several connected changes related to properties and character classes:
- properties of strings
- character classes may contain multi-character-string elements, via string literals or certain properties
- nested classes
- set operators
- simpler parsing of dashes and square brackets

For more discussion see [issue 2](https://github.com/tc39/proposal-regexp-set-notation/issues/2).

### What’s the precedent in other RegExp flavors?

Several other regex engines support some or all of the proposed extensions in some form:

| language/implementation                                                                                                                      | union | subtraction      | intersection | nested classes | symmetric difference |
| -------------------------------------------------------------------------------------------------------------------------------------------- | ----- | ---------------- | ------------ | -------------- | -------------------- |
| [ICU regex](https://unicode-org.github.io/icu/userguide/strings/regexp.html#set-expressions-character-classes)                               | ✅    | ✅               | ✅           | ✅             | ❌                   |
| [`java.util.regex.Pattern`](https://docs.oracle.com/en/java/javase/15/docs/api/java.base/java/util/regex/Pattern.html)                       | ✅    | 🤷 <sup>\*</sup> | ✅           | ✅             | ❌                   |
| [Perl (“experimental feature available starting in 5.18”)](https://perldoc.perl.org/perlrecharclass#Extended-Bracketed-Character-Classes)    | ✅    | ✅               | ✅           | ✅             | ✅                   |
| [.Net](https://docs.microsoft.com/en-us/dotnet/standard/base-types/character-classes-in-regular-expressions#CharacterClassSubtraction)       | ✅    | ✅               | ❌           | ✅             | ❌                   |
| [XML Schema](https://www.w3.org/TR/xmlschema-2/#charcter-classes)                                                                            | ✅    | ✅               | ❌           | ✅             | ❌                   |
| [Apache Xerces2 XPath regex](https://xerces.apache.org/xerces2-j/javadocs/xerces2/org/apache/xerces/impl/xpath/regex/RegularExpression.html) | ✅    | ✅               | ✅           | ✅             | ❌                   |
| [Python regex module](https://pypi.org/project/regex/) (not built-in "re")                                                                   | ✅    | ✅               | ✅           | ✅             | ✅                   |
| [Ruby Regexp](https://docs.ruby-lang.org/en/2.0.0/Regexp.html#class-Regexp-label-Character+Classes)                                          | ✅    | ❌               | ✅           | ❌             | ❌                   |
| ECMAScript prior to this proposal                                                                                                            | ✅    | ❌               | ❌           | ❌             | ❌                   |
| ECMAScript with this proposal                                                                                                                | ✅    | ✅               | ✅           | ✅             | ❌                   |

<sup>\*</sup> Subtraction is [documented](https://docs.oracle.com/en/java/javase/15/docs/api/java.base/java/util/regex/Pattern.html#subtraction1) as intersection with negation. With only support for negation + nested classes, you already have the functional equivalent of intersection & subtraction: `[^[^ab][^cd]] === [[ab]&&[cd]]` and `[^[^ab][cd]] === [[ab]--[cd]]`. This is just not very readable. For this reason, our proposal includes dedicated syntax for intersection and subtraction as well.

These all differ somewhat in syntax and semantics (e.g. operator precedence). References:

- [regular expression flavors that support character class subtraction](https://www.regular-expressions.info/charclasssubtract.html)
- [regular expression flavors that support character class intersection](https://www.regular-expressions.info/charclassintersect.html)

Some Stack Overflow discussions:

- [#3201689](https://stackoverflow.com/q/3201689/96656)
- [#10777728](https://stackoverflow.com/q/10777728/96656)
- [#15930181](https://stackoverflow.com/q/15930181/96656)
- [#17327765](https://stackoverflow.com/q/17327765/96656)
- [#29859968](https://stackoverflow.com/q/29859968/96656)
- [#44771741](https://stackoverflow.com/q/44771741/96656)
- [#55095497](https://stackoverflow.com/q/55095497/96656)

### How does this interact with [properties of strings a.k.a. the sequence properties proposal](https://github.com/tc39/proposal-regexp-unicode-sequence-properties)?

We described the exact interactions between the two proposals on the path to stage 2. (See [issue #3](https://github.com/tc39/proposal-regexp-set-notation/issues/3) for background.)

We propose to require the new flag in order to enable properties-of-strings as well as allowing new-syntax character classes to contain multi-character-string elements (from string literals or properties-of-strings used inside a class).

### Can a property of strings change into a property of characters, or vice versa?

Short answer: no.

Long answer: We brought this up with the Unicode Technical Committee (UTC) in May 2019 (see [L2/19-168](https://www.unicode.org/cgi-bin/GetMatchingDocs.pl?L2/19-168) + [meeting notes](https://www.unicode.org/L2/L2019/19122.htm#:~:text=45-,B.13.8%20Supporting,Action%20Item%20for,-Mathias)), and later (in April 2021) proposed a concrete new stability policy (see [L2/21-091](https://www.unicode.org/cgi-bin/GetMatchingDocs.pl?L2/21-091) + [meeting notes](https://www.unicode.org/L2/L2021/21066.htm#:~:text=D.2%20Stability,C11%5D%20Consensus)). The UTC reached consensus to approve our proposal. The domain of a normative or informative Unicode property must never change. In particular, a property of characters must never be changed into a property of strings, and vice versa.

### What about symmetric difference?

We considered also proposing an operator for symmetric difference (see [issue #5](https://github.com/tc39/proposal-regexp-set-notation/issues/5)), but we did not find a good use case and wanted to keep the proposal simple.

Instead, we are proposing to reserve doubled ASCII punctuation and symbols for future use. That will allow for future proposals to add `~~` for example, as suggested in [UTS \#18](https://www.unicode.org/reports/tr18/#Subtraction_and_Intersection), for symmetric difference.

### Does this proposal affect ECMAScript lexing?

No. It’s an explicit goal of our proposal that a correct ECMAScript lexer before this proposal remains a correct ECMAScript lexer after this proposal.

## TC39 meeting notes + slides

- [November 2020](https://github.com/tc39/notes/blob/master/meetings/2020-11/nov-18.md#adopting-unicode-behavior-for-set-notation-in-regular-expressions) ([slides](https://docs.google.com/presentation/d/1kroPvRuQ8DMt6v2XioFmFs23J1drOK7gjPc08owaQOU/edit))
- [January 2021](https://github.com/tc39/notes/blob/master/meetings/2021-01/jan-28.md#regexp-set-notation-for-stage-1) ([slides](https://docs.google.com/presentation/d/1vXlLpf3mEa_8Y-GDiRKLCqSzNXPOKWCF7tPb0H2M9hk/edit))
- [March 2021](https://github.com/tc39/notes/blob/master/meetings/2021-03/mar-10.md#regexp-set-notation-update) ([slides](https://docs.google.com/presentation/d/1dWEHdfSsWPwoln5RD2dwnBomIIytX20UFzYSnmRsIPY/edit))
- [April 2021 Incubator Call](https://github.com/tc39/incubator-agendas/blob/master/notes/2021/04-08.md) ([slides](https://docs.google.com/presentation/d/1H2Doh8gbsRCthUoKCgFS-mTKxG2qY0vfC6D0dKvv4kc/edit))
- [April 2021](https://github.com/tc39/notes/blob/master/meetings/2021-04/apr-20.md#regexp-unicode-set-notation--properties-of-strings-update) ([slides](https://docs.google.com/presentation/d/1nV0NHUG5bd201rUSfJinLl8NTmnnyL5gTIhD0llsW1c/edit))
- May 2021 ([slides](https://docs.google.com/presentation/d/1nb_6ZcAjG4AKwVrwpalu1Ep-h7TONxoSm-uxKx83Wik/edit))

## Specification

Temporary:
* [Google Doc](https://docs.google.com/document/d/1Tbv3hfX9CxQtzH9r-JdxJsQZhmmDsidRUKKxg345JV0/edit)

TBD:
* [Ecmarkup source](https://github.com/tc39/proposal-regexp-set-notation/blob/main/spec.html)
* [HTML version](http://tc39.es/proposal-regexp-set-notation/)

## Implementations

* none yet
