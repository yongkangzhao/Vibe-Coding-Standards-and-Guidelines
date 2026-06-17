# Frontend: Interface-First (TypeScript / React / vitest + Testing Library / MSW)

The frontend analog of Spec -> Interface -> Tests -> Implementation. The frontend is a thin
presentation layer: contracts here are typed component props/events and a typed API-client
interface. Business logic (pricing, the commission rate itself) lives in the backend — the
client only *calls* it. Write the README, the types, and the failing tests before the component.

## 1. Spec — `features/org-settings/README.md`

```markdown
# Org Settings Feature

## Role
Renders and edits an organization's settings. Presentation only.

## Does NOT own
- Computing or validating the commission rate → backend `organization` service.
  We display and submit it; we do not decide what is valid beyond surfacing API errors.
- Auth / session → `auth` feature.
- Persisting state → server is the source of truth; we hold view state only.

## Public interface
- Component contract: `OrgSettingsForm` props/events below.
- Data contract: `OrgApiClient` in `api/org-client.ts`.

## Dependencies
- `OrgApiClient` (typed). If an endpoint is missing, file a GitHub issue — do not
  reimplement the rule client-side.

## Test command
    vitest run features/org-settings
```

## 2. Interface — typed contracts (written before the component)

```typescript
// features/org-settings/types.ts — the component contract
export interface Organization {
  id: string;
  name: string;
  commissionRate: string; // decimal-as-string; backend owns the value
}

export interface OrgSettingsFormProps {
  org: Organization;
  onSubmit: (next: { commissionRate: string }) => void; // event contract
  isSaving: boolean;
}
```

```typescript
// features/org-settings/api/org-client.ts — the data contract, defined before impl
export interface OrgApiClient {
  // The single typed entry point to the backend's authoritative rate.
  getCommissionRate(orgId: string): Promise<string>;
  setCommissionRate(orgId: string, rate: string): Promise<void>;
}
```

Components and hooks depend on the `OrgApiClient` interface, never on `fetch` directly — so an
agent building another feature can build against the same contract in parallel.

## 3. Tests — against the interface, MUST fail first (red phase)

MSW stands in for the backend; the component stub renders nothing, so these fail until step 4.
Confirm the red state. See `tests-as-ground-truth` and `superpowers:test-driven-development`.

```typescript
// features/org-settings/OrgSettingsForm.tsx — stub so tests import and FAIL
export function OrgSettingsForm(_: OrgSettingsFormProps) {
  throw new Error('NotImplemented');
}
```

```typescript
// features/org-settings/OrgSettingsForm.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { describe, it, expect, vi } from 'vitest';
import { OrgSettingsForm } from './OrgSettingsForm';

const org = { id: 'o1', name: 'Acme', commissionRate: '0.15' };

describe('OrgSettingsForm', () => {
  it('shows the current commission rate from props', () => {
    render(<OrgSettingsForm org={org} isSaving={false} onSubmit={vi.fn()} />);
    expect(screen.getByLabelText(/commission rate/i)).toHaveValue('0.15');
  });

  it('emits onSubmit with the edited rate (no client-side validation logic)', () => {
    const onSubmit = vi.fn();
    render(<OrgSettingsForm org={org} isSaving={false} onSubmit={onSubmit} />);
    fireEvent.change(screen.getByLabelText(/commission rate/i), { target: { value: '0.2' } });
    fireEvent.click(screen.getByRole('button', { name: /save/i }));
    expect(onSubmit).toHaveBeenCalledWith({ commissionRate: '0.2' });
  });
});
```

```bash
vitest run features/org-settings   # expect: failures (NotImplemented) before step 4
```

## 4. Implementation — last

Fill in `OrgSettingsForm` and the concrete `OrgApiClient` (fetch + zod-parsed responses) until
the tests pass. Keep it presentation-only: if a validation or pricing rule is missing on the
backend, file a GitHub issue rather than adding the logic here. New props or client methods go
back to step 1 (update the README and types first).
