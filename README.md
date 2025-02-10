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
- Others not disclosed here. 

The reason this particular issue is fingered for causing security vulnerabities is
that it adds many paths for user code execution which otherwise don't exist, and
is not always obviously a possbility.

Of particular danger is where specification authors think of newborn objects of known
types as known quantities, only to call `Promise.resolve` on them. At this point 
when they are provided a JS wrapper the JS wrapper typically has Object as their 
prototype, making them vulnerable to thenables. 

Beyond security, this also just injects complexity. There are test cases in WPT that
exist purely to work out the expected behaviour [for someone breaking `then`](https://searchfox.org/mozilla-central/source/testing/web-platform/tests/fetch/api/response/response-stream-with-broken-then.any.js#4-24)

## How do I propose we fix this?

As a Stage 0 proposal, I'm convinced we should try to improve the situation here,
but am less convinced of a particular solution.

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

## HTML Integration

We would further want to recommend that builtin prototypes in embeddings (particularly HTML) 
also get `[[InternalProto]]` internal slot. 

## Is this a bulletproof fix? 

No. Of the previously described security bugs this mitigation would fix 

- [Out of bound access in ReadableStream::Close](https://issues.chromium.org/issues/40051366)
- [CVE-2024-43357](https://github.com/tc39/ecma262/security/advisories/GHSA-g38c-wh3c-5h9r) on the specification.
- Some of the undisclosed bugs. 

It would not have fixed 

- [CVE-2024-9086](https://www.welivesecurity.com/en/eset-research/romcom-exploits-firefox-and-windows-zero-days-in-the-wild/),
   as in this bug it would be sufficient to add the "then" property to an instance already escaped to script

I am not sure about the Chrome UAF in animation.

## Compatibility

I have telemetry probes that suggest caution here. From Firefox Nightly: 

1. 2.3% of pages resolve a thenable object.
2. Of those 2.3%, 2.0% find the thenable function on a prototype
3. Of those, 0.12% of pages load the "thenable" function off the prototype of a "Standard" class,
   which is basically anything in [this list](https://searchfox.org/mozilla-central/source/js/public/ProtoKey.h#68-169)
   except Promise is explicitly carved out.

Note the 0.12% is an undercount -- we did not gather data for HTML builtin prototypes. 

## Performance:

This introduces a new form of `Get` which will require implementation in a
performant manner given that promise resolution is a performance sensitive
operation.

I believe that this ultimately should be more optimizatiable than a straight up get,
as all our regular optimizations will apply, and less prototypes will need traversal.

## Potential Alternatives:

It's possible we could choose to try to fix this for only WebIDL, by specifying the something about how
resolving a WebIDL dictionary works. Personally I would start at the language tho.

## Prior Art & Related Work

- [`Symbol.thenable`](https://github.com/tc39/proposal-symbol-thenable) "Withdrawn;
  changing thenability on Module Namespace objects is not web compatible, and
  allowing non-Promise use of "then" is not worth slowing down all Promise
  operations"
- [Proposal Stabilize](https://github.com/Agoric/proposal-stabilize/) is trying to
  provide generalizable machineries for invariants -- this could be more of
  an invariant we could provide to user code as well. 
