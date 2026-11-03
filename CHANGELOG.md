# Changelog

All notable changes to curves-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-11-03

### Fixed

- **`t_for_length` finds the length's parameter on a cubic with a cusp
  and on a quadratic that turns back on its line.**  The search keeps a
  bracket around the answer and bisects it wherever Newton's step does
  not exist (zero speed) or would leave the bracket.  Before, a first
  guess that landed on a zero-speed parameter was answered as is: half
  the length of the cusp cubic whose cusp is at `t = 0.5` answered
  `0.5`, where the half-length parameter is 0.7550689.  Applies to
  `curvecubic.t_for_length` and `curvequad.t_for_length`.

This release requires novo 0.23.0 or later.

## 0.1.0 — 2026-10-12

**The bodies land.** Every `pub fn` under `src/` is implemented and the
four suites the interface release shipped under `tests/` pass, so the
interface published as 0.0.1 is a working library.  No signature, no
effect row and no refusal changed: the tests are the specification the
bodies were fitted to, and every assertion in them is one the interface
release already made.

### Added

- `curvecubic` — `eval` is de Casteljau rather than the Bernstein sum,
  because Bernstein answers `-2.8e-17` where the suite asks for `0.0`;
  `segment` is the blossom, which puts `eval` at both ends by
  construction and so is exact at the ends where two cuts are not;
  `arc_length` is a five-point Gauss-Legendre quadrature refined
  pairwise to the caller's tolerance; `bounds` is the roots of a
  quadratic and not the control hull.
- `curvequad` — a closed-form `arc_length`: the integral of the speed
  is a polynomial term plus a logarithm, in three cases by the sign of
  the curvature.  `t_for_length` is a ratio on a straight uniformly
  parameterised quadratic, which is what makes `t_for_length(straight,
  1.0, 1e-9)` exactly `0.25`; `nearest_point` solves the cubic rather
  than searching; `of_cubic` drops the cubic term and bounds the error
  it left.
- `curvearc` — `to_center` is SVG Appendix B.2.4 including the radii
  scaling, and `to_endpoint` is B.2.3.  `eval` and `point_at_angle`
  are the closed form, so they are exact and not approximate.
  `arc_length` answers a circle exactly and integrates the general
  ellipse.  `bounds` is the endpoints plus whichever of the four axis
  extremes the sweep passes through.  `transform` takes the singular
  value decomposition of the linear part, which is what lets a
  non-uniform scale answer a tilted ellipse instead of a lie.
- `curvespline` — de Boor's algorithm with the clamped endpoint
  convention, so a clamped spline starts and ends exactly on its outer
  controls; a derivative that is itself a B-spline one degree lower,
  which makes the conversion to cubics exact for degrees 1 to 3; the
  non-uniform Catmull-Rom of Yuksel et al. for all three spacing
  rules.
- `curveseg` — the line's own formulas, and for an arc a coarse sweep
  and a ternary search for `nearest_point` (an ellipse's projection
  has no closed form) and a transformation onto the unit circle for
  `intersect_line`.  `point_at_distance` refuses a distance past the
  end rather than clamping, and `path_flatten` drops the repeated join
  points.
- **A fifth suite**, `tests/curves_nv_edge_tests.nv`, over the cases
  the API suites leave open: the two kinds of degenerate control point
  and the empty one that is refused, a closed cubic, a cusp, the
  flattening ceiling, the general (non-straight) nearest-point and
  line-intersection paths, and the corners of the two searches — an
  arc-length query whose first Newton step overshoots past either end
  of the domain, a tolerance below what a double holds, and a first
  guess that lands exactly on a cusp.
