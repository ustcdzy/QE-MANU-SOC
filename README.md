# QE-MANU-SOC

Patch for Quantum ESPRESSO adding continuous SOC scaling via `soc_scale` in `pw.x`.

## Overview

This patch adds a `soc_scale` parameter to `&SYSTEM`. Set it alongside `noncolin=.true.` and `lspinorb=.true.` to continuously control the strength of spin-orbit coupling:

- `soc_scale=0` gives the scalar-relativistic limit (same as `lspinorb=.false.`)
- `soc_scale=1` gives the original fully relativistic result (default)

The parameter must be non-negative.

## Installation

Apply the [qe66-manu-soc.patch](qe66-manu-soc.patch) file to your QE source tree and rebuild:

```bash
cd /path/to/espresso
patch -p1 < qe66-manu-soc.patch
make -j4 pw
```

## Usage

```text
&SYSTEM
  noncolin = .true.
  lspinorb = .true.
  soc_scale = 0.5    ! Scale SOC to 50%
/
```

## Background

QE does not expose SOC as a single tunable term. The difference between `lspinorb=.true.` and `.false.` is handled at the pseudopotential level: `average_pp` averages each `j=l+1/2` / `j=l-1/2` channel pair into one scalar channel, shrinks the channel arrays, and sets `has_so=.FALSE.`.

This patch works at the same level. For each `j` pair in a norm-conserving fully relativistic UPF, it computes the `average_pp` target values for the projector functions (`beta`), the Kleinman-Bylander `D` matrix (`dion`), and the atomic wavefunctions (`chi`), then interpolates between the original and averaged values:

```text
X_new = X_avg + soc_scale * (X_old - X_avg)
```

The averaging weights are the same `(l+1)/(2l+1)` and `l/(2l+1)` used by `average_pp`. Unlike `average_pp`, both `j` channels are always kept and no arrays are resized. At `soc_scale=0` the two channels collapse to identical values; the SO matrices built downstream are then effectively zero.

## Test results

bcc Fe, same pseudopotential and parameters throughout:

```text
soc_scale=0.0  ->  -250.61671861 Ry   (matches official lspinorb=.false.)
soc_scale=0.5  ->  -250.62084357 Ry
soc_scale=1.0  ->  -250.62323619 Ry   (matches official lspinorb=.true.)
```

## Limitations

Only norm-conserving fully relativistic UPF pseudopotentials are supported. FR ultrasoft and PAW will abort if `soc_scale` is set to anything other than `1.0`. Only the `pw.x` code path is patched.

Carefully tested with QE 6.6. Versions after 6.7 are not yet supported; patches for newer versions will be added when time permits. Any issues are welcome.

## License

GPLv2, same as [upstream QE](https://github.com/QEF/q-e/blob/master/License). See [LICENSE](LICENSE).
