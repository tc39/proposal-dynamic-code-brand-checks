# TC39 Proposal: Dynamic Code Brand Checks

## Table of Contents

- [Status](#status)
- [TL;DR](#tldr)
- [Motivation](#motivation)
- [Problem 1: %eval% does not accept objects in lieu of strings for code](#problem-1-eval-does-not-accept-objects-in-lieu-of-strings-for-code)
- [Problem 2: Host callout does not receive type information](#problem-2-host-callout-does-not-receive-type-information)
- [Problem 3: Host callout does not receive the full code to check](#problem-3-host-callout-does-not-receive-the-full-code-to-check)
- [Tests](#tests)

## Status

Champion(s): koto<br>
Author(s): mikesamuel, koto <br>
Stage: [1](https://tc39.github.io/process-document/) <br>
Spec: [ecmarkup output][draft spec], [source][]

## TL;DR

Allow hosts to create _code-like_ objects and change _HostEnsureCanCompileStrings( calleeRealm, parameterStrings, bodyString, direct )_ to _HostEnsureCanCompileStrings( calleeRealm, parameterStrings, bodyString, codeString, compilationType, parameterArgs, bodyArg )_.
Hosts can change the to code string to be compiled.

## Motivation

> ### `eval` is Evil
>
> The `eval` function is the most misused feature of JavaScript. Avoid it.
>
> -- <cite>Douglas Crockford, "JavaScript: The Good Parts"</cite>

`eval` and its friend `new Function` are problematic because, too
often, an attacker can turn it against the application.

Most code avoids `eval`, but JS programs are no longer small, and
self-contained as they were when Crock wrote that.

If one module uses `eval`, even if it's as simple as
[`Function('return this')()`][core-js-example] to get a handle to the
global object then `eval` has to work.

This prevents the use of security measures like:

- [Content-Security-Policy](https://csp.withgoogle.com/docs/index.html)
- `node --disallow_code_generation_from_strings`

which turn off `eval` globally.

As JavaScript programs get larger, the chance that no part of it needs `eval` or `Function()`
to operate gets smaller.

---

It is difficult in JavaScript for a code reviewer to determine that
code never uses these operators. For example, the below can when `x` is constructor.

```js
({}[x][x](y)());

// ({})[x]       === Object
// ({})[x][x]    === Function
// ({})[x][x](y)  ~  Function(y)
```

So code review and developer training are unlikely to prevent abuse of these
operators.

---

This aims to solve the problem by providing more context to host environments so that they
can make finer-grained trust decisions. Since strings in Javascript programs
often come from untrusted sources (DOM-Based XSS in web platform), the trust
decisions might be based on brand-checking _objects_ representing the code that
the application author trusts to be evaluated.

### Trusted Types

The [Trusted Types][tt] proposal ([explainer for TC39](https://docs.google.com/presentation/d/e/2PACX-1vQCbxmHKjPWUq7MC91x6tJKanFYU2i9Z13wwfkngcseHt96EfU_xyA0awkxb4SoNW3hQ3S2z-ByX0T9/pub?start=false&loop=false&delayms=60000)</a>) seeks to guard risky operations like dynamic code
evaluation by requiring that code portions have a runtime type that indicates that they have
been explicitly trusted.

Specifically, when Trusted Types enforcement is turned on, it would like to ensure
that arguments to <code>eval()</code> and <code>new Function()</code> are
(or are convertible to) a [TrustedScript](https://w3c.github.io/trusted-types/dist/spec/#typedefdef-trustedtype) - an object wrapping around a string stored in an internal,
immutable slot. This enhances the existing dynamic code evaluation guards that the web platform already has with [CSP](https://www.w3.org/TR/CSP3/) (Trusted Types also integrates with other CSP mechanisms).

This proposal seeks to solve the following problems:

## Problem 1: %eval% does not accept objects in lieu of strings for code

Currently, `typeof x === 'string' || eval(x) === x`. Eval exits early when its
argument is not a string.

For example:

```js
console.log(eval(123) === 123);

const array = ["alert(1)"];
console.log(eval(array) === array); // Does not alert

const touchy = {
  toString() {
    throw new Error();
  },
};
console.log(eval(touchy) === touchy); // Does not throw.
```

This follows from step 2 of the definition of [PerformEval](https://tc39.github.io/ecma262/#sec-performeval):

> ### Runtime Semantics: PerformEval ( _x_, _evalRealm_, _strictCaller_, _direct_ )
>
> <p>The abstract operation PerformEval with arguments *x*, *evalRealm*, *strictCaller*, and *direct* performs the following steps:</p>
>
> 1. Assert: If _direct_ is **false**, then _strictCaller_ is also **false**.
> 1. If Type(_x_) **is not String**, return _x_.

### Backwards compatibility constraints

To avoid breaking the web, `eval(x)` should do nothing different when x is a
non-string value that existing applications might create.

### Solution

Define a spec abstraction, _IsCodeLike_, that allows some object values through
but without changing the semantics of pre-existing programs.

- Define `IsCodeLike(*x*)`, a spec abstraction that returns true for objects
  containing a host-defined internal slot (`[[HostDefinedIsCodeLike]]`).
- Tweak `PerformEval`, which is called by both direct and indirect `eval`, to
  use `IsCodeLike(*x*)` to ensure Code-like objects will not cause the early-exit.

##### Pros

- `IsCodeLike` has no observable side effect for values. eval is backwards
  compatible when a program produces no code like values.
- User code can only create code-like values via host-defined APIs.

##### Cons

- Can't be tested in test262 (code-like object creation is host-defined).
- Code-like values are not proxyable.

## Problem 2: Host callout does not receive type information

Currently the information available to decide whether to allow compilation of
a string is a realm, a list of strings from parameters, a string from body, and a boolean.

> HostEnsureCanCompileStrings( __calleeRealm_, _parameterStrings_, _bodyString_, _direct_ )
>
> HostEnsureCanCompileStrings is an implementation-defined abstract
> operation that allows host environments to block certain ECMAScript
> functions which allow developers to compile strings into ECMAScript
> code.

For the hosts to be able to guard dynamic code evaluation effectively, additional
type information is needed.

### Solution

This proposal aims to provide additional context to `HostEnsureCanCompileStrings`,
and reorder the steps in `CreateDynamicFunction` so that `HostEnsureCanCompileStrings`
receives runtime type information - namely, whether the `eval` argument, or all
the Function constructor arguments passed the `IsCodeLike` check.

##### Pros

- No TOCTOU issues - code-like object passing the check is immediately
  stringified before the host callout happens.

##### Cons

- Requires changes to the host callout (see also below):
- Complex host interface; Requires passing objects to the host.

## Problem 3: Host callout does not receive the full code to check

`HostEnsureCanCompileStrings` is called with parameters for the source code, but they're not in a unified string.

### Solution

This proposal updates the host callout to contain the full code string to be executed,
and moves the callout in `CreateDynamicFunction` after the function body is
assembled.

## Tests

Related tests at

- https://github.com/tc39/test262/blob/master/test/language/eval-code/direct/non-string-object.js
- https://github.com/tc39/test262/blob/master/test/language/eval-code/indirect/non-string-object.js

but testing these require affecting the behavior of host callouts so will probably need to
be specified as [web-platform-tests](https://github.com/web-platform-tests/wpt).

[core-js-example]: https://github.com/zloirock/core-js/blob/2a005abe68520248d4431cab70d86e40b55d6e98/packages/core-js/internals/global.js#L5
[tt]: https://w3c.github.io/trusted-types/dist/spec/
[source]: https://github.com/tc39/proposal-dynamic-code-brand-checks/blob/master/spec.emu
[default policy]: https://w3c.github.io/trusted-types/dist/spec/#default-policy-hdr
[createscript callback]: https://w3c.github.io/trusted-types/dist/spec/#callbackdef-createscriptcallback
[draft spec]: https://tc39.es/proposal-dynamic-code-brand-checks/
