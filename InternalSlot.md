# The "Internal Slot" Solution

The following contains an archived set of content (pre-February 2026) describing the internal slot solution.

The goal of this solution was to be relatively general, however, the proposal has evolved to be slightly less general. Nevertheless, I would like the text to be readable without having to rummage in git history in case anyone is wondering about what was being talked about in the historical presentations.

--- 

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