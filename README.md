# mi-ideal

IDEAL for the Motorola 6809 / Hitachi HD63C09 (6309): a compact, embodied
interaction-loop target rather than a wholesale .NET port.

The near-term path is conservative: prove the Section 1 IDEAL trace in
6809-compatible RAM code, use M6x09-I for the first physical experiment, then
bring the same image to M6x09-II and only afterward evaluate 6309 native mode.

- [The case for 6309](CASE-FOR-6309.md) — rationale, architecture, platform
  roles, risks, and staged implementation plan.
- [Current status](CURRENT-STATUS.md) — existing assets, blockers, experiment
  sequence, and seed-milestone completion criteria.
