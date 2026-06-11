# Standard 13: Build for Extractability from Day One

> Agents can't reason about future architecture. They optimize for what works now, in the current session, with the current context. If module boundaries aren't enforced through interfaces and dependency injection from day one, agents will create direct couplings that make future extraction expensive. By the time you need microservices, the codebase is too entangled to extract — not because the agents were wrong, but because nothing prevented them from taking shortcuts that felt locally correct.

> **Standards** (must follow):
> - Interface-based dependency injection from day one — every service implements a Protocol
> - Cross-module dependencies wired at a single composition root
> - Modules use opaque identifiers, not direct ORM foreign keys across module boundaries
>
> **Guidelines** (recommended):
> - Specific DI framework or composition root pattern
> - How to structure the composition root for testability
> - When to actually extract a module to a service (scale signals)

*Extractability* is the property that any module can later be lifted out into its own service without rewriting the code that calls it. You design for it from day one so that growth never forces a rewrite.

The biggest objection to careful early architecture is "we're too small for this." In a narrow sense, that's true. A small team doesn't need microservices.

But the decisions you make in a modular monolith today are the same decisions you'd make in a microservices architecture. They're not premature — they're early.

Return to the commission rate example. If the platform eventually grows large enough that transaction processing needs its own service:

```python
# Today — local module call
self.org_service: OrganizationServiceInterface = OrganizationService(db)

# Tomorrow — remote service call
self.org_service: OrganizationServiceInterface = OrganizationHTTPClient(
    base_url=settings.ORG_SERVICE_URL
)
```

The composition root is the only place the wiring changes — every module that calls `self.org_service.get_commission_rate(org_id)` keeps compiling against the same interface, not a concrete class. That is what makes extraction *tractable* rather than a rewrite.

What the shared interface does **not** do for free: crossing a network boundary adds concerns an in-process call never had — timeouts, retries, and partial failures the caller must now handle; serialization of `Decimal`, `UUID`, and `datetime` over the wire; the loss of a single database transaction spanning both modules; latency on every call; and authentication on the new boundary. The Protocol keeps the *call sites* stable; you still have to do the distributed-systems work behind the seam.

This works because:
1. Every service implements a `Protocol` interface, not a class
2. Cross-module dependencies are injected at a single location (the composition root)
3. Modules use opaque string identifiers, not database foreign keys — no ORM coupling to unpick

A codebase built this way is *extraction-ready* from day one. Not because microservices were the target, but because module isolation and interface contracts are the same idea at different scales — and when the day comes, you are hardening one existing seam, not carving a new boundary through entangled code.
