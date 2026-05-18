# Graduation System — Training Documentation

> CIAST 2026 training course materials for building a Laravel 13 graduation registration system from scratch.

## What's in here

| Document | Audience | Purpose |
|---|---|---|
| [build-spec.md](./build-spec.md) | Claude Code (AI builder) | The original build brief — a sequenced spec that instructs Claude Code to scaffold the full reference app in one pass. Hard rules + 12 phases + verification commands. |
| [product-requirements.md](./product-requirements.md) | Product / QA / Stakeholders | The PRD — derived from the 27 reference commits. Functional + non-functional requirements, authorization matrix, user journeys, acceptance criteria. |
| [material-guide/](./material-guide/) | Training participants | Step-by-step build guide — one markdown file per commit (27 steps + prerequisites + index). Each step has Why / What / Steps / Expected output / Common pitfalls. |

## Relationship to the reference repository

The reference application built from these docs lives separately at `graduation-system/` (its own git repo). After each material-guide step, participants can `git checkout <sha>` of that repo to compare their work against the trainer's commit at the same step.

## How to use

- **First-time participant?** Start at [material-guide/README.md](./material-guide/README.md) → [00 — Prerequisites](./material-guide/00-prerequisites.md).
- **Designing or pitching the system?** Read [product-requirements.md](./product-requirements.md).
- **Curious about the original AI-driven build process?** Read [build-spec.md](./build-spec.md).

## License

Training materials. Use for your own learning and teaching.
