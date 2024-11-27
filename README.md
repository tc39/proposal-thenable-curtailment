# Curtailing the power of "Thenables"

Quoting MDN:

> The JavaScript ecosystem had made multiple Promise implementations long before it
> became part of the language. Despite being represented differently internally, at
> the minimum, all Promise-like objects implement the Thenable interface. A thenable
> implements the `.then()` method, which is called with two callbacks: one for when the
> promise is fulfilled, one for when it's rejected. Promises are thenables as well.
>
> To interoperate with the existing Promise implementations, the language allows using
> thenables in place of promises. For example, Promise.resolve will not only resolve
> promises, but also trace thenables.

The problem we would like to address is that `then` lookup follows the whole
prototype chain. Including builtin prototypes and `Object.prototype`.

## Why is this a problem?

The most concrete one is security vulnerabilities. We must be ever-vigilant about
this compatability supporting feature in all standard work, and throughout
the web-platform. Failure to do so has the consequence of possible exploitation:

- [CVE-2024-43357](https://github.com/tc39/ecma262/security/advisories/GHSA-g38c-wh3c-5h9r) on the specification.
- [Out of bound access in ReadableStream::Close](https://issues.chromium.org/issues/40051366)
- [CVE-2021-21206: Chrome Use-After-Free in Animations](https://googleprojectzero.github.io/0days-in-the-wild//0day-RCAs/2021/CVE-2021-21206.html)
- [CVE-2024-9086](https://www.welivesecurity.com/en/eset-research/romcom-exploits-firefox-and-windows-zero-days-in-the-wild/)

The reason this particular issue is fingered for causing security vulnerabities is
that it adds many paths for user code execution which otherwise don't exist, and
is not always obviously a possbility.

Beyond security, this also just injects complexity. There are test cases in WPT that
exist purely to work out the expected behaviour [for someone breaking `then`](https://searchfox.org/mozilla-central/source/testing/web-platform/tests/fetch/api/response/response-stream-with-broken-then.any.js#4-24)

## How do I propose we fix this?

As a Stage 0 proposal, I'm convinced we need to solve this, but am less convinced
of a particular solution.

During the remediation of the specification security issue, for defence in depth a
few compatible resolutions were proposed:

1. Normative: make Object.prototype exotically reject "then" properties. Change the
   `[[DefineOwnProperty]]` MOP operation on `Object.prototype` to silently noop when
   the property key is `"then"`.
2. Normative: make some promise resolve functions not respect thenables. Exclude
   certain internal resolution functions to not respect thenables.

I would propose a third solution:

- Specification defined prototypes gain a new internal slot `[[InternalProto]]`
- The lookup for the "then" property changes from `Get` to `GetNonInternal`, a new
  specification AO which walks the prototype chain, but stops as soon as it
  encounters a prototype with the `[[InternalProto]]` internal slot.

## Compatability

I suspect this will be almost entirely web compatible, but it would be worthwhile
to investigate before this hits Stage 4.

## HTML Integration

We would further want to recommend that builtin prototypes in embeddings (particularly HTML) 
also get `[[InternalProto]]` internal slot. 

## Performance:

This introduces a new form of `Get` which will require implementation in a
performant manner given that promise resolution is a performance sensitive
operation.

I believe that this ultimately should be more optimizatiable than a straight up get,
as all our regular optimizations will apply, and less prototypes will need traversal.

## Prior Art:

- [`Symbol.thenable`](https://github.com/tc39/proposal-symbol-thenable) "Withdrawn;
  changing thenability on Module Namespace objects is not web compatible, and
  allowing non-Promise use of "then" is not worth slowing down all Promise
  operations"
