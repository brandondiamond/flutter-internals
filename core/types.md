# Types


## What types are used to describe positions?

* `OffsetBase` represents a 2-dimensional, axis-aligned vector. Subclasses are immutable and comparable using standard operators.
* Offset is an `OffsetBase` subclass that may be understood as a point in cartesian space or a vector. Offsets may be manipulated algebraically using standard operators; the “&” operator allows a Rect to be constructed by combining the offset with a Size \(the offset identifies the rectangle’s top left corner\). Offsets may also be interpolated.
* Point is a dart class for representing a 2-dimensional point on the cartesian plane.

## What types are used to describe magnitudes?

* Size is an `OffsetBase` subclass that represents a width and a height. Geometrically, Size describes a rectangle with its top left corner coincident with the origin. Size includes a number of methods describing a rectangle with dimensions matching the current instance and a top left corner coincident with a specified offset. Sizes may be manipulated algebraically using standard operators; the “+” operator expands the size according to a provided delta \(via Offset\). Sizes may also be interpolated.
* Radius describes either a circular or elliptical radius. The radius is expressed as intersections of the x- and y-axes. Circular radii have identical values. Radii may be manipulated algebraically using standard operators and interpolated.

## What types are used to describe regions?

* Rect is an immutable, 2D, axis-aligned, floating-point rectangle whose coordinates are relative to a given origin. A rectangle can be described in various ways \(e.g., by its center, by a bounding circle, by offsets from its left, top, right, and bottom edges, etc\) or constructed by combining an Offset and a Size. Rectangles can be inflated, deflated, combined, intersected, translated, queried, and more. Rectangles can be compared for equality and interpolated.
* `RRect` augments a Rect with four independent radii corresponding to its corners. Rounded rectangles can be described in various ways \(e.g., by offsets to each of its sides and one or more radii, by a bounding box fully enclosing the rounded rectangle with one or more radii, etc\). Rounded rectangles define a number of sub-rectangles: a bounding rectangle \(`RRect.outerRect`\), a rectangle with identical sides and edges centered within the rounded corners \(`RRect.middleRect`\), tall and wide inner rectangles with height and width matching the rounded rectangle \(`RRect.tallMiddleRect`, `RRect.wideMiddleRect`\), and more. A rounded rectangle is said to describe a “stadium” if it possesses a side with no straight segment \(e.g., entirely drawn by the two rounded corners\). Rounded rectangles can be interpolated.

## What types are used to describe coordinate spaces?

* Axis represents the X- or Y-axis \(horizontal or vertical, respectively\) relative to a coordinate space. The coordinate space can be arbitrarily transformed and therefore need not be parallel to the screen’s edges.
* `AxisDirection` applies directionality to an axis. The value represents the spatial direction in which values increase along the axis, with the origin being rooted at the opposite end \(e.g., `AxisDirection.down` positions the origin at the top with positive values growing downward\).
* `GrowthDirection` is the direction of growth relative to the current axis direction \(e.g., how items are ordered along the axis\). `GrowthDirection.forward` implies an ordering consistent with the axis direction \(the first item is at the origin with subsequent items following\). `GrowthDirection.reverse` is exactly the opposite \(the last item is at the origin with preceding items following\).
  * Growth direction does not flip the meaning of “leading” and “trailing,” it merely determines how children are ordered along a specified axis.
  * For a viewport, the origin is positioned based on the axis direction \(e.g., `AxisDirection.down` positions the origin toward the top of the screen, `AxisDirection.up` positions the origin toward the bottom of the screen\), with the growth direction determining how children are ordered at the origin. As a result, both pieces of information are necessary to determine where a set of slivers should actually appear.
* `ScrollDirection` represents the user’s scroll direction relative to the positive scroll offset direction \(itself determined by axis direction and growth direction\). Includes an idle state \(`ScrollDirection.idle`\).
  * Confusingly, this refers to the direction the content is moving on screen rather than where the user is scrolling \(e.g., scrolling down a webpage causes the page’s contents to move upward; this would be classified as `ScrollDirection.reverse` since this motion is opposite the axis direction\).

## What types are used to describe graphics?

* Color is a 32-bit immutable quantity describing alpha, red, green, and blue color channels. Alpha can be defined using an opacity value from zero to one. Colors may be freely interpolated and converted into a luminance value.
* Shadow represents a single drop shadow with a color, an offset from the casting element, and a blur radius characterizing the gaussian blur applied to the shadow.
* Gradient describes one or more smooth color transitions. Gradients may be interpolated and scaled; gradients may also be used to obtain a corresponding shader. Linear, radial, and sweep gradients are supported \(via `LinearGradient`, `RadialGradient`, and `SweepGradient`, respectively\). `TileMode` determines how a gradient paints beyond its defined bounds \(a linear gradient defines a strip, a radial gradient defines a disc\). Gradients may be clamped \(e.g., hold their initial and final values\), repeated \(e.g., restarted at their bounds\), or mirrored \(e.g., restarted but with initial and final values alternating\).

