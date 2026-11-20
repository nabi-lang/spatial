<img src="https://raw.githubusercontent.com/nabi-lang/nabi/main/assets/icons/icon-512-transparent.png" alt="Nabi" width="96">

# spatial

[![CI](https://github.com/nabi-lang/spatial/actions/workflows/ci.yml/badge.svg)](https://github.com/nabi-lang/spatial/actions/workflows/ci.yml) [![packages.nabi-lang.org](https://img.shields.io/github/v/tag/nabi-lang/spatial?label=packages.nabi-lang.org&sort=semver)](https://packages.nabi-lang.org/packages/spatial) [![License: MIT OR Apache-2.0](https://img.shields.io/badge/license-MIT%20OR%20Apache--2.0-blue)](#license)

Game and graphics math for Nabi: vectors, quaternions, matrices, affine
transforms, projections and the shapes a renderer or a physics step tests
against. Every type is small, fixed-size and `#[repr(C)]`, so a `Vec3`,
a `Mat4` or a whole `struct` of them goes into a GPU buffer or across the
C ABI as it is. `Mat4` products run on SIMD lanes where the target has
them (SSE2, AVX2, NEON, wasm `simd128`).

Each type is generic over its component, `f32` when left out: `Vec3` is
`Vec3<f32>`, `DVec3` is `Vec3<f64>`, `IVec3` and `UVec3` are the `i32`
and `u32` vectors. For large n-dimensional arrays and factorizations, use
`linalg`.

## Add it

```sh
nabi add spatial
```

Needs Nabi 0.1 or later. Pure Nabi: every target works, wasm included.

## Usage

A game loop: move and turn a ship, build its model-view-projection
matrix, and pick it with a ray.

```nabi
use spatial::{Aabb, Mat4, Quat, Ray, Transform, Vec3};

struct Ship {
    body: Transform;
    velocity: Vec3;
    spin: f32;
}

fn step(ref ship: Ship, dt: f32) {
    ship.body.translation += ship.velocity * dt;
    ship.body.rotation = (Quat::from_rotation_y(ship.spin * dt) * ship.body.rotation).normalize();
}

let ship = Ship { body: Transform::from_translation(Vec3::new(0.0, 0.0, -10.0)), velocity: Vec3::new(0.0, 0.0, 2.0), spin: 1.0 };
let camera = Mat4::look_at_rh(Vec3::new(0.0, 2.0, 5.0), Vec3::ZERO, Vec3::Y);
let projection = Mat4::perspective_rh(60.0f32.to_radians(), 16.0 / 9.0, 0.1, 100.0);
let view_proj = projection * camera;

let dt = 1.0f32 / 60.0;
for _ in 0..120 {
    step(ref ship, dt);
    let mvp = view_proj * ship.body.to_mat4();   // upload mvp.to_cols_array()
    assert(mvp.project_point(Vec3::ZERO).z <= 1.0);  // in front of the far plane
}

let hull = Aabb::new(Vec3::splat(-1.0), Vec3::ONE).transform(ship.body.to_mat4());
let ray = Ray::new(Vec3::new(0.0, 0.0, 5.0), -Vec3::Z);
println(ray.intersect_aabb(hull));   // 9.674532, the distance to the hull
```

Vectors are values with the usual operators, a scalar on either side:

```nabi
use spatial::{DVec3, IVec2, Vec3};

let v = Vec3::new(1.0, 2.0, 2.0);
println(v.length());                 // 3
println(v.normalize() * 3.0 == v);   // true
println(Vec3::X.cross(Vec3::Y));     // (0, 0, 1)
println(v.zyx());                    // (2, 2, 1)
println(IVec2::new(3, 4) * 2);       // (6, 8)
println(DVec3::ONE.lerp(DVec3::ZERO, 0.25)); // (0.75, 0.75, 0.75)
```

Rotations compose right to left, as matrices do: `a * b` rotates by `b`
first. A quaternion rotates a vector with `q * v`.

```nabi
use spatial::{EulerOrder, Mat3, Quat, Vec3};

let yaw = Quat::from_rotation_y(0.5);
let pitch = Quat::from_rotation_x(-0.2);
let q = yaw * pitch;                                  // pitch, then yaw
let e = Quat::from_euler(EulerOrder::YXZ, 0.5, -0.2, 0.0);
println(q.approx_eq(e, 1e-6));                        // true
println((q * Vec3::Z).approx_eq(Mat3::from_quat(q) * Vec3::Z, 1e-6)); // true
let halfway = Quat::IDENTITY.slerp(q, 0.5);
println(halfway.is_normalized());                     // true
```

### Element types

Every type here declares `T = f32`, so `Vec3` is `Vec3<f32>`, and the
unsuffixed literals that fill `T` are `f32` too: `Vec3::new(1.0, 2.0, 3.0)`
is a `Vec3` (f32), and so is `Vec3::new(1, 2, 3)`. f64 code writes
`DVec3::new`, or an expected `DVec3` fixes it (`let v: DVec3 =
Vec3::new(1.0, 2.0, 3.0)`); a typed argument does too (`Vec3::new(x, 0.0,
0.0)` with `x: f64` is a `DVec3`). The constants follow the same rule:
`Vec3::X` is `f32` on its own and `f64` where a `DVec3` is expected, as in
`DVec3::new(1.0, 2.0, 3.0) + Vec3::X`. Numbers are never converted: an
`f64` scalar times a `Vec3` is an error whose help names `x as f32` and
`DVec3`.

## What is in it

| Item | Contents |
|---|---|
| `Vec2`, `Vec3`, `Vec4` | `+ - * /` with a vector or a scalar, `-`, `+=` and friends, `v[i]`; `dot`, `cross` (3D), `perp_dot` (2D), `length`, `normalize`, `try_normalize`, `normalize_or_zero`, `distance`, `lerp`, `midpoint`, `min`, `max`, `clamp`, `abs`, `floor`, `ceil`, `round`, `project_onto`, `reject_from`, `reflect`, `angle_between`, `approx_eq`, `map`; `xy`, `xz`, `yz`, `zyx`, `xyz`, `wzyx`, `extend`, ...; constants `ZERO`, `ONE`, `X`, `Y`, `Z`, `W` |
| `DVec2..4`, `IVec2..4`, `UVec2..4` | the `f64`, `i32` and `u32` vectors (the integer ones without the float functions) |
| `vec2`, `vec3`, `vec4`, `dvec3`, `ivec3`, `uvec3`, ... | constructors with a fixed element type: `vec3(1.0, 2.0, 3.0)` is a `Vec3` (`f32`) |
| `Vec3A` | a `Vec3` padded to 16 bytes and aligned to 16, as std140 / std430 / WGSL lay out a `vec3` |
| `Quat` | `from_axis_angle`, `from_rotation_x/y/z`, `from_euler` / `to_euler`, `from_mat3`, `from_rotation_arc`, `to_axis_angle`, `*` (compose, rotate a `Vec3`), `inverse`, `conjugate`, `normalize`, `slerp`, `nlerp`, `angle_between`, `approx_eq` |
| `Mat2`, `Mat3`, `Mat4` | column-major: `from_cols`, `from_cols_array`, `to_cols_array`, `col`, `row`, `*` with a matrix, a vector or a scalar, `transpose`, `determinant`, `inverse`, `try_inverse`, `approx_eq`; rotations from axes, angles, quaternions and Euler angles; `Mat4::from_scale_rotation_translation` and back, `transform_point`, `transform_vector`, `project_point`, `look_at_rh/lh`, and the projections below; `Mat3::transform_point2` for 2D |
| `Affine2`, `Affine3` | a matrix plus a translation: `transform_point`, `transform_vector`, `*`, `inverse`, to and from `Mat3` / `Mat4` |
| `Transform` | translation, rotation and scale kept apart: `to_mat4`, `transform_point`, `forward`, `right`, `up`, `looking_at`, `*` (parent times child), `inverse`, `lerp` |
| `Aabb`, `Ray`, `Plane`, `Sphere` | `contains`, `intersects`, `union`, `extend`, `closest_point`, `Aabb::transform`; `Ray::intersect_aabb`, `intersect_plane`, `intersect_sphere`; `Plane::from_points`, `distance`, `project` |
| `EulerOrder` | `XYZ`, `XZY`, `YXZ`, `YZX`, `ZXY`, `ZYX` |
| `Scalar`, `Real` | the component bounds: `f32`, `f64`, `i32`, `u32`; `Real` is the two floats |

### Conventions

- Column vectors: `m * v`, and `a * b` applies `b` first.
- Matrices are stored column by column (`x_axis`, `y_axis`, `z_axis`,
  `w_axis`), as GLSL, WGSL, HLSL `column_major`, Metal and glTF read them.
- Angles are radians; positive turns counterclockwise looking down the
  axis toward the origin.
- Right-handed unless a name ends in `_lh`.
- `from_euler(order, a, b, c)` is `R_a * R_b * R_c` about the axes `order`
  names: on a vector, `c` applies first. `YXZ` is yaw, pitch, roll for a
  y-up world; `ZYX` for a z-up one. `to_euler` gives `b` in
  `-pi/2..=pi/2`.

### Projections

Each projection comes in four forms, for the handedness of view space and
the depth range of the API's clip space:

| | depth `0..1` (Vulkan, Metal, Direct3D, WebGPU) | depth `-1..1` (OpenGL) |
|---|---|---|
| right-handed, camera looks down -z | `perspective_rh`, `orthographic_rh` | `perspective_rh_gl`, `orthographic_rh_gl` |
| left-handed, camera looks down +z | `perspective_lh`, `orthographic_lh` | `perspective_lh_gl`, `orthographic_lh_gl` |

Pair `look_at_rh` with an `_rh` projection and `look_at_lh` with an `_lh`
one. Vulkan's clip space has +y down; flip `y_axis.y` of the projection
(or the viewport) there.

### Layout

| Type (`f32`) | Bytes | Align | Notes |
|---|---|---|---|
| `Vec2`, `Vec3`, `Vec4` | 8, 12, 16 | 4 | components in order, no padding |
| `Vec3A` | 16 | 16 | a GPU `vec3` in a uniform or storage buffer |
| `Quat` | 16 | 4 | `x, y, z, w` |
| `Mat2`, `Mat3`, `Mat4` | 16, 36, 64 | 4 | column by column |
| `Affine2`, `Affine3` | 24, 48 | 4 | matrix, then translation |
| `Transform` | 40 | 4 | translation, rotation, scale |

The `f64` forms are twice the size; `DVec3A` is 32 bytes. A WGSL
`mat3x3<f32>` pads each column to 16 bytes: upload `Mat4::from_mat3(m)`
or three `Vec3A` columns.

### Inspectors and saving

Every type derives `Inspect`, so an engine's inspector or a REPL lists
and edits the fields by name (`Vec3::<f32>::field_names()`,
`v.set_field("y", 2.0f32)`). With the feature `nabion` they also derive
`Nabion` and read and write `.nabion` scene files:

```sh
nabi add spatial --features nabion
```

## Accuracy and speed

`f32` results agree with the same formulas in `f64` to about 1e-6
relative; `approx_eq` takes the tolerance. `normalize` of a zero vector
gives NaN components; `try_normalize` gives `none`. `inverse` of a
singular matrix gives infinite or NaN entries; `try_inverse` gives `none`.

`Mat4` products run on SIMD lanes: four lane-wise multiply-adds per
column, for `f32` and `f64` alike. A `Vec4` dot product and the
component-wise operators stay scalar, because packing four fields into a
vector register and reducing it back costs more than it saves. `nabi
bench`, 4096 operations per iteration, `-O2`:

| Operation | Apple M, SIMD | Apple M, scalar | x86_64, SIMD | x86_64, scalar |
|---|---|---|---|---|
| `Mat4 * Mat4`, `f32` | 108 us | 173 us | 332 us | 416 us |
| `Mat4 * Mat4`, `f64` | 65 us | 113 us | 233 us | 353 us |
| `Vec4` dot, `f32` (`Simd` + reduce vs `Vec4::dot`) | 17.9 us | 16.6 us | 68.7 us | 47.1 us |
| `Vec4` dot, `f64` | 13.8 us | 3.5 us | 49.5 us | 11.1 us |

## Docs

https://packages.nabi-lang.org/packages/spatial/latest/docs

## License

Licensed under either of [MIT](LICENSE-MIT) or [Apache-2.0](LICENSE-APACHE), at
your option.
