# HTSlib Upgrade Plan: 1.3 → 1.21

## Status: Deferred
Last updated: 2026-04-13

## Reason for Deferral

- HTSlib 1.21 has limited native Windows/MSVC support (5 files with `_WIN32` guards)
- STAR uses internal HTSlib APIs that may have changed between 1.3 and 1.21
- The current HTSlib 1.3 with Windows patches works correctly (byte-identical output verified)
- The upgrade requires careful API migration, new dependency management (liblzma, libbz2, libdeflate), and full regression testing
- Risk/reward ratio is unfavorable: current htslib works, upgrade could introduce regressions

## When to Revisit

- If a security CVE is found in HTSlib 1.3
- If new BAM/CRAM format features are needed
- If the project moves to an active maintenance cadence with CI regression testing

## Current Windows Patches (for reference)

See git diff of `source/htslib/` for the full patch set.
Key files: `win32_compat.h`, `win32_stubs/*`, `bgzf.c`, `hfile.c`, `cram/thread_pool.h`, `htslib/hts_defs.h`.
