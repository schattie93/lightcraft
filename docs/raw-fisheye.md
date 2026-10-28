# DNG WarpFisheye

Implemented independently from the published [DNG specification 1.6.0.0](https://paulbourke.org/dataformats/dng/dng_spec_1_6_0_0.pdf),
chapter 7, WarpFisheye, pages 92–94. No SDK or external decoder implementation was used.

For a corrected pixel, normalize its displacement from the optical centre by the distance to
the farthest image corner. With radius `r` and `theta = atan(r)`, the source radius is
`k0 theta + k1 theta^3 + k2 theta^5 + k3 theta^7`. The continuous central limit of
`atan(r) / r` is 1. The raw path uses the existing bilinear sampler; the application uses its
existing geometry resampler. One coefficient set applies to RGB, or three sets apply per plane.
Non-finite, excessively large or malformed parameter records remain unknown; direct calls
with invalid parameters leave the image unchanged. Full-image passes retain the existing cap.

## Application path

`OpcodeList3` executes in order, including the existing DNG 1.6 skip rule: a Fisheye directly
after an optional WarpRectilinear2 is a fallback and is not applied again. The file loader
keeps lens correction out of the editable source pixels and carries it in `EmbeddedLens`.
The original normalization radius and optical centre are retained through crop and EXIF
orientation. A fisheye-containing list uses the existing ordered warp stages (at most three)
and CPU geometry path, including when the remaining rendering stages run on the GPU.
Changing output size changes neither the normalized geometry nor channel registration.
The existing profile correction switch and distortion strength apply to it.

The scalar atan calculation has no claimed portable interval enclosure. Block coverage
therefore falls back to the existing per-pixel checks for these warps, instead of claiming an
unsafe bound. Old serialized warp stages omit the new false `fisheye` field and read as before.

A cropped fisheye cannot in general be written as another four-coefficient WarpFisheye:
`atan(s r)` is not a finite polynomial in `atan(r)`. Merge loading therefore materialises the
**whole ordered lens list** before the default crop when an applied Fisheye is present. Those
merged pixels are already corrected, and the output carries no duplicate lens opcode. This
means its correction cannot subsequently be disabled on the merged DNG. Other merge inputs
continue to carry reframed lens metadata. `Opcode::cropped` deliberately returns None for
Fisheye; callers must not silently treat that as permission to discard its geometry.

## Evidence and limits

Tests use original procedural images and independently evaluated coordinates from the
specification. They cover a non-central optical centre, individual RGB coefficients, the
centre limit, opcode parse/write and full DNG round trips, execution order, optional fallback
skips, the pass cap, malformed parameters, small images, crop, all eight orientations,
resolution scaling, editable-source separation, serialization and merge materialisation.
A real camera DNG carrying opcode 2 has not yet been pinned or independently compared;
this change adds no verified camera model and claims no measured camera or Lightroom parity.

RENDER_CACHE_VERSION and LOOK_VERSION advance so stale renders and DNG proxies are rebuilt.
This is a source-decoding correction, not a new develop process (`process-versions.md`).
