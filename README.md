# gr-fosphor on Ubuntu 26.04

Patches to make [gr-fosphor](https://gitea.osmocom.org/sdr/gr-fosphor)'s
non CL/GL-sharing path work with [PoCL](https://portablecl.org/), so that
fosphor's waterfall renders on machines with no usable GPU OpenCL stack.

This is what you need if you have an older Intel GPU (Sandy Bridge and
similar) where `beignet` is long gone and neither the Intel compute runtime
nor rusticl will give you an OpenCL device.

## The problem

fosphor falls back to a copy-through-host-memory path when
`cl_khr_gl_sharing` isn't available. On PoCL that fallback can't initialize
at all. Two independent issues:

1. **The images can't be created.** The waterfall and histogram images are
   created as `CL_R` / `CL_FLOAT`. PoCL doesn't support single channel
   images at all — it only ever exposes `CL_RGBA` and `CL_BGRA` — so
   `clCreateImage()` fails with `CL_INVALID_OPERATION` (`-59`).

2. **The images can't be cleared.** Even once they're `CL_RGBA`, PoCL
   rejects `clEnqueueFillImage()` on them with
   `CL_IMAGE_FORMAT_NOT_SUPPORTED` (`-10`). Because the clear only happens
   in the `CL_BOOTING` state, the failure leaves fosphor retrying it
   forever and nothing is ever displayed.

## The fix

| Patch | Change |
| --- | --- |
| `0001` | Probe with `clGetSupportedImageFormats()` and fall back to `CL_RGBA` when `CL_R` isn't available. |
| `0002` | Fall back to a host side fill via `clEnqueueWriteImage()` when `clEnqueueFillImage()` is refused. |

Both keep the existing behaviour as the fast path and only fall back when
the runtime actually refuses, so nothing changes on a GPU that does support
`CL_R` images and image fills.

The OpenCL kernels are untouched: `display.cl` already reads and writes
`float4` and only ever uses the `.x` component, so the fallback only has to
widen the host side buffer stride and the GL upload format.

```
 lib/fosphor/cl.c      | 100 ++++++++++++++++++++++++++++++++++++++++++++------
 lib/fosphor/fosphor.c |   6 ++-
 lib/fosphor/gl.c      |  10 +++--
 lib/fosphor/private.h |   1 +
 4 files changed, 99 insertions(+), 18 deletions(-)
```

## Building

The patches are `git format-patch` output against upstream commit
`74d54fc`, and apply cleanly to the Ubuntu 26.04 source package
(`3.9~git20240323.74d54fc-1ubuntu1`).

Enable source repositories, then:

```sh
sudo sed -i 's/^Types: deb$/Types: deb deb-src/' /etc/apt/sources.list.d/ubuntu.sources
sudo apt update
sudo apt install dpkg-dev devscripts quilt
sudo apt build-dep gr-fosphor

apt-get source gr-fosphor
cd gr-fosphor-3.9~git20240323.74d54fc

cp /path/to/000*.patch debian/patches/
(cd debian/patches && printf '%s\n' 000*.patch >> series)

DEB_BUILD_OPTIONS="nocheck nodoc parallel=4" dpkg-buildpackage -us -uc -b -j4
```

Then install the two resulting packages:

```sh
sudo apt install ../gr-fosphor_*.deb ../libgnuradio-fosphor3.9.0_*.deb
```

## Runtime requirements

You also need an actual OpenCL implementation. The packages only depend on
`ocl-icd-libopencl1`, which is just the ICD *loader* — without an
implementation you still get zero OpenCL devices:

```sh
sudo apt install pocl-opencl-icd
```

Verify with `clinfo -l` that a device shows up before launching
`osmocom_fft -F`.

> **Note on upgrades.** The rebuilt packages carry the same version string
> as the ones in the Ubuntu archive, so apt sees no difference and can
> silently replace your patched build. Either hold them:
>
> ```sh
> sudo apt-mark hold gr-fosphor libgnuradio-fosphor3.9.0
> ```
>
> or bump the version in `debian/changelog` (e.g. to
> `3.9~git20240323.74d54fc-1ubuntu1+pocl1`) so it sorts above the archive
> version.

## Tested on

- ThinkPad X220, Core i5-2520M, Intel HD Graphics 3000 (Sandy Bridge, Gen6)
- Lubuntu 26.04, kernel 7.0
- PoCL 6.0, Mesa 26.0.8 (`crocus` for GL, OpenGL 3.3)
- GNU Radio 3.10.12.0, gr-fosphor 3.9~git20240323.74d54fc
- RTL-SDR (Nooelec NESDR SMArt v5, R820T) via `osmocom_fft -F`

Waterfall, histogram and spectrum all render correctly. The `CL_R` path is
still selected when the runtime advertises it, so GPU setups are unaffected.

Performance note: this is a CPU-side FFT plus a CPU-side OpenCL display
kernel, with a readback each frame instead of zero-copy texture sharing. On
the X220 that's roughly 200% CPU (of 400% available) at ~2.5 Msps, so expect
it to run out of headroom somewhere around 3–4 Msps.

