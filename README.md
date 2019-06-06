# TC39 Proposal evalable (Stage [0](https://tc39.github.io/process-document/))

Relax the requirement that the argument to eval be a string in a non-breaking way.

## Background

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

## Goal

Encourage *source to sink* checking of arguments to `eval` to reduce the risk of XSS.

Source to sink checking means we do something at the source of a value so that, when the value
reaches a "sensitive sink" (like `eval`), we can double-check that it is safe.

The [Trusted Types][] proposal brings source-to-sink checks to browser built-ins
like `.innerHTML`.

> 1. Introduce a number of types that correspond to the XSS sinks we wish to protect.
>
> **TrustedScript**: This type would be used to represent a trusted JavaScript code block i.e.
> something that is trusted by the author to be executed by ... passing to an `eval` function family.
>
> 2. Enumerate all the XSS sinks we wish to protect. ...
>
> 3. Introduce a mechanism for disabling the raw string version of each of the sinks identified. ...
>
> ----
>
> As valid trusted type objects must originate from a policy, those policies alone form the trusted
> codebase in regards to DOM XSS.

So by double-checking at *sinks* like `eval` we can ensure that 
only carefully reviewed code that is the *source* of *TrustedScript*
values will load alongside trusted code.

This won't work today because, as explained above, `eval(myTrustedScript)` does not actually
evaluated the textual content of *TrustedScript* values.

## The Larger Context

This proposal is useful in the context of other planned changes:

1.  The [Trusted Types][] proposal as explained.
1.  A proposed [tweak to the HostEnsureCanCompileStrings][host callout proposal]
    that would allow browsers to take more context into account when deciding
    whether to allow a particular call to `eval`.

## Backwards compatibility constraints

To avoid breaking the web, `eval(x)` should do nothing different when x is a
non-string value that existing applications might create.

This should be the case regardless of whether HostEnsureCanCompileStrings says
yes or no as demonstrated in node:

```sh
$ npx node@10 --disallow_code_generation_from_strings -e 'console.log(eval(() => {}))'
[Function]

$ npx node@10 --disallow_code_generation_from_strings -e 'console.log(eval("() => {}"))'
[eval]:1
console.log(eval("() => {}"))
        ^

EvalError: Code generation from strings disallowed for this context
```

When `eval` is passed a non-string value, even though the Host callback rejects all
attempts to `eval`, it doesn't throw when given the value `() => {}`, but does when
given the string `"() => {}"`.

## Possible solutions

You can browse the [ecmarkup output](https://mikesamuel.github.io/evalable/)
or browse the [source](https://github.com/mikesamuel/evalable/blob/master/spec.emu).

All the approaches below do the following

*  Define IsCodeLike(*x*), a spec abstraction that returns true for strings
   so is backwards compatible, and returns true for some other values.
   (See details of specific proposals below.)
*  Tweak PerformEval, which is called by both direct and indirect `eval`, to
   use IsCodeLike(*x*) instead of the existing (Type(*x*) is String).
*  Use ToString(*x*) for the code to parse.

### Internal slot

*  IsCodeLike(*x*) additionally returns true when *x* is an object that
   has an internal slot \[\[CodeLike\]\].

#### Pros

*  IsCodeLike has no observable side effect for values so is backwards
   compatible when a program produces no code like values.

### Well-known Symbol

From [ecma262 issue #938](https://github.com/tc39/ecma262/issues/938#issuecomment-457352474):

> # Include the string to be compiled in the call to
> `HostEnsureCanCompileStrings` Perhaps we could tweak the `eval` algo
> so that if the input is a string or has a particular symbol property
> then it is stringified to avoid breaking code that depends on the
> current behavior where `eval` is identity except for strings.

*  Define a well-known symbol, *\@\@evalable*.
*  IsCodeLike(*x*) additionally returns true when *x* is an object and
   `x[Symbol.evalable]` is truthy.
   Note: using a new symbol makes IsCodeLike backwards comparible for programs
   that do not use `Symbol.evalable`.

#### Pros

*  Code-like values are proxyable.  (TODO: is this a feature?)
*  User code can define code-like values.  (TODO: is this a feature?)
*  Can be tested in test262 via user code.


## Testing

TODO: Updates to

*  https://github.com/tc39/test262/blob/master/test/language/eval-code/direct/non-string-object.js
*  https://github.com/tc39/test262/blob/master/test/language/eval-code/indirect/non-string-object.js


## Related

https://github.com/tc39/ecma262/issues/1495


[Trusted Types]: (https://wicg.github.io/trusted-types)
[host callout proposal]: https://github.com/mikesamuel/proposal-hostensurecancompilestrings-passthru


# Proposal: HostEnsureCanCompileStrings Passthru

## Status

Champion(s): mikesamuel <br>
Author(s): mikesamuel <br>
Stage: 0

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


## Use cases

The [Trusted Types proposal][TT] aims to allow JavaScript development to scale securely.

It makes it easy to separate:

*  the decisions to trust a chunk of code to load.
*  the check that an input to a sensitive operator like `eval` is trustworthy.

This allows trust decisions to be made where the maximum context is
available and allows these decisions to be concentrated in small
amounts of thoroughly reviewed code

The checks can also be moved into host code so that they reliably happen before
irrevocable actions like code loading complete.

Specifically, Trusted Types would like to require that the code portions of the
inputs to %Function% and %eval% are [*TrustedScript*][TrustedScript].


## What are code portions

`eval(x)` treats `x` as code.

`new Function(x)` also treates `x` as code.

The parameter list portion of a function can also be code since *FormalParameterList* elements may contain arbitrary default value expressions.

For example, the following function has a blank body, but still has a side effect.

```js
(new Function('a = alert(1)', ''))()
```


## Host callout should be a two-way street

Trusted Types defines a [default policy][] which
is invoked when a string reaches a sink to make it easier to migrate applications that pass around strings.

This policy is invoked when a string value reaches a sink, so any default policy's
[`createScript` callback][createScript callback] is invoked on `eval(myString)`.

If the callback throws an error, then `eval` should be blocked.
But if the callback returns a value, *result*, then ToString(*result*) should be used as the source text to parse.

Being able to adjust the code that runs provides the maximum flexibility when dealing with a thorny legacy module
that might otherwise prevent the entire application from running with XSS protections enabled.

To enable that, this proposal adjusts `eval` and `new Function` to expect return values from the host callout and
to use those in place of the inputs.


## Possible spec language

You can browse the [ecmarkup output][] or browse the [source][].


[core-js-example]: https://github.com/zloirock/core-js/blob/2a005abe68520248d4431cab70d86e40b55d6e98/packages/core-js/internals/global.js#L5
[TT]: https://wicg.github.io/trusted-types/dist/spec/
[TrustedScript]: https://wicg.github.io/trusted-types/dist/spec/#typedef-trustedscript
[ecmarkup output]: https://mikesamuel.github.io/proposal-hostensurecancompilestrings-passthru/
[source]: https://github.com/mikesamuel/proposal-hostensurecancompilestrings-passthru/blob/master/spec.emu
[default policy]: https://wicg.github.io/trusted-types/dist/spec/#default-policy-hdr
[createScript callback]: https://wicg.github.io/trusted-types/dist/spec/#callbackdef-createscriptcallback
