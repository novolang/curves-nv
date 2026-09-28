# Changelog

All notable changes to curves-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
