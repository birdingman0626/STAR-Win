# Unit Test Strategy Plan

## Status: Pending
Last updated: 2026-04-13

## Recommendation

Yes, this project needs a substantially stronger automated test suite, but **not** a "unit test every file" effort.

STAR is a mixed codebase:
- some logic is pure and deterministic
- some paths are algorithmically complex but still testable with synthetic fixtures
- some behavior is only meaningful as end-to-end alignment output

The right approach is a **layered test strategy**:
- unit tests for pure helpers and bug-prone local logic
- focused component tests for STARsolo and parser/statistics code
- tiny integration tests with bundled fixtures
- existing smoke/release comparison kept as the final safety net

## Why this is worth doing

Current coverage is thin:
- CTest currently only checks `STAR --version`
- CI mostly proves that binaries build and start
- smoke/release validation is valuable, but too slow and too coarse to catch small regressions early

Without narrower tests, every bug fix or optimization must be validated through expensive end-to-end runs.

## Scope boundaries

Do **not** start with:
- full transcript-alignment correctness as unit tests
- giant fixture bundles
- test frameworks that require major dependency or build-system churn

Start with code that is:
- deterministic
- isolated
- numerically or combinatorially tricky
- previously involved in bugs or optimization work

## Phase 1: Establish Test Harness

- [ ] Add a lightweight C++ test target under CMake, driven by `ctest`
- [ ] Keep the initial harness dependency-free or minimal; avoid large external frameworks unless the maintenance cost is clearly justified
- [ ] Add CI steps that run unit/component tests on Linux first, then expand to other platforms

## Phase 2: First Wave of High-Value Tests

### Pure helper / utility tests

- [ ] `PackedArray`
- [ ] `binarySearch2`
- [ ] `blocksOverlap`
- [ ] Windows portability helpers in `wincompat.h` where behavior differs from POSIX assumptions

### STARsolo algorithm/component tests

- [ ] `SoloFeature_collapseUMI_Graph.cpp`
  - connected components
  - directional collapse edge cases
  - long-chain non-recursive behavior
- [ ] `SoloFeature_emptyDrops_CR.cpp`
  - p-value counting equivalence
  - streamed-checkpoint equivalence
  - no-candidate / low-count edge cases
- [ ] barcode / UMI parsing and filtering logic

### Regression tests for known bug-prone areas

- [ ] `winBin` / window bin bookkeeping
- [ ] Windows-specific read pipeline behavior
- [ ] file/path handling and temp-file fallbacks

## Phase 3: Tiny Integration Fixtures

- [ ] Add a very small bundled test dataset under a stable path such as `test/data/`
- [ ] Include:
  - tiny genome index or reduced synthetic genome
  - tiny FASTQ pair
  - tiny whitelist / Solo inputs where needed
- [ ] Assert a few stable outputs:
  - run completes
  - key summary metrics match
  - selected output files match known-good references

These tests should run in CI in minutes, not hours.

## Phase 4: CI and Gating

- [ ] Keep `version_check`, but stop treating it as meaningful test coverage
- [ ] Run fast unit/component tests on every PR
- [ ] Run tiny integration tests on every PR
- [ ] Keep 1M-read smoke tests for release branches or selected CI lanes
- [ ] Keep full release comparison as pre-release validation, not routine PR gating

## Phase 5: Optimization-Support Tests

- [ ] Add synthetic fixtures specifically for optimization-sensitive code:
  - UMI graph collapse
  - EmptyDrops simulation / lookup
  - `stitchWindowAligns` counters and branch behavior
- [ ] Require optimization PRs to add or update narrow tests before merging

## Success criteria

- Developers can verify many fixes without running full release validation
- STARsolo and utility regressions are caught before expensive end-to-end testing
- CI distinguishes build success from behavioral correctness
- optimization work becomes safer because hotspot logic is protected by focused tests

## Bottom line

The project needs a **comprehensive automated testing strategy**, but the implementation should be **incremental and layered**, not an attempt to unit-test the entire aligner core at once.
