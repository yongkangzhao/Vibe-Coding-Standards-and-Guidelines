# Standard 12: Build for Extractability from Day One

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

One line changes in the composition root. Nothing else changes. Every module that calls `self.org_service.get_commission_rate(org_id)` keeps working identically, because they were always calling an interface, not a concrete class.

This works because:
1. Every service implements a `Protocol` interface, not a class
2. Cross-module dependencies are injected at a single location (the composition root)
3. Modules use opaque string identifiers, not database foreign keys — no ORM coupling to unpick

A codebase built this way is microservice-ready from day one. Not because microservices were the target, but because module isolation and interface contracts are the same idea at different scales.
