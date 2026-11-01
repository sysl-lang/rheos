# sysl issues encountered by Rheos

This log distinguishes confirmed compiler defects, language or library gaps, and tooling irritations.
Physics mistakes, Metal shader mistakes and sandbox restrictions are not compiler defects.

## Confirmed compiler defects

None confirmed so far (sysl 0.8.11).

## Tooling: a skipped GPU check is reported as a passing test

Observed with sysl 0.8.11 on macOS in the development sandbox.

The test runner hides stdout for successful tests. Rheos's GPU test prints a skip message and returns
when SDL cannot access a display, but the runner reports that test as `ok` and counts it as passed.
The suite summary therefore does not distinguish a shader check that ran from one that did not.

Reproducer from `examples/box` in a session without display access:

```sh
SYSL_NO_CACHE=1 sysl test . --file gpu_tests --keep-binary .sysl/gpu-check
.sysl/gpu-check/box-tests 'demo$shader_pipeline'
```

The runner reports `ok`; running the retained binary directly reveals:

```text
SKIPPED GPU verification: The video driver did not add any displays
```

This is a reporting/API limitation, not incorrect compiler execution: the test returns normally.
A runtime skip result recognised by the runner would let the summary distinguish pass, fail and skip.
Rheos must report the GPU check as unverified when that branch is taken.

## UI ergonomics: the basic Text leaf clips long diagnostics

Observed in syslUI 0.1.16: `Text.measure` returns the natural text size without using the offered
width, and `Text.paint` clips the run to its rectangle. Rheos's longer collision errors therefore
disappeared beyond the side panel. This is a toolkit behaviour, not a compiler bug. The demo now
uses a small diagnostic view that wraps words using the canvas's real font measurements, and prints
the complete error and reproduction seed to the terminal. A wrapping text leaf in syslUI would
avoid each application needing to handle this case.

## Excluded from the compiler issue count

- The initial shader used `quad`, a reserved Metal type name. Renaming it fixed the shader.
- SDL cannot access the desktop display from this sandbox, and the standalone Metal toolchain is
  unavailable here. These are environment limitations.
- `Matrix` was found in `sh.sysl.linalg`, rather than the standard library. This is an API location,
  not a demonstrated language gap.

For each future confirmed defect, record the compiler version, expected behaviour and relevant
reference page, actual diagnostic or result, and the smallest reproducible program and command.
