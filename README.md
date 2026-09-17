# curves-nv

A Bézier curve is a smooth line defined by a small number of control
points, and it is what almost every drawing on a screen is made of:
font outlines, vector illustrations, animation paths. This package
provides the four families that appear in two-dimensional drawing —
quadratic and cubic Béziers, elliptical arcs, and B-spline and
Catmull–Rom splines — as values, with the operations that measure,
split and draw them. The reference implementation it is measured
against is the Rust crate [`kurbo`](https://docs.rs/kurbo).

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What a curve is here

A **Bézier curve** of degree `n` is defined by `n + 1` **control
points**. It starts at the first, ends at the last, and is pulled
towards the ones between without passing through them. A **quadratic**
has three control points; a **cubic** has four.

A curve is written as a function of a **parameter**, conventionally
called `t`, running from 0 at the start to 1 at the end. The parameter
does **not** move at constant speed along the curve. On a cubic whose
controls are bunched at one end, `t = 0.5` can be a long way from the
halfway point of the curve. That single fact is behind most of this
package's surface.

**Arc length** is the distance travelled along the curve. A cubic's has
no closed form — it is an integral that cannot be written down — so it
is computed by numerical integration to a **tolerance** the caller
gives. A quadratic's does have a closed form and is exact. The
**inverse**, the parameter at which a given length has been covered, is
what a caller placing dashes, arrowheads or text along a path needs.

**Flattening** turns a curve into a polyline whose greatest deviation
from it is no more than a tolerance. It is how a curve is eventually
drawn, because hardware draws line segments.

**Subdivision** splits a curve at a parameter into two curves that
together are the original, exactly. The algorithm is **de Casteljau's**,
and it is repeated subdivision that flattening is built on.

An **elliptical arc** is a piece of an ellipse. It is not a polynomial,
so no Bézier is equal to one, and a conversion in that direction is
always an approximation.

A **B-spline** is a curve defined by control points and a **knot
vector**, a non-decreasing list of numbers that decides how much of the
curve each control point affects. It approximates its control points. A
**Catmull–Rom** spline interpolates: it passes through every one of its
points.

| Family | Passes through its points | Arc length |
| --- | --- | --- |
| Quadratic Bézier | Endpoints only | Closed form |
| Cubic Bézier | Endpoints only | Numerical |
| Elliptical arc | Endpoints only | Closed form if circular |
| B-spline | Endpoints only, when clamped | Numerical |
| Catmull–Rom | Every point | Numerical |

## Install

```
novo pkg add curves-nv
```

## Example

```novo
use std.list
use curvepoint
use curvecubic

fn main() [io]
    // A cubic that arches from (0,0) to (1,0), with both controls
    // pulled upwards.
    let c = curvecubic.cubic(curvepoint.pt(0.0, 0.0), curvepoint.pt(0.0, 1.0),
                             curvepoint.pt(1.0, 1.0), curvepoint.pt(1.0, 0.0))

    // The exact bounding box. The curve reaches 0.75, not the 1.0 its
    // control points reach.
    println("${curvecubic.bounds(c).height}")

    // How long it is, to a tolerance this caller chose. No function
    // here has a hidden one.
    match curvecubic.arc_length(c, 1e-6)
        Err(e) => println("cannot measure: ${e.message()}")
        Ok(len) =>
            println("${len}")

            // The parameter at the halfway point BY LENGTH, which is
            // not t = 0.5.
            match curvecubic.t_for_length(c, len / 2.0, 1e-6)
                Ok(t)  => println("${t}")
                Err(e) => println("cannot invert: ${e.message()}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: curves-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `curvefault` | The few ways an operation refuses, and whether the caller's arguments caused it. |
| `curvepoint` | The point, the vector and the rectangle this package is written in, and the conversions to geometry-nv's. |
| `curvecubic` | The cubic Bézier and every operation, which the other families reduce to. |
| `curvequad` | The quadratic Bézier, whose arc length is exact, and the conversions to and from a cubic. |
| `curvearc` | The elliptical arc in both parameterisations, and the conversion between them. |
| `curvespline` | B-splines and Catmull–Rom splines, and the three spacing rules. |
| `curveseg` | One type over the four families, and the path operations over a list of them. |

## How to choose an entry point

**`curvecubic` is where the work is.** If you have a cubic, use it
directly: the segment union costs a match on every call.

**`curvequad.arc_length` takes no tolerance.** Use a quadratic rather
than an elevated cubic when the length matters, and when the source is
TrueType, whose outlines are quadratics.

**`curvearc` keeps an arc an arc.** Convert to cubics only when
something downstream insists, and read `cubic_error` when you do.

**`curveseg` is for a path.** Use it when the pieces are of mixed
kinds: `path_length`, `path_flatten`, `path_bounds` and
`point_at_distance` are written once over the union.

**`curvespline.catmull_rom` is for points you must pass through**, and
`clamped_bspline` is for control points a user drags.

## The rules a user needs

1. **Every tolerance is an argument.** No function here has a default
   one. A flattener with a hidden tolerance draws a different polyline
   on every caller's screen, and a caller drawing at four times the
   zoom cannot ask for four times the fidelity.

2. **A tolerance of zero is refused.** Flattening to zero does not
   terminate: the error bound never falls below zero. Clamping it to
   some small positive number would be choosing the caller's precision
   silently.

3. **The parameter is not the arc length.** `t = 0.5` is not the
   halfway point of a cubic. Spacing marks along a path by parameter
   puts them visibly closer together wherever the curve is tight.
   `t_for_length` and `curveseg.point_at_distance` are the functions
   that space them correctly.

4. **A cubic's arc length is numerical and a quadratic's is exact.**
   The cubic's integrand has no antiderivative in closed form. The
   quadratic's integrates to a logarithm. An ellipse's is the complete
   elliptic integral of the second kind, which is why the
   circumference of an ellipse has no formula.

5. **Exact bounds are smaller than control-hull bounds.** The arch in
   the example above reaches 0.75, and its control points reach 1.0.
   `bounds` is exact and costs the roots of a quadratic; `hull_bounds`
   is the control points' box and costs four comparisons. Use the
   second for a broad-phase test and the first when the answer has to
   be right.

6. **An arc is not a Bézier.** A circular arc is not a polynomial, so
   no Bézier equals one. `to_cubics` splits at 90 degrees or less and
   approximates each piece; the greatest error is about one part in
   40000 of the radius. `cubic_error` reports it.

7. **Converting an endpoint arc to a centre arc can change the radii.**
   The SVG specification (Appendix B.2.4) requires radii too small to
   reach between the endpoints to be scaled up together until they just
   reach. That silently changes the shape a caller asked for, and it is
   what every arc implementation gets wrong once.

8. **Catmull–Rom is centripetal here.** The uniform formulation, which
   is what "Catmull–Rom" means in most libraries, cusps and
   self-intersects when knots are close together and then far apart.
   The centripetal formulation is proved never to (Yuksel, Schaefer and
   Keyser, 2011). `may_self_intersect` says which you have.

9. **An unclamped B-spline does not start at its first control point.**
   `clamped_bspline` is the one that does. `uniform_bspline` starts
   somewhere inside the control polygon, which surprises everybody
   once.

10. **A degree-elevated quadratic is exact; the other direction is
    not.** `to_cubic` gives the same curve at the same parameter.
    `of_cubic` answers the error as well as the curve, because a cubic
    has an inflection a quadratic cannot have.

## What is not included

- **Three-dimensional curves.** Everything here is planar. A 3-D
  Bézier shares the algebra and none of the geometry: bounds,
  intersection and offsetting are different problems.
- **Rational curves and NURBS.** A rational Bézier carries a weight per
  control point and can represent a circle exactly, which is exactly
  what `curvearc` is for here. Weighted splines are a later release.
- **Offset curves and stroke outlining.** The offset of a cubic is not
  a cubic, so an offsetter is an approximation with its own tolerance
  and its own joins and caps. It is a package on top of this one.
- **Curve fitting.** Finding the curve that best passes near a set of
  sampled points is a least-squares problem, not a curve operation.
- **Curve–curve intersection.** Only `intersect_line` is here. Two
  cubics meet in up to nine places and finding them is a
  subdivision-and-clipping algorithm several times the size of this
  package.
- **Boolean operations on regions.** Union, intersection and difference
  of filled paths need curve–curve intersection first.
- **Rendering.** Nothing here draws. `flatten` answers the polyline a
  renderer takes, and `curvepoint.to_geom_list` converts it to the
  type a vertex buffer wants.

## Related packages

**geometry-nv** owns `GeomPointF`, `GeomVecF`, `GeomRectF` and
`GeomXform`. Its types are unboxed, which makes them the right thing
for a drawing loop and means they cannot appear in an optional, a
`Result`, a tuple or a boxed struct's field — which is every position
this package's API needs. So this package carries its own boxed
`CurvePoint` and converts, and `curvepoint` holds the six conversion
functions so that no consumer writes them.

**svg-nv** has `SvgPath`, a typed command list, and its own `flatten`.
The difference is what each is for: `SvgPath` is a path as a document
stores it, including the pen state and the close-path command, and
`svgpath.flatten` produces the polyline a renderer draws. This package
has no pen and no document; it has the curves, and the operations that
measure them.

**font-nv** reads glyph outlines, which are quadratics in TrueType and
cubics in PostScript flavours.

## Test vectors

There is no specification to quote, so the suite asserts the
mathematics.

- The arch from (0,0) to (1,0) with controls at (0,1) and (1,1) reaches
  exactly 0.75. Its control hull reaches 1.0. Both are asserted, which
  is what separates an exact bound from a hull bound.
- A straight cubic built by `curvecubic.line` is uniformly
  parameterised, so `eval(0.25)` is a quarter of the way along. That is
  asserted, because the control placement that makes it true is not the
  obvious one.
- `to_cubic` followed by `curvecubic.eval` agrees with `curvequad.eval`
  at the same parameter: degree elevation is exact.
- A semicircular arc converts to exactly two cubics, and its
  `cubic_error` is below 0.001 of the radius.
- The clamped knot vector for `n` controls at degree `d` is
  `n + d + 1` long.
- Centripetal Catmull–Rom answers false to `may_self_intersect`, and
  uniform and chordal answer true.

Today every one of those assertions reaches a `not implemented` panic.

## Implementation status

| Area | Status |
| --- | --- |
| Point, vector, rectangle, geometry-nv bridge | declared, not implemented |
| Cubic Bézier, all operations | declared, not implemented |
| Quadratic Bézier, with exact arc length | declared, not implemented |
| Elliptical arc, both parameterisations | declared, not implemented |
| B-spline and Catmull–Rom | declared, not implemented |
| The segment union and the path operations | declared, not implemented |
| Rational curves and NURBS | not declared — a later release |
| Offset curves and stroking | not declared |
| Curve–curve intersection | not declared |

## Licence

Apache-2.0. See [LICENSE](LICENSE).
