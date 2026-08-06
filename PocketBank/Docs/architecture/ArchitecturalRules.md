# Architectural Rules

The following rules apply to all future development:

- Features may depend on Core.
- Features must not depend on other Features.
- Core must not depend on SwiftUI.
- Infrastructure implements protocols defined by Core.
- Views never instantiate concrete services.
- Dependencies are injected.
- Shared mutable state should be isolated using Actors where appropriate.
- Every new architectural decision that changes these rules requires a new ADR.
