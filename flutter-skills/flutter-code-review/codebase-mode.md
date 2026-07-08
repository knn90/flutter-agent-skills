# Codebase mode — how the stages adapt when there is no diff

Scope is the whole repo, not a change. Adapt each stage:

- **Phase 0:** enumerate source files (the always-exclude list still applies); group by feature/package
  folder. Every file is `modified`-bucket — there is no adjacent quarantine. Never trivial.
  Banner: `Scope: codebase · packages: N · files: N · HIGH-RIGOR: yes/no` (HIGH-RIGOR = any
  `high_rigor_domains` code exists in the repo).
- **Stage 1 (spec):** skip — there is no change intent. Mark "not assessed".
- **Stage 2:** run the lenses per package (one parallel agent per feature/package folder). Specialist
  routing (2.0) keys off file content instead of diff hunks.
- **Stage 3 (red-team):** replace "stated intent" with each package's responsibility. Lenses 2–4
  (security, perf/reliability, contracts/coverage) apply as-is; lens 1 becomes invariant-hunting —
  state-transition holes, contract drift *between* packages.
- **Synthesis / severity / report:** unchanged, but cap the report at the top findings per severity —
  precision over recall matters even more at codebase scale.
