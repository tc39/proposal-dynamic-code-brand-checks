# TC39 Proposal: Dynamic Code Brand Checks

## Table of Contents
* [Status](#status)
* [Historical Note](#historical-note)
* [Motivation](#motivation)
* [Problem: %eval% does not accept objects in lieu of strings for code](#problem-eval-does-not-accept-objects-in-lieu-of-strings-for-code)
  + [Backwards compatibility constraints](#backwards-compatibility-constraints)
  + [Solution](#solution)
    - [Internal slot](#internal-slot)
      * [Pros](#pros)
    - [Well-known Symbol](#well-known-symbol)
      * [Pros](#pros-1)
* [Problem: Host callout does not receive type information](#problem-host-callout-does-not-receive-type-information)
  + [Solution](#solution-1)
* [Problem: host callout cannot adjust values](#problem-host-callout-cannot-adjust-values)
  + [Solution](#solution-2)
* [Tests](#tests)

## Status

Champion(s): mikesamuel <br>
Author(s): mikesamuel <br>
Stage: [1](https://tc39.github.io/process-document/) <br>
Spec: [ecmarkup output][draft spec], [source][]

## Historical Note

After feedback from TC39, this proposal was merged from two stage-0 proposals: [evalable](https://github.com/mikesamuel/evalable) and [host-passthru](https://github.com/mikesamuel/proposal-hostensurecancompilestrings-passthru).

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

*  [Content-Security-Policy](https://csp.withgoogle.com/docs/index.html)
*  `node --disallow_code_generation_from_strings`

which turn off `eval` globally.

As JavaScript programs get larger, the chance that no part of it needs `eval` or `Function()`
to operate gets smaller.

----

It is difficult in JavaScript for a code reviewer to determine that
code never uses these operators.  For example, the below can when `x` is constructor.

```js
({})[x][x](y)()

// ({})[x]       === Object
// ({})[x][x]    === Function
// ({})[x][x](y)  ~  Function(y)
```

So code review and developer training are unlikely to prevent abuse of these
operators.

----

This aims to solve the problem by providing more context to host environments so that they
can make finer-grained trust decisions.

## Problem: %eval% does not accept objects in lieu of strings for code

Currently, `typeof x === 'string' || eval(x) === x`.

For example:

```js
console.log(eval(123) === 123);

const array = [ 'alert(1)' ];
console.log(eval(array) === array);  // Does not alert

const touchy = { toString() { throw new Error(); } };
console.log(eval(touchy) === touchy);  // Does not throw.
```

This follows from step 2 of the definition of [PerformEval](https://tc39.github.io/ecma262/#sec-performeval):

> ### Runtime Semantics: PerformEval ( *x*, *evalRealm*, *strictCaller*, *direct* )
> <p>The abstract operation PerformEval with arguments *x*, *evalRealm*, *strictCaller*, and *direct* performs the following steps:</p>
>
> 1. Assert: If *direct* is **false**, then *strictCaller* is also **false**.
> 1. If Type(*x*) **is not String**, return *x*.

### Backwards compatibility constraints

To avoid breaking the web, `eval(x)` should do nothing different when x is a
non-string value that existing applications might create.

### Solution

Define a spec abstraction, *IsCodeLike*, that allows some object values through
but without changing the semantics of pre-existing programs.

There are two broad options outlined in the [draft spec][].

All the approaches below do the following

*  Define IsCodeLike(*x*), a spec abstraction that returns true for strings
   so is backwards compatible, and returns true for some other values.
   (See details of specific proposals below.)
*  Tweak PerformEval, which is called by both direct and indirect `eval`, to
   use IsCodeLike(*x*) instead of the existing (Type(*x*) is String).
*  Use ToString(*x*) for the code to parse.

#### Internal slot

*  IsCodeLike(*x*) additionally returns true when *x* is an object that
   has an internal slot \[\[CodeLike\]\].

##### Pros

*  IsCodeLike has no observable side effect for values so is backwards
   compatible when a program produces no code like values.

#### Well-known Symbol

From [ecma262 issue #938](https://github.com/tc39/ecma262/issues/938#issuecomment-457352474):

> Include the string to be compiled in the call to
> `HostEnsureCanCompileStrings` Perhaps we could tweak the `eval` algo
> so that if the input is a string or has a particular symbol property
> then it is stringified to avoid breaking code that depends on the
> current behavior where `eval` is identity except for strings.

*  Define a well-known symbol, *\@\@evalable*.
*  IsCodeLike(*x*) additionally returns true when *x* is an object and
   `x[Symbol.evalable]` is truthy.
   Note: using a new symbol makes IsCodeLike backwards comparible for programs
   that do not use `Symbol.evalable`.

##### Pros

*  Code-like values are proxyable.  (TODO: is this a feature?)
*  User code can define code-like values.  (TODO: is this a feature?)
*  Can be tested in test262 via user code.



## Problem: Host callout does not receive type information

Currently the information available to decide whether to allow compilation of
a string is a pair of realms.

> HostEnsureCanCompileStrings( *callerRealm*, *calleeRealm* )
>
> HostEnsureCanCompileStrings is an implementation-defined abstract
> operation that allows host environments to block certain ECMAScript
> functions which allow developers to compile strings into ECMAScript
> code.

There is a web reality issue here.  [CSP3's issue 8](https://www.w3.org/TR/CSP3/#issues-index)
notes:

> <code>HostEnsureCanCompileStrings()</code> does not include the
> string which is going to be compiled as a parameter. We’ll also need
> to update HTML to pipe that value through to CSP.
> &lt;https://github.com/tc39/ecma262/issues/938&gt;

### Solution

Pass extra parameters to the host callout including:

*  The grammar production used to parse the source text.  E.g. *ScriptBody* for
   <tt>eval(x)</tt> and *FunctionBody* for <tt>new Function(x)</tt>
*  The argument list.

## Problem: host callout cannot adjust values

Trusted Types defines a [default policy][] to make it easier to
migrate applications that pass around strings.

This policy is invoked when a string value reaches a sink, so any default policy's
[`createScript` callback][createScript callback] is invoked on `eval(myString)`.

If the callback throws an error, then `eval` should be blocked.  But
if the callback returns a value, *result*, then ToString(*result*)
should be used as the source text to parse.

Being able to adjust the code that runs provides the maximum
flexibility when dealing with a thorny legacy module that might
otherwise prevent the entire application from running with XSS
protections enabled.

### Solution

This proposal adjusts `eval` and `new Function` to expect return
values from the host callout and to use those in place of the inputs.

## Tests

Related tests at

*  https://github.com/tc39/test262/blob/master/test/language/eval-code/direct/non-string-object.js
*  https://github.com/tc39/test262/blob/master/test/language/eval-code/indirect/non-string-object.js

but testing these require affecting the behavior of host callouts so will probably need to
be specified as [web-platform-tests](https://github.com/web-platform-tests/wpt).




[core-js-example]: https://github.com/zloirock/core-js/blob/2a005abe68520248d4431cab70d86e40b55d6e98/packages/core-js/internals/global.js#L5
[TT]: https://wicg.github.io/trusted-types/dist/spec/
[Trusted Types]: https://wicg.github.io/trusted-types
[TrustedScript]: https://wicg.github.io/trusted-types/dist/spec/#typedef-trustedscript
[source]: https://github.com/mikesamuel/proposal-hostensurecancompilestrings-passthru/blob/master/spec.emu
[default policy]: https://wicg.github.io/trusted-types/dist/spec/#default-policy-hdr
[createScript callback]: https://wicg.github.io/trusted-types/dist/spec/#callbackdef-createscriptcallback
[draft spec]: https://mikesamuel.github.io/dynamic-code-brand-checks/
