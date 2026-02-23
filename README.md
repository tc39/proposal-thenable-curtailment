# Curtailing the power of "Thenables"

**Champion**: Matthew Gaudet (Mozilla)

**Stage**: 1 (As of February Plenary) 

## Introduction & Problem:

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
prototype chain. Including builtin prototypes and `Object.prototype`. This is
particularly dangerous when working with types where 'thenable' dispatch was
unexpected.

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

I'd like to propose we add a (maybe-)delaying resolve operation. As per ususal naming will be a substantial
challenging, but a general principle here would be, it is functionally identical to
[`PromiseResolve`](https://tc39.es/ecma262/#sec-promise-resolve) except there is a pre-step where
we check for the conditions under which we could run user-code. If we cannot run any user code, we simply
tail-call into `PromiseResolve`. If we *could* run user-code, we instead enqueue a new job whose
responsibility is to call into `PromiseResolve`, while also putting the promise into a 'parked' state
such that any future resolutions are ignored (Thank you very much to Mark Miller for catching this
requirement in TG3 review discussion).

The next step is to decide how to consume this. There is interest from the Mozilla DOM to explore
using this to replace [the steps for resolving a promise in WebIDL](https://webidl.spec.whatwg.org/#resolve)
and more generally powering all the promise resolution code in Mozilla's DOM. This would help make
C++ code safer by making promise resolution into an operation that never runs script, which
simplifies the reasoning required when implementing code.

Another topic of discussion would be: If we builds this capability, do we expose it to userland code
and if so, under what name.

## Is this a bulletproof fix?

No. The impact of this will of course depend entirely on the scope of adoption.s

Of the previously described security bugs this mitigation would fix

- [Out of bound access in ReadableStream::Close](https://issues.chromium.org/issues/40051366)
- Some of the undisclosed bugs.
- [CVE-2024-9086](https://www.welivesecurity.com/en/eset-research/romcom-exploits-firefox-and-windows-zero-days-in-the-wild/),
   as in this bug it would be sufficient to add the "then" property to an instance already escaped to script
- I suspect the Chrome Animation UAF but am not 100% sure

It would not however fix

- [CVE-2024-43357](https://github.com/tc39/ecma262/security/advisories/GHSA-g38c-wh3c-5h9r) on the specification, as
  it's very unlikely we would adopt this new operation on that path.

## Compatibility

This could change the order in which microtasks get resolved when 'thenables' are involved. The hope
is that the majority of code dealing with promises is already relatively robust to execution
order. However, it is certainly plausible this could cause a web compatibility problem.

## Prior Art & Related Work

- [`Symbol.thenable`](https://github.com/tc39/proposal-symbol-thenable) "Withdrawn;
  changing thenability on Module Namespace objects is not web compatible, and
  allowing non-Promise use of "then" is not worth slowing down all Promise
  operations"
- [Proposal Stabilize](https://github.com/Agoric/proposal-stabilize/) is trying to
  provide generalizable machineries for invariants -- this could be more of
  an invariant we could provide to user code as well.

## Proposal History
- [Presented at February 2025 Plenary](https://docs.google.com/presentation/d/1Sny2xC5ZvZPuaDw3TwqOM4mj7W6NZmR-6AMdpskBE-M/edit#slide=id.p) -- Achieved Stage 1. [(Notes)](https://github.com/tc39/notes/blob/main/meetings/2025-02/february-18.md#curtailing-the-power-of-thenables-for-stage-1)
- [Presented at July 2025 Plenary](https://docs.google.com/presentation/d/1_RCnI7dzyA1COgi_Ib7GkmHioV1nVSEMmZvy0bvJ8Kk/edit?slide=id.p#slide=id.p). ([Notes](https://github.com/tc39/notes/blob/main/meetings/2025-07/july-29.md#how-to-make-thenables-safer) & [Notes from Continuation](https://github.com/tc39/notes/blob/main/meetings/2025-07/july-30.md#continuation-how-to-make-thenables-safer))
