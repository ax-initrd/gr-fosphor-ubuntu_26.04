[PATCH 0/2] fosphor/cl: Make the non CL/GL-sharing path work with PoCL

The non CL/GL-sharing fallback path currently can't initialize at all on
PoCL, which makes fosphor unusable on any machine without a working GPU
OpenCL stack. Two independent issues, one patch each:

 1) The waterfall/histogram images are created as CL_R / CL_FLOAT. PoCL
    doesn't support single channel images at all, it only ever exposes
    CL_RGBA and CL_BGRA, so clCreateImage() fails with
    CL_INVALID_OPERATION.

 2) Even once the images are CL_RGBA, PoCL rejects clEnqueueFillImage()
    on them with CL_IMAGE_FORMAT_NOT_SUPPORTED. Since the clear only
    happens in the CL_BOOTING state, the failure leaves fosphor retrying
    it forever and nothing is ever displayed.

Both patches keep the existing behaviour as the fast path and only fall
back when the runtime actually refuses, so nothing changes on a GPU that
does support CL_R images and image fills. In particular the kernels are
untouched: display.cl already reads and writes float4 and only ever uses
the .x component, so the fallback only has to widen the host side buffer
stride and the GL upload format.

Tested on a ThinkPad X220 (Sandy Bridge, HD 3000) running Ubuntu 26.04
with PoCL 6.0 and Mesa 26.0.8 (crocus for GL), driving osmocom_fft with
an RTL-SDR: waterfall, histogram and spectrum all render correctly.
Also sanity checked that the CL_R path is still selected when the
runtime advertises it.

  fosphor/cl: Fall back to CL_RGBA images if CL_R is not supported
  fosphor/cl: Fall back to a host side fill if clEnqueueFillImage fails

 lib/fosphor/cl.c      | 100 ++++++++++++++++++++++++++++++++++++++++++++------
 lib/fosphor/fosphor.c |   6 ++-
 lib/fosphor/gl.c      |  10 +++--
 lib/fosphor/private.h |   1 +
 4 files changed, 99 insertions(+), 18 deletions(-)

-- 
2.54.0 (Apple Git-157)

