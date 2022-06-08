<!--#region:intro-->
# Regular Expression Atomic Operators for ECMAScript

This proposal seeks to introduce syntax to ECMAScript regular expressions to control backtracking in certain scenarios
by treating certain portions of a pattern as "atomic", where backtracking information specific to that portion of the
pattern is discarded when it matches successfully.

For example, currently the pattern `a(bc|b)c` will match both `"abcc"` and `"abc"`. In the case of `"abcc"`, we 
match the term `a`, then the alternative `bc`, and finally the term `c`. In the case of `"abc"`, we match the
term `a`, then the alternative `bc` (consuming the rest of the input), but fail to match the final term `c`. Since the
rest of the match failed, we ***backtrack*** to the beginning of the disjunction `bc|b` and attempt the alternative. We then
match the alternative `b`, and finally the term `c`.

In some cases such backtracking is unwanted or is potentially catastrophic. To address such cases, we propose the 
addition of Atomic Groups (`(?> Disjunction )`) and Possessive Quantifiers (`n*+`, `n++`, etc.). In an _Atomic Group_
we discard backtracking information for the group when it matches successfully. If the rest of the pattern following
the atomic group fails to match, we would no longer backtrack to the beginning of the group in an attempt to process
any alternatives.

In the case of `a(?>bc|b)c`, we would still match `"abcc"` but would no longer match `"abc"`. In the case of `"abc"`,
we match the term `a`, then the alternative `bc`, but fail to match the term `c`. Since backtracking information for
the disjunction `bc|b` was discarded upon the successful match of `bc`, we cannot attempt the alternative `b`. As a result,
the match for `"abc"` fails.

It is worth noting that lookaround operators (i.e., `(?=)`, `(?!)`, `(?<=)`, and `(?<!)`) are already atomic operations.

<!--#endregion:intro-->

<!--#region:status-->
## Status

