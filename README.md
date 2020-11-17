# ECMAScript proposal: set notation in regular expressions

## Authors

- Markus Scherer
- Mathias Bynens

## Status

This proposal is at stage 0 of [the TC39 process](https://tc39.es/process-document/).

## Summary

In ECMAScript regex character classes, we propose to add syntax & semantics for the following set operations:

- difference/subtraction
- intersection
- nested character classes

## Motivation

Many regular expression engines support named character properties, mostly reflecting Unicode character properties, to avoid hardcoding character classes that may require hundreds of ranges and that may change with new versions of Unicode.

However, a character property is often just a starting point. It is common to need additions (union), exceptions (subtraction), and “both this and that” (intersection). See the recommendation to support set operations in [UTS #18: Unicode Regular Expressions](https://www.unicode.org/reports/tr18/#Subtraction_and_Intersection).

ECMAScript regular expression patterns already support one set operation in limited form: one can create a union of characters, ranges, and classes, as long as those classes are `CharacterClassEscape`s like `\s` or `\p{Decimal_Number}`.

A web search for questions about regular expressions with such set operations reveals workarounds such as hardcoding the ranges resulting from set operations (losing the benefits of named properties) and lookahead assertions (which are unintuitive for this purpose and perform less well).

We propose adding syntax & semantics for difference and intersection, as well as nested character classes.

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
    [\p{Other}\p{Separator}\p{White_Space}\p{Default_Ignorable_Code_Point}--[\x20]]
    ```

- Looking for “first letter in each script” starting from:

    ```
    [\P{NFC_Quick_Check=No}--\p{Script=Common}--\p{Script=Inherited}--\p{Script=Unknown}]
    ```

    Note that ECMAScript currently doesn’t support [`\p{NFC_Quick_Check=…}`](https://www.unicode.org/reports/tr15/#Quick_Check_Table) — this is an illustrative example regardless.

Several other regex engines support some or all of the proposed extensions in some form:

| language/implementation                                                                                                                      | union | subtraction      | intersection | nested classes | symmetric difference |
| -------------------------------------------------------------------------------------------------------------------------------------------- | ----- | ---------------- | ------------ | -------------- | -------------------- |
| [ICU regex](https://unicode-org.github.io/icu/userguide/strings/regexp.html#set-expressions-character-classes)                               | ✅    | ✅               | ✅           | ✅             | ❌                   |
| [`java.util.regex.Pattern`](https://docs.oracle.com/en/java/javase/15/docs/api/java.base/java/util/regex/Pattern.html)                       | ✅    | 🤷 <sup>\*</sup> | ✅           | ✅             | ❌                   |
| [Perl (“experimental feature available starting in 5.18”)](https://perldoc.perl.org/perlrecharclass#Extended-Bracketed-Character-Classes)    | ✅    | ✅               | ✅           | ✅             | ✅                   |
| [.Net](https://docs.microsoft.com/en-us/dotnet/standard/base-types/character-classes-in-regular-expressions#CharacterClassSubtraction)       | ✅    | ✅               | ❌           | ✅             | ❌                   |
| [XML Schema](https://www.w3.org/TR/xmlschema-2/#charcter-classes)                                                                            | ✅    | ✅               | ❌           | ✅             | ❌                   |
| [Apache Xerces2 XPath regex](https://xerces.apache.org/xerces2-j/javadocs/xerces2/org/apache/xerces/impl/xpath/regex/RegularExpression.html) | ✅    | ✅               | ✅           | ✅             | ❌                   |
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

## Proposed solution

We propose to extend the syntax for character classes to add support for set difference/subtraction, set intersection, and nested character classes.

## High-level API

…

### FAQ

…

## Illustrative examples

### Matching emoji sequences

…

### Matching hashtags

…

## TC39 meeting notes

- TBD

## Specification

* [Ecmarkup source](https://github.com/mathiasbynens/proposal-regexp-set-notation/blob/main/spec.html)
* [HTML version](https://mathiasbynens.github.io/proposal-regexp-set-notation/)

## Implementations

* none yet
