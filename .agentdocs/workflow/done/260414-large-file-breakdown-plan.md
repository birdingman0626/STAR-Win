# Large File Breakdown Plan

## Requirements Summary

Break down owned code files larger than 800 lines into smaller, responsibility-focused units without changing STAR behavior.

Current first-party inventory:
- `source/Parameters.cpp` is the only owned source file above the 800-line threshold at 1147 lines.

Non-goals:
- Do not split vendored code under `source/htslib/`, `source/opal/`, or `source/SimpleGoodTuring/`.
- Do not force churn in files that are below 800 lines unless needed to support the `Parameters.cpp` split.

## Acceptance Criteria

- No owned source file in the targeted scope exceeds 800 lines after the refactor.
- `source/Parameters.cpp` is split by responsibility, not by arbitrary line count.
- Public behavior is unchanged: parameter parsing, validation, logging, and run-mode setup behave identically.
- Existing tests continue to pass, and new regression coverage is added around moved logic where practical.
- Build graph remains simple: no new third-party dependencies and no duplicated parameter-registration logic.

## Target File Analysis

`source/Parameters.cpp` currently mixes several distinct responsibilities:
- parameter registration in `Parameters::Parameters()` at `source/Parameters.cpp:19`
- command-line and file parsing in `Parameters::inputParameters(...)` starting at `source/Parameters.cpp:314`
- log/output bootstrap around `source/Parameters.cpp:386`
- temp-dir and runtime setup around `source/Parameters.cpp:494`
- output-format and BAM/SAM setup around `source/Parameters.cpp:529-705`
- parameter validation and mode-specific checks through the remainder of the file

This is the correct first target because it is both above the threshold and already serves as a central coordinator for unrelated concerns.

## Implementation Steps

### Phase 1: Lock behavior

- Add focused tests for parameter parsing and validation paths before moving logic.
- Prefer coverage for:
  - `outWigType` / `outWigStrand` / `outWigNorm`
  - `outSAMtype` and sorted-BAM requirements
  - `runMode` dispatch edge cases
  - two-pass / genome-load compatibility

Suggested test touch points:
- `test/CMakeLists.txt`
- new `test/unit/test_Parameters*.cpp`

### Phase 2: Split parameter registration

Extract the constructor registration blocks from `source/Parameters.cpp:19-309` into narrow helpers, for example:
- `Parameters_register_core.cpp`
- `Parameters_register_output.cpp`
- `Parameters_register_alignment.cpp`
- `Parameters_register_solo.cpp`

Rules:
- Keep helper ownership on the `Parameters` class.
- Do not change parameter names, defaults, or registration order unless tests prove it is safe.

### Phase 3: Split input/bootstrap flow

Extract `Parameters::inputParameters(...)` into internal helpers grouped by lifecycle:
- command-line normalization and default loading
- log/output stream bootstrap
- parameter-file loading and final effective command reconstruction
- temp-dir and runtime setup

Suggested file names:
- `Parameters_inputBootstrap.cpp`
- `Parameters_inputSources.cpp`
- `Parameters_runtimeSetup.cpp`

### Phase 4: Split validation/setup domains

Move domain-specific validation blocks out of the main file into separate compilation units:
- output wiggle and duplicate-removal validation
- SAM/BAM output setup
- filtering and SJ validation
- two-pass and read-file initialization
- WASP / quant / SAM-attribute validation

Suggested file names:
- `Parameters_validateOutput.cpp`
- `Parameters_validateAlignment.cpp`
- `Parameters_validateQuant.cpp`

### Phase 5: Header and helper cleanup

- Keep `Parameters.h` stable if possible.
- If new private helper declarations are needed, add only the minimum required surface.
- Prefer free functions in an anonymous namespace or private member helpers over expanding the public API.

### Phase 6: Threshold guardrail

After `Parameters.cpp` is reduced below 800 lines, add a lightweight repository rule:
- owned files under top-level `source/` should stay below 800 lines
- exceptions must be explicit and justified in a workflow note or PR rationale

This should be documented in contributor guidance rather than enforced with a hard failing build at first.

## Risks And Mitigations

- Risk: registration-order drift changes behavior.
  - Mitigation: preserve registration order exactly and add regression checks around effective command output.
- Risk: moved validation logic changes initialization sequencing.
  - Mitigation: split by existing execution order, not by category alone.
- Risk: helper sprawl makes the code harder to follow.
  - Mitigation: create a small number of lifecycle- and domain-based files, not dozens of tiny fragments.

## Verification Steps

- Run `ctest` for the existing test suite plus new `Parameters` tests.
- Build `STAR` with the current CMake path after each phase.
- Run a smoke command for:
  - `--runMode alignReads`
  - `--runMode genomeGenerate`
  - `--runMode soloCellFiltering`
- Compare key logs or error messages for at least one invalid-parameter case and one valid-run case.

## Follow-up

After `Parameters.cpp`, re-run the file-size inventory. Current next-largest owned files are below the threshold, led by:
- `source/ParametersSolo.cpp` at 690 lines
- `source/SoloFeature_collapseUMIall.cpp` at 626 lines
- `source/ReadAlign_alignBAM.cpp` at 543 lines

Those should be monitored, not proactively split yet.
