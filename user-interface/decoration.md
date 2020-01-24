# Decoration

## What are decorations?

* A decoration is a high level description of graphics to be painted onto a canvas, generally corresponding to a box but may also describe other shapes, too. In addition to being paintable, decorations support interaction and interpolation.

## What are the common components of a decoration?

* `DecorationImage` describes an image \(as obtained via `ImageProvider`\) to be inscribed within a decoration, accepting many of the same arguments as `paintImage`. The alignment, repetition, and box fit determine how the image is laid out within the decoration and, if enabled, horizontal reflection will be applied for right-to-left locales.
  * `DecorationImagePainter` \(obtained via `DecorationImage.createPainter`\) performs the actual painting; this is a thin wrapper around `paintImage` that resolves the `ImageProvider` and applies any clipping and horizontal reflection.
* `BoxShadow` is a `Shadow` subclass that additionally describes spread distance \(i.e., the amount of dilation to apply to the casting element’s mask before computing the shadow\). Shadows are typically arranged into a list to support a single decoration casting multiple shadows.
* `BorderRadiusGeometry` describes the border radii of a particular box \(via `BorderRadius` or `BorderRadiusDirectional` depending on text direction sensitivity\). `BorderRadiusGeometry` is composed of four immutable `Radius` instances.
* `BorderSide` describes a single side of a border; the precise interpretation is determined by the enclosing `ShapeBorder` subclass. Each side has a color, a style \(via `BorderStyle`\), and a width. A width of `0.0` will enable hairline rendering; that is, the border will be 1 physical pixel wide \(`BorderStyle.none` is necessary to prevent the border from rendering\). When hairline rendering is utilized, pixels may appear darker if they are painted multiple times by the given path. Border sides may be merged provided that they share a common style and color. Doing so produces a new `BorderSide` having a width equal to the sum of its constituents.

## What are the components of a shape decoration?

* `ShapeBorder` is the base class of all shape outlines, including those used by box decorations; in essence, it describes a single shape with edges of defined width \(typically via `BorderSide`\). Shape borders can be interpolated and combined \(via the addition operator or `ShapeBorder.add`\). Additionally, borders may be scaled \(affecting properties like border width and radii\) and painted directly to a canvas \(via `ShapeBorder.paint`\); painting may be adjusted based on text direction. Paths describing the shape’s inner and outer edges may also be queried \(via `ShapeBorder.getInnerPath` and `ShapeBorder.getOuterPath`\).

## What are the components of a box decoration?

* `BoxBorder` is a subclass of `ShapeBorder` that is further specialized by `Border` and `BorderDirectional` \(the latter adding text direction sensitivity\). These instances describe a set of four borders corresponding to the cardinal directions; their precise arrangement is left undefined until rendering. Borders may be combined \(via `Border.merge`\) provided that all associated sides share a style and color. If so, the corresponding widths are added together.
  * Borders must be made concrete by providing a rectangle and, optionally, a `BoxShape`. The provided rectangle determines how the borders are actually rendered; uniform borders are more efficient ot paint.
* `BoxShape` describes how a box decoration \(or border\) is to be rendered into its bounds. If rectangular, painting is coincident with the bounds. If circular, the box is painted as a uniform circle with diameter matching the smaller of the bounding dimensions.

## What are the decoration building blocks?

* Decoration describes an adaptive collection of graphical effects that may be applied to an arbitrary rectangle \(e.g., box\). Decorations optionally specify a padding \(via `Decoration.padding`\) to ensure that any additional painting within a box \(e.g., from a child widget; note that decorations do not perform clipping\) does not overlap with the decoration’s own painting. Additionally, certain decorations can be marked as complex \(via `Decoration.isComplex`\) to enable caching.
  * Decorations support hit testing \(via `Decoration.hitTest`\). A size is provided so that the decoration may be scaled to a particular box. The given offset describes a position within this box relative to its top-left corner. An optional `TextDirection` supports containers that are sensitive to this parameter.
  * Decorations support linear interpolation \(via `Decoration.lerp`, `Decoration.lerpFrom`, and `Decoration.lerpTo`\). The “t” parameter represents a position on a timeline with 0 corresponding to 0% \(i.e., the pre-state\) and 1 corresponding to 100% \(i.e., the post-state\); note that values outside of this range are possible. If the source or destination value is null \(indicating that a true interpolation isn’t possible\), a default interpolation should be computed that reasonably approximates a true interpolation.
* `BoxDecoration` is a `Decoration` subclass that describes the appearance of a graphical box. Boxes are composed of a number of elements, including a border, a drop shadow, and a background. The background is itself comprised of color, gradient, and image layers. While typically rectangular, boxes may be given rounded corners or even a circular shape \(via `BoxDecoration.shape`\). `BoxDecorations` provide a `BoxPainter` subclass capable of rendering the described box given different `ImageConfigurations`.
* `ShapeDecoration` is analogous to `BoxDecoration` but supports rendering into any shape \(via `ShapeBorder`\). Rendering occurs in layers: first a fill color is painted, then a gradient, and finally an image. Next, the `ShapeBorder` is painted \(clipping the previous layers\); the border also serves as the casting element for all associated shadows. `ShapeDecoration` also uses a `BoxPainter` subclass for rendering.
  * Shape decorations may be obtained from box decorations \(via `ShapeDecoration.fromBoxDecoration`\) since the latter is derived from the former. In general, box decorations are more efficient since they do not need to represent arbitrary shapes; however, shapes support a wider arrange of interpolation \(e.g., rectangle to circle\).
* `DecoratedBox` incorporates a decoration into the widget hierarchy. Decorations can be painted in the foreground or background via `DecorationPosition` \(i.e., in front of or behind the child, respectively\). Generally, `Container` is used to incorporate a `DecoratedBox` into the UI.
* `NotchedShape` describes the difference of two shapes \(i.e., a guest shape is subtracted from a host shape\). A path describing this shape is obtained by specifying two bounding rectangles \(i.e., the host and the guest\) sharing a coordinate space. The `AutomaticNotchedShape` subclass uses these bounds to determine the concrete dimensions of `ShapeBorder` instances before computing their difference.

## How are decorations painted?

* `BoxPainter` provides a base class for instances capable of rendering a `Decoration` to a canvas given an `ImageConfiguration`. The configuration specifies the final size, scale, and locale to be used when rendering; this information allows an otherwise abstract decoration to be made concrete. Since decorations may rely on asynchronous image providers, `BoxPainter.onChanged` notifies client code when the associated resources have changed \(i.e, so that painting may be repeated\).

