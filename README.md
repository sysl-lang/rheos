# Rheos

3D physics in [sysl](https://sysl.sh), built to stress the language and drive compiler development.
Pronounced **REE-oss** (/ˈriːɒs/), from the Greek word for flow.

Spheres, cuboids, ellipsoids and asymmetric convex solids exchange linear and angular momentum through
normal impulses and Coulomb friction. Bodies carry quaternion orientation and a full symmetric inertia
tensor. The physics is written in sysl, with the quaternion package for rotations; it uses no C physics
engine. Tekton is a prospective consumer, with nex as a possible scripting layer later.

Compiler findings and tooling irritations are recorded in [SYSL_ISSUES.md](SYSL_ISSUES.md).

## Floating bodies demo

Twenty-four bodies drift and collide in zero gravity: four spheres, four cubes, four rectangular
cuboids, four ellipsoids, four scalene tetrahedra and four skewed wedges. Half start without spin,
so their later rotation comes from collisions. Irregular geometry is generated afresh for each seed.

The window and controls use **syslui-sdl**. A depth-tested SDL_GPU viewport draws the bodies with Metal
shaders. The same perspective camera projects the bodies and the enclosure. A point light illuminates
the bodies per pixel, with inverse-square distance attenuation, diffuse surface shading and a small
ambient contribution. Ellipsoid normals account for their unequal axes.

The demo currently needs macOS, a desktop session, sysl **0.8.11+**, SDL3 and PlutoVG.

```sh
cd examples/box
sysl build .
./rheos-box
```

Drag to orbit; scroll to zoom. The panel has Pause/Resume, Single step, Slow motion, New scene and Replay,
a mixed-shapes/spheres switch, and a friction toggle. It displays the current seed, physics steps,
collision impulses and kinetic energy. Restitution is 1; friction is 0.2 by default, so sliding contacts
dissipate energy and the motion gradually slows. Physics advances at 120 Hz; long display intervals are
clamped to 0.1 seconds to avoid a backlog after a stall.

### Random scenes and reproducibility

Startup gets a seed from OS entropy. **New scene generates a fresh seed** and randomizes positions,
velocities, orientations, sizes, masses, spins and irregular geometry. Placement rejects overlapping
bounding spheres, so scenes start separated and wholly inside the enclosure. **Replay restores the
current seed** with the current shape and friction settings. The seed is displayed and printed to the
terminal, and `--seed` reproduces a scene in another run with the same build.

```sh
./rheos-box --seed 42
./rheos-box --simulate 7200 --seed 42                  # one minute without a window
./rheos-box --frames 120 --seed 42 --shot frame.png    # fixed frame steps, capture and exit
./rheos-box --control /tmp/rheos.sock                  # syslui-sdl control socket
```

`--hidden` creates an invisible window for automation; it still needs a working desktop display.
Control names are `pause`, `step`, `speed`, `reset` (New scene), `replay`, `shapes` and `friction`.

## Library

The module is `sh.sysl.rheos`. Lengths are metres, mass is kilograms, time is seconds, and angular
velocity is radians per second in body coordinates. Body position is its centre of mass. Y is up in
the demo.

```sysl
import sh.sysl.rheos.*

var bodies = [
    ellipsoid(Vec3(-0.9,0.3,0.0), Vec3(1.0,0.0,0.0), semi_axes = Vec3(0.4,0.25,0.3)),
    cuboid(Vec3(0.0,0.0,0.0), Vec3(0.0,0.0,0.0), half_size = Vec3(0.5,0.6,0.4)),
]
val room = Enclosure(Vec3(3.0,3.0,3.0), restitution = 1.0)
val stats = step(bodies,room,1.0/120.0).unwrap()
```

`sphere`, `cuboid`, `ellipsoid` and `convex` construct bodies. `hull(vertices, faces)` validates a
closed, outward-wound convex triangular mesh, copies its geometry, computes volume, centre of mass
and all six independent inertia entries, and recentres its vertices. It returns `Result[&Hull,string]`.
Several `convex` bodies can share a hull; geometry views retain their backing storage. Hull construction
allocates; successful stepping uses fixed scratch storage and does not allocate geometry.

A sphere's default friction is 0, preserving the original smooth-sphere API; the other constructors
default to 0.3. Pair friction is the geometric mean; walls use the body's coefficient. Coefficients
must be in [0,1]. Cuboid half sizes and ellipsoid semi-axes are in local coordinates. `radius` is a
bounding-sphere radius for all other shapes; bounds must fit inside the enclosure in every orientation.

### Dynamics and contacts

`Body.inertia()` returns the full `Inertia` tensor in body coordinates. Its `x`, `y`, `z` fields are
the diagonal; `xy`, `xz`, `yz` are the off-diagonal terms. Spheres, cuboids and ellipsoids have analytic
diagonal tensors. Convex hull tensors come from tetrahedral volume integrals. Quaternion rotations
apply the tensor in world space without allocating matrices. Implicit midpoint Euler equations and a
Cayley quaternion update preserve torque-free energy and world angular momentum to iteration tolerance.

Spheres and boxes retain their specialised collision paths. Oriented boxes use 15-axis SAT and clipped
face manifolds; edge contacts use closest points on supporting edges. Ellipsoids have **analytic support
mappings**, so their physics surface is independent of the rendered mesh's tessellation. Other pairs
use GJK/EPA, reconstructing shared contact points from support witnesses. Their contact accuracy is
`1e-5 * (a.radius + b.radius)`; shallower overlaps are treated as touching.

The solver accumulates normal impulses and clamps tangential impulses to a two-dimensional Coulomb
friction disk. Four contact sweeps handle short chains. Adaptive substeps bound translation and rotation
using energy and inertia, up to 4,096 subdivisions per call. `step` accepts time steps from 0 to 0.1 s.

`contacts_checked(a,b)` returns a manifold or an explicit convergence error; `contacts(a,b)` unwraps
that result for callers treating failure as a programming error. GJK and EPA have finite iteration and
scratch-capacity limits. `step` validates input before changing bodies, but a numerical query failure
can leave a partially advanced step. The demo pauses and shows the error; its seed permits replay.

The solver draws on [Erin Catto's publications](https://box2d.org/publications/) and Box2D's contact solver;
[Catto's GJK talk](https://box2d.org/files/ErinCatto_GJK_GDC2010.pdf) and the
[dyn4j EPA explanation](https://dyn4j.org/2010/05/epa-expanding-polytope-algorithm/) are algorithm references.

There is no persistent contact cache, warm starting, gravity, joints, concave mesh decomposition,
spatial tree or continuous time-of-impact solver yet. Hull input must already be a convex solid;
`hull` does not build a convex hull from an arbitrary point cloud. Discrete steps can miss grazing
impacts, dense initial overlaps may not fully resolve, and positional penetration correction is
geometric rather than momentum conserving. Multi-contact restitution is iterative.

## Verification

```sh
sysl test .
cd examples/box
sysl test .
```

The 47 core tests cover analytical collisions and friction impulses, momentum, energy, world inertia,
free tumbling, full polyhedral mass properties, ellipsoid support and penetration, convex queries against
SAT, nearly touching geometry, geometry-view ownership, fast motion and invalid input. AddressSanitizer
checks the core with instrumentation confirmed.

Seven numerical demo tests cover long motion, dissipation, energy growth, seeded replay, non-overlapping
packing and several random mixed scenes. The GPU test compiles the real body shaders and their point
light, but prints a skip if it cannot access a desktop display. Successful test stdout is normally
hidden by the runner; see the issue log. Two additional UI tests check that collision diagnostics
wrap within the panel without dropping words.

The reported long-run failure (seed `17727295752622095070`, friction off, step 19,167) has an isolated
unit regression. The corrected build completed a five-minute replay (36,000 steps):

```sh
./rheos-box --simulate 36000 --seed 17727295752622095070 --no-friction
```

`--no-friction` starts both interactive and numerical runs with friction disabled.

The previous sphere/box renderer was confirmed visually by the user. The new ellipsoid/hull renderer
and lighting still need a desktop check: SDL cannot access a display in the development sandbox, and
the standalone Metal toolchain is unavailable here.

This package is unreleased; the demo consumes this checkout through a relative path dependency.

## Licence

ISC. See [LICENSE](LICENSE).
