# Limitations and Risks

This playbook is intentionally pragmatic: it highlights where delivery risk appears and how to set expectations early.

## Preview Status

- WAC VM Conversion is Preview
- Feature behavior and support boundaries may evolve
- Production usage should be governed by pilot validation and customer acceptance

## Technical Constraints

- VMware source support is tied to vCenter 6.x and 7.x
- Active snapshots must be addressed before migration
- Guest OS support should be validated per workload
- Hyper-V capacity gaps can block migration sequencing

## Delivery Risks

- Underestimating migration wave complexity
- Incomplete cutover validation during change windows
- Lack of explicit rollback decision criteria

## Communication Risks

- Framing this as fully automated or zero-risk migration
- Omitting Preview caveats in customer-facing material
- Skipping phased rollout recommendations

## Recommended Mitigations

- Start with a pilot and publish objective exit criteria
- Keep wave sizes small and measurable
- Communicate limitations as part of presales qualification, not after project start
