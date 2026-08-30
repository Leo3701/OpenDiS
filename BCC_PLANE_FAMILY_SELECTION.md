# Strict BCC slip-plane family selection

This modified OpenDiS tree supports a strict whitelist for BCC glide-plane
families. The whitelist applies to glissile `1/2<111>` segments while retaining
all original `<100>` junction planes and the collision, topology, remeshing,
cross-slip, and multiplication machinery.

## Repository baseline

The modified ExaDiS code is based on the official unified `main` branch at
commit `d550bc03ef76f1e9df7ceb29bc08f67e6ec189c5`. The integrated version is merge
commit `a44ee062f32504bb5acb38800e06a095f1473a8a`, which combines the complete
unified `main` history with the strict BCC plane-family implementation. It also
includes the upstream GPU `ForceFFT` complex-atomic fix by applying
`atomic_add()` to the real component of each FFT grid value.

The unified-memory baseline differs from the former `nounified` WSL baseline.
Use a clean build directory when switching to this version; existing
`nounified` build artifacts are not compatible and WSL/CUDA behavior must be
validated for the local toolchain.

## Python interface

Select exactly one family in the simulation `state` dictionary:

```python
state = {
    "crystal": "bcc",
    "bcc_plane_families": ["112"],  # or ["123"]
    # Remaining ExaDiS parameters...
}
```

The following forms are supported:

```python
"bcc_plane_families": "110"
"bcc_plane_families": ["112"]
"bcc_plane_families": ["123"]
"bcc_plane_families": ["110", "112"]
"bcc_plane_families": "all"
```

The equivalent low-level bit mask is:

| Family | Bit |
|---|---:|
| `{110}` | 1 |
| `{112}` | 2 |
| `{123}` | 4 |

For example, `bcc_plane_family_mask=2` permits only `{112}`, and
`bcc_plane_family_mask=4` permits only `{123}`.

Do not specify `bcc_plane_families` together with
`bcc_plane_family_mask` or `num_bcc_plane_families`. The Python
wrapper rejects these ambiguities.

Strict selection automatically enables `use_glide_planes=1` and
`enforce_glide_planes=1`. Explicitly disabling either option together with a
strict whitelist is rejected instead of silently allowing off-plane motion.

## Initial prismatic loops

The Python prismatic-loop generator accepts the same family explicitly:

```python
slip_family = 112
G.generate_prismatic_config(
    "bcc", Lbox, num_loops, radius, maxseg,
    uniform=True, plane_family=slip_family,
)
```

Use `plane_family=110`, `112`, or `123`, matching the selector in the simulation
state. `{110}` and `{112}` loops have six sides; `{123}` loops have twelve. Each
edge direction is constructed as `t = n x b`, so the edge geometry and stored
plane normal belong to the selected family. Omitting `plane_family` preserves
the original `{110}` default.

## Backward compatibility

When neither new option is present, `num_bcc_plane_families` keeps its original
behavior for values 1-3 and adds two strict selectors:

- `1`: `{110}`
- `2`: `{110}+{112}`
- `3`: `{110}+{112}+{123}`
- `4`: `{112}` only
- `5`: `{123}` only

For example, the following is equivalent to
`"bcc_plane_families": "112"`:

```python
"num_bcc_plane_families": 4
```

The legacy automatic BCC network generator also remains `{110}`-only. When the
strict whitelist is present, automatic generation uses all and only the selected
glissile systems.

## Important behavior

- `find_precise_glide_plane()` can snap a glissile segment only to a whitelisted
  family.
- `pick_screw_glide_plane()` chooses only from whitelisted planes.
- Cross-slip candidates are restricted to whitelisted planes. To prohibit all
  cross-slip within a selected family, omit the `CrossSlip` module separately.
- `<100>` junction Burgers vectors retain their original 16 zonal planes. This is
  required for BCC junction formation and topology operations.
- Collision, topology, remeshing, and source multiplication modules remain
  enabled by the simulation driver exactly as before.

## Verification

After rebuilding, run:

```bash
cd core/exadis/tests/unit_tests
OMP_PROC_BIND=false python3 run_tests.py --noplot
```

The `Test_BCC_Plane_Families` test checks strict `{110}`, `{112}`, and `{123}`
selection, orthogonality to each `1/2<111>` Burgers vector, preservation of all
junction planes, and legacy defaults.
