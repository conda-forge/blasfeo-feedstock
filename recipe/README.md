# BLASFEO packaging on conda-forge

This feedstock supplies two packages:

- **`libblasfeo`**: the BLASFEO library, headers and CMake config files.
- **`libblasfeo-microarch-abi`**: an empty *marker* package. Its version is
  the BLASFEO version followed by the x86-64 microarch level of the matching
  `libblasfeo` build (for example `0.1.4.3.1` or `0.1.4.3.3`), or `0` on
  platforms without microarch variants.

## Why the marker exists

On x86-64, `libblasfeo` is built in one variant per
[microarch level](https://conda-forge.org/docs/maintainer/knowledge_base/#microarch),
each with a different BLASFEO target:

| Microarch level | BLASFEO target | Double panel size (`D_PS`) | Float panel size (`S_PS`) |
|---|---|---:|---:|
| 1 | `X64_INTEL_CORE` | 4 | 4 |
| 3 | `X64_INTEL_HASWELL` | 4 | 8 |
| 4 (currently disabled) | `X64_INTEL_SKYLAKE_X` | 8 | 16 |

The panel sizes are compiled into code that uses the installed headers, for
example through the `BLASFEO_DMATEL` and `BLASFEO_SMATEL` macros, while the
library packs matrices with its own panel size. A few exported symbols also
differ between the variants. By default, nothing prevents a package compiled
against one variant from running with another, which silently gives wrong
results for the affected functions (see
[#25](https://github.com/conda-forge/blasfeo-feedstock/issues/25)).

## How recipes use these packages

### A. The recipe only uses the double-precision API

The double-precision layout is the same for the enabled levels (1 and 3), so
no extra care is needed:

```yaml
requirements:
  host:
    - libblasfeo
```

### B. The recipe uses the single-precision matrix API, accesses the matrix memory directly, or uses symbols that only exist in some variants

Also add `libblasfeo-microarch-abi` to `host`:

```yaml
requirements:
  host:
    - libblasfeo
    - libblasfeo-microarch-abi
```

Its `run_exports` pins the exact microarch level, and `libblasfeo` constrains
`libblasfeo-microarch-abi` to its own level, so the package only runs with the
`libblasfeo` variant it was compiled against. HPIPM is an example of this case.

If the recipe exposes BLASFEO types in its public headers and has a `*-devel`
output, add `libblasfeo-microarch-abi` to the `run` requirements of that
output too, so its consumers get the same pin.