**Stage:** 1  
**Champion:** Ron Buckton ([@rbuckton](https://github.com/rbuckton))  

_For detailed status of this proposal see [TODO](#todo), below._  
<!--#endregion:status-->

<!--#region:authors-->
## Authors

* Ron Buckton ([@rbuckton](https://github.com/rbuckton))  
<!--#endregion:authors-->

<!--#region:motivations-->
# Motivations

> NOTE: See https://github.com/rbuckton/proposal-regexp-features for an overview of
> how this proposal fits into other possible future features for Regular Expressions.

This proposal seeks to introduce new syntax to allow users fine control the performance
characteristics of regular expressions through possessive quantifiers and backtracking
control.

<!--#endregion:motivations-->

<!--#region:prior-art-->
# Prior Art 

* Possessive Quantifiers:
  * [Perl](https://rbuckton.github.io/regexp-features/engines/perl.html#feature-possessive-quantifiers)  
  * [PCRE](https://rbuckton.github.io/regexp-features/engines/pcre.html#feature-possessive-quantifiers)  
  * [Boost.Regex](https://rbuckton.github.io/regexp-features/engines/boost.regex.html#feature-possessive-quantifiers)  
  * [Oniguruma](https://rbuckton.github.io/regexp-features/engines/oniguruma.html#feature-possessive-quantifiers)  
  * [ICU](https://rbuckton.github.io/regexp-features/engines/icu.html#feature-possessive-quantifiers)  
  * [Glib/GRegex](https://rbuckton.github.io/regexp-features/engines/glib-gregex.html#feature-possessive-quantifiers)  
* Atomic Groups
  * [Perl](https://rbuckton.github.io/regexp-features/engines/perl.html#feature-non-backtracking-expressions)  
  * [PCRE](https://rbuckton.github.io/regexp-features/engines/pcre.html#feature-non-backtracking-expressions)  
  * [Boost.Regex](https://rbuckton.github.io/regexp-features/engines/boost.regex.html#feature-non-backtracking-expressions)  
  * [.NET](https://rbuckton.github.io/regexp-features/engines/dotnet.html#feature-non-backtracking-expressions)  
  * [Oniguruma](https://rbuckton.github.io/regexp-features/engines/oniguruma.html#feature-non-backtracking-expressions)  
  * [ICU](https://rbuckton.github.io/regexp-features/engines/icu.html#feature-non-backtracking-expressions)  
  * [Glib/GRegex](https://rbuckton.github.io/regexp-features/engines/glib-gregex.html#feature-non-backtracking-expressions)  

See https://rbuckton.github.io/regexp-features/features/possessive-quantifiers.html and
https://rbuckton.github.io/regexp-features/features/non-backtracking-expressions.html for 
additional information.
<!--#endregion:prior-art-->

<!--#region:syntax-->
# Syntax

## Atomic Groups

An Atomic Group is a non-backtracking expression which is matched independent of
neighboring patterns, and will not backtrack in the event of a failed match. This
is often used to improve performance.

- `(?>pattern)` &mdash; Matches the provided pattern, but no backtracking is performed 
  if the match fails.

> NOTE: This has no conflicts with existing syntax, as ECMAScript currently produces an
  error for this syntax in both `u` and non-`u` modes.

## Possessive Quantifiers

Possessive quantifiers are like normal (a.k.a. "greedy") quantifiers, but do not
backtrack if the rest of the pattern to the right fails to match. Possessive
quantifiers are often used as a performance tweak to avoid catastrophic backtracking.

- `*+` &mdash; Match zero or more instances of the preceding atom without backtracking.
- `++` &mdash; Match one or more instances of the preceding atom without backtracking.
- `?+` &mdash; Match zero or one instances of the preceding atom without backtracking.
- `{n}+` &mdash; Where _n_ is an integer. Matches the preceding atom exactly _n_ times without backtracking.
- `{n,}+` &mdash; Where _n_ is an integer. Matches the preceding atom at-least _n_ times without backtracking.
- `{n,m}+` &mdash; Where _n_ and _m_ are integers, and _m_ >= _n_. Matches the preceding atom at-least _n_ times and at-most _m_ times without backtracking.

> NOTE: This has no conflicts with existing syntax, as ECMAScript currently produces
> an error for this syntax in both `u` and non-`u` modes.

Possessive quantifiers are syntactic sugar over `(?> Disjunction )`:

- `atom*+` is equivalent to `(?>atom*)`
- `atom++` is equivalent to `(?>atom+)`
- `atom?+` is equivalent to `(?>atom?)`
- `atom{n}+` is equivalent to `(?>atom{n})`
- `atom{n,}+` is equivalent to `(?>atom{n,})`
- `atom{n,m}+` is equivalent to `(?>atom{n,m})`

<!--#endregion:syntax-->

<!--#region:semantics-->
<!-- # Semantics -->


<!--#endregion:semantics-->

<!--#region:examples-->
# Examples

```js
// NOTE: x-mode flag used to illustrate difference
// without atomic groups:
const re1 = /\((      [^()]+   | \([^()]*\))+ \)/x;
re1.test("((()aaaaaaaaaaaaaaaaaaaaaaaaaaaaa"); // can take several seconds to fail

// with atomic groups
const re2 = /\((  (?> [^()]+ ) | \([^()]*\))+ \)/x;
//                ^^^--------^ atomic group
re2.test("((()aaaaaaaaaaaaaaaaaaaaaaaaaaaaa"); // significantly faster as less backtracking is involved

// with posessive quantifiers
const re2 = /\((      [^()]++  | \([^()]*\))+ \)/x;
//                         ^^- possessive quantifier
re2.test("((()aaaaaaaaaaaaaaaaaaaaaaaaaaaaa"); // significantly faster as less backtracking is involved
```

See https://github.com/rbuckton/proposal-regexp-x-mode for more information on the `x` flag.

<!--#endregion:examples-->

<!--#region:api-->
<!-- # API -->

<!--#endregion:api-->

<!--#region:grammar-->
<!-- # Grammar

```grammarkdown
``` -->
<!--#endregion:grammar-->

<!--#region:references-->
<!-- # References

> TODO: Provide links to other specifications, etc.

* [Title](url)   -->
<!--#endregion:references-->

# History

- October 28, 2021 &mdash; Proposed for Stage 1 ([slides](https://1drv.ms/p/s!AjgWTO11Fk-TkfoUE-yb6OHZGaPtvw?e=mJQWJ8))
  - Outcome: Remained at Stage 0

<!--#region:todo-->
# TODO

The following is a high-level list of tasks to progress through each stage of the [TC39 proposal process](https://tc39.github.io/process-document/):

### Stage 1 Entrance Criteria

* [x] Identified a "[champion][Champion]" who will advance the addition.  
* [x] [Prose][Prose] outlining the problem or need and the general shape of a solution.  
* [x] Illustrative [examples][Examples] of usage.  
* [ ] ~~High-level [API][API].~~  

### Stage 2 Entrance Criteria

* [x] [Initial specification text][Specification].  
* [ ] [Transpiler support][Transpiler] (_Optional_).  

### Stage 3 Entrance Criteria

* [x] [Complete specification text][Specification].  
* [ ] Designated reviewers have [signed off][Stage3ReviewerSignOff] on the current spec text.  
* [ ] The ECMAScript editor has [signed off][Stage3EditorSignOff] on the current spec text.  

### Stage 4 Entrance Criteria

* [ ] [Test262](https://github.com/tc39/test262) acceptance tests have been written for mainline usage scenarios and [merged][Test262PullRequest].  
* [ ] Two compatible implementations which pass the acceptance tests: [\[1\]][Implementation1], [\[2\]][Implementation2].  
* [ ] A [pull request][Ecma262PullRequest] has been sent to tc39/ecma262 with the integrated spec text.  
* [ ] The ECMAScript editor has signed off on the [pull request][Ecma262PullRequest].  
<!--#endregion:todo-->

<!-- The following links are used throughout the README: -->

[Process]: https://tc39.es/process-document/
[Proposals]: https://github.com/tc39/proposals/
[Grammarkdown]: http://github.com/rbuckton/grammarkdown#readme
[Champion]: #status
[Prose]: #motivations
[Examples]: #examples
[API]: #api
[Specification]: https://rbuckton.github.io/proposal-regexp-atomic-operators

[Transpiler]: #todo
[Stage3ReviewerSignOff]: #todo
[Stage3EditorSignOff]: #todo
[Test262PullRequest]: #todo
[Implementation1]: #todo
[Implementation2]: #todo
[Ecma262PullRequest]: #todo