- **Two more pins, found by mutation**, each written because the
  mutation survived without them.  Negating `curvecubic.curvature`'s
  cross product left every suite green, because the edge suite pinned
  the quadratic's sign (the hump's apex is `-1`) and the cubic's only
  through a product that is invariant under negating curvature; the
  suite now checks a cubic that turns the positive way, against the
  parabola `y = 2t^2 - 2t` (`x = t`, whose speed at the apex is one and
  whose second derivative is four, so the answer is exactly four).  And raising `FLATTEN_LIMIT` 256-fold left
  every suite green, because the ceiling's test asserted the fault's
  code and not its value; the refusal's message ends with `the limit is
  4096`, and the suite now says so.

### Changed

- The `novo` floor is `>= 0.20.2`, the version the bodies were checked
  against.  The old `>= 0.9.0` was the floor of a release that had no
  bodies.
- `stability` moves from `draft` to `experimental`, and the
  description no longer says the bodies are unimplemented.  The
  interface-release comment above the key is gone with it.
- `README.md` loses its `NOT IMPLEMENTED` block and its implementation
  status table.
- **Two doc blocks cited the wrong SVG appendix.**  `to_center` said
  B.2.2, which is "Parameterization alternatives"; the
  endpoint-to-centre conversion is B.2.4.  `to_endpoint` said B.2.1,
  which is the endpoint syntax; the centre-to-endpoint conversion is
  B.2.3.  Both citations now name the section each function implements,
  which is what the README (item 7) already said.  Filed as
  `diagnostics/curvearc-doc-blocks-cite-the-wrong-svg-arc-appendix-for-both-conversions`;
  neither function's behaviour changed.
- **Two unreachable lines carry `// cov: skip` markers** rather than a
  test that cannot exist: `curvearc.cubic_error`'s `count == 0` (the
  list `to_cubics` answers opens at one piece and never shrinks) and
  `curveseg.arc_t_for_length`'s `total <= 0.0` (its only caller gates on
  the identical `curvearc.arc_length(a, tolerance)` under
  `span > 0.0`).  The publish gate reads both as excused, with the
  reason printed beside them.

### Known

- **`curvecubic.t_for_length` stops at a cusp rather than stepping over
  it.**  Its first guess is `length / total`, and when that lands on a
  parameter where the speed is zero the Newton step is a division by
  it; the guard answers the parameter it is standing on.  On the cusp
  cubic whose cusp is at `t = 0.5`, asking for half the total length
  answers `0.5` where the half-length parameter is 0.7550689, so the
  answer is a quarter of the curve short.  Pinned as observed — with
  that parameter written beside it — in
  `tests/curves_nv_edge_tests.nv`, and filed as
  `diagnostics/curvecubic-t-for-length-answers-a-cusp-when-the-first-guess-lands-on-one`;
  the fix is a bisection fallback, which is a change to the search's
  behaviour on inputs the interface release's doc does not describe.

- **`cubic_error` is a measurement on a panel**, not a bound: the
  divergence of the standard construction peaks at a fixed fraction
  along each piece, so the panel samples there and at sixteen even
  points between.  For a circular arc that is the exact maximum; for
  an ellipse it is the same measurement along the radius.
- **Nearest point on an arc is numerical.**  A circle's is closed-form
  and is not taken here, because the general elliptical case is the
  one the union cannot avoid; the answer is to the tolerance the
  caller gave.
- **Curve-curve intersection is still not declared at all.**  Two
  cubics meet in up to nine places and finding them is a
  subdivision-and-clipping algorithm several times the size of this
  package.
- **The `@value` gap still shapes the point types.**  If
  `a-value-struct-cannot-cross-a-boxed-boundary-so-a-fallible-codec-has-no-unboxed-return`
  is fixed, `CurvePoint` becomes an alias for `GeomPointF` and the six
  bridge functions become the identity.  That is a source change and
  not a signature change, so it does not break a consumer.

## 0.0.3 — 2026-10-25

A file that names a sibling module's type, function or constant now has the `use` line for that module (E2033, E2003).  No signature and no behaviour changed.

## 0.0.2 — 2026-09-28

The dependency ranges move to the dependencies' current releases.  A
pre-1.0 caret range admits only the release it names, so the old
ranges held this package on interface releases, and a program could
not take this package beside those packages' current releases.  No
signature in this package changed.

