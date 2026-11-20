# Changelog

All notable changes to this package. Versions follow semver, independent of the Nabi toolchain.

## 0.1.0 (unreleased)

First release.

- Vectors `Vec2`, `Vec3`, `Vec4` (f32 by default; `DVec*`, `IVec*`, `UVec*`)
  and `Vec3A`, padded as GPU buffers lay out a `vec3`.
- `Quat`, column-major `Mat2`, `Mat3`, `Mat4`, `Affine2`, `Affine3` and
  `Transform`, with Euler angles in every `EulerOrder`.
- Right- and left-handed perspective and orthographic projections for
  `0..1` and `-1..1` depth, and `look_at_rh` / `look_at_lh`.
- Shapes `Aabb`, `Ray`, `Plane`, `Sphere` with containment and intersection
  tests.
- Every type `#[repr(C)]` and deriving `Inspect`; the `nabion` feature adds
  `Nabion` for `.nabion` scene files.
- `Mat4` products on SIMD lanes where the target has them. Pure Nabi, every
  target including wasm. Needs Nabi 0.1 or later.