- geometry-nv: `^0.0.1` to `^0.1.0`.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `curvepoint` — **the decision the row asked for, and the compiler
  made it.** This package carries its own boxed `CurvePoint`,
  `CurveVec` and `CurveRect`, and depends on geometry-nv only for the
  six conversion functions that bridge to its types.

  geometry-nv's `GeomPointF` is the right type for a drawing loop: it
  is `@value`, so it is unboxed, register passed, and stored in a list
  as one flat buffer. SPEC § 14.5 therefore forbids it in an optional,
  a `Result`, a tuple, an enum payload or a boxed struct's field — and
  every one of those is a position this package's API needs.
  `CurveCubic` holds four points as fields; `subdivide` answers a
  tuple; `segment` answers a `Result`; `unit_tangent` answers an
  optional; `CurveSegment` is an enum. Making the curve types `@value`
  satisfies the first and fails the other four, because `@value` is
  viral upward with no ceiling. Each of the five positions is refused
  with E2015, under the filing
  `a-value-struct-cannot-cross-a-boxed-boundary-so-a-fallible-codec-has-no-unboxed-return`,
  open against the language.

  So the bridge is written ONCE, here, rather than at every call site
  in every consumer — which is the shape geometry-nv's own header
  comment anticipates. When that filing closes, these become aliases
  and the bridge becomes the identity.
- `curvecubic` — the centre. Every other family answers `to_cubics`,
  and arc length, nearest point, exact bounds and line intersection
  are written once here. `derivative_controls` answers a quadratic
  exactly, which is what makes `bounds` exact rather than the control
  hull. `bounds` and `hull_bounds` are BOTH public, because the exact
  one costs the roots of a quadratic and the hull one costs four
  comparisons, and a broad-phase test over ten thousand curves wants
  the second. `eval` clamps and the searches refuse, which is the
  right division: a `Result` around three linear interpolations is a
  `match` nobody reads.
- `curvequad` — kept as a type of its own rather than folded into the
  cubic, for two reasons that are both in its signatures: `arc_length`
  takes NO tolerance, because a quadratic's integrand has a closed
  form and a cubic's does not; and TrueType outlines are quadratics,
  so a caller that elevated them all on the way in would spend a third
  more memory on a page of text and could never convert back exactly.
  `of_cubic` answers the error beside the curve, because that
  direction is an approximation.
- `curvearc` — both parameterisations, because SVG's path syntax says
  where an arc ENDS and everything that draws one needs the CENTRE.
  `to_center` is SVG Appendix B.2.2, including the rule that radii too
  small to reach are scaled up together — the piece of arc code
  everybody writes once and gets wrong once. A zero radius is refused
  rather than drawn as a line: SVG's rule is right for a renderer and
  wrong for a library asked for an arc. `to_cubics` is an
  approximation and `cubic_error` says how large, so a caller checks
  it against its own tolerance instead of trusting a comment.
- `curvespline` — both families, because neither is a better version
  of the other: a B-spline approximates and is what a designer drags,
  a Catmull-Rom interpolates and is what an animator and a chart need.
  `catmull_rom` is CENTRIPETAL, which is the only one of the three
  spacing rules proved never to cusp or self-intersect; the uniform
  formulation that most libraries call "Catmull-Rom" does both.
  `may_self_intersect` tells a caller which it has.
- `curveseg` — the union, so a path is a list. A LINE is a segment of
  its own even though `curvecubic.line` exists, because a line's
  length is a subtraction and its nearest point a projection, and a
  path is mostly lines. An arc stays an arc inside the union until
  somebody asks for cubics, which is a conversion that cannot be
  undone.
- `curvefault` — eleven refusals and no more, because evaluating a
  cubic cannot fail. A tolerance of zero is the one worth naming: it
  does not terminate, and clamping it would be choosing the caller's
  precision silently.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented:
  curves-nv.<module>.<fn>`.
- **The `@value` gap shapes the whole package.** If
  `a-value-struct-cannot-cross-a-boxed-boundary-so-a-fallible-codec-has-no-unboxed-return`
  is fixed before 0.1.0, `CurvePoint` becomes an alias for
  `GeomPointF` and the six bridge functions become the identity. That
  is a source change and not a signature change, so it does not break
  a consumer.
- **Curve-curve intersection is not declared at all.** Two cubics meet
  in up to nine places and finding them is a subdivision-and-clipping
  algorithm several times the size of this package. Offset curves and
  boolean operations both need it first.
