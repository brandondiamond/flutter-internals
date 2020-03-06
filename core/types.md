# Types

## What types are used to describe positions?

* `OffsetBase` represents a 2-dimensional \(2D\), axis-aligned vector. Subclasses are immutable and comparable using standard operators.
* `Offset` is an `OffsetBase` subclass that may be understood as a point in cartesian space or a vector. Offsets may be manipulated algebraically using standard operators; the `&` operator allows a `Rect` to be constructed by combining the offset with a `Size` \(the offset identifies the rectangle’s top left corner\). Offsets can be interpolated.
* `Point` is a dart class for representing a 2D point on the cartesian plane.

## What types are used to describe magnitudes?

* `Size` is an `OffsetBase` subclass that represents a width and a height. Geometrically, `Size` describes a rectangle with its top left corner coincident with the origin. `Size` includes a number of methods describing a rectangle with dimensions matching the current instance and a top left corner coincident with a specified offset. Sizes may be manipulated algebraically using standard operators; the `+` operator expands the size according to a provided delta \(via `Offset`\). Sizes can be interpolated.
* `Radius` describes either a circular or elliptical radius. The radius is expressed as intersections of the x- and y-axes. Circular radii have identical values. Radii may be manipulated algebraically using standard operators and interpolated.

## What types are used to describe regions?

* `Rect` is an immutable, 2D, axis-aligned, floating-point rectangle whose coordinates are relative to a given origin. A rectangle can be described in various ways \(e.g., by its center, by a bounding circle, by the magnitude of its left, top, right, and bottom edges, etc.\) or constructed by combining an `Offset` and a `Size`. Rectangles can be inflated, deflated, combined, intersected, translated, queried, and more. Rectangles can be compared for equality and interpolated.
* `RRect` augments a `Rect` with four independent radii \(via `Radius`\) corresponding to its corners. Rounded rectangles can be described in various ways \(e.g., by offsets to each of its sides and one or more radii, by a bounding box fully enclosing the rounded rectangle with one or more radii, etc.\). Rounded rectangles define a number of sub-rectangles: a bounding rectangle \(`RRect.outerRect`\), an inner rectangle with left and right edges matching the base rectangle and top and bottom edges inset to coincide with the rounded corners' centers \(`RRect.wideMiddleRect`\), a similar rectangle but with the sets reversed \(`RRect.tallMiddleRect`\), and a rectangle that is the intersection of these two \(`RRect.middleRect`\). A rounded rectangle is said to describe a “stadium” if it possesses a side with no straight segment \(e.g., entirely drawn by the two rounded corners\). Rounded rectangles can be interpolated.

## What types are used to describe coordinate spaces?

* `Axis` represents the X- or Y-axis \(horizontal or vertical, respectively\) relative to a coordinate space. The coordinate space can be arbitrarily transformed and therefore need not be parallel to the screen’s edges.
* `AxisDirection` applies directionality to an axis. The value represents the direction in which values increase along an associated axis, with the origin rooted at the opposite end \(e.g., `AxisDirection.down` positions the origin at the top with positive values growing downward\).
* `GrowthDirection` is the direction of growth relative to the current axis direction \(e.g., how items are ordered along the axis\). `GrowthDirection.forward` implies an ordering consistent with the axis direction \(the first item is at the origin with subsequent items following\). `GrowthDirection.reverse` is exactly the opposite \(the last item is at the origin with preceding items following\).
  * Growth direction does not flip the meaning of “leading” and “trailing,” it merely determines how children are ordered along a specified axis.
  * For a viewport, the origin is positioned according to an axis direction \(e.g., `AxisDirection.down` positions the origin at the top of the screen, `AxisDirection.up` positions the origin at the bottom of the screen\), with the growth direction determining how children are ordered starting from the origin. As a result, both pieces of information are necessary to determine where a set of slivers should actually appear.
* `ScrollDirection` represents the user’s scroll direction relative to the positive scroll offset direction \(i.e., the direction in which positive scroll offsets increase as determined by axis direction and growth direction\). Includes an idle state \(`ScrollDirection.idle`\).
  * This is typically subject to the growth direction \(e.g., the scroll direction is flipped when growth is reversed\).
  * Confusingly, this refers to the direction the content is moving on screen rather than where the user is scrolling \(e.g., scrolling down a web page causes the page’s contents to move upward; this would be classified as `ScrollDirection.reverse` since this motion is opposite the axis direction\).

## What types are used to describe graphics?

* `Color` is a 32-bit immutable quantity describing alpha, red, green, and blue color channels. Alpha can be defined using an opacity value from zero to one. Colors can be interpolated and converted into a luminance value.
* `Shadow` represents a single drop shadow with a color, an offset from the casting element, and a blur radius characterizing the Gaussian blur applied to the shadow.
* `Gradient` describes one or more smooth color transitions. Gradients can be interpolated and scaled; gradients can also be used to obtain a reference to a corresponding shader. Linear, radial, and sweep gradients are supported \(via `LinearGradient`, `RadialGradient`, and `SweepGradient`, respectively\). `TileMode` determines how a gradient paints beyond its defined bounds. Gradients may be clamped \(e.g., hold their initial and final values\), repeated \(e.g., restarted at their bounds\), or mirrored \(e.g., restarted but with initial and final values alternating\).

## How are tree nodes modeled?

* `AbstractNode` represents a node in a tree without specifying a particular child model \(i.e., the tree's actual structure is left as an implementation detail\). Concrete implementations must call `AbstractNode.adoptChild` and `AbstractNode.dropChild` whenever the child model changes.
  * `AbstractNode.owner` references an arbitrary object shared by all nodes in a subtree.
    * `AbstractNode.attach` assigns an owner to the node. Adopting children will attach them automatically. Used by the owner to attach the tree via its root node.
      * Subclasses should attach all children since the parent can change its attachment state at any time and must keep its children in sync.
    * `AbstractNode.detach` clears a node's owner. Dropping children will detach them automatically. Used by the owner to detach the tree via its root node.
      * Subclasses should detach all children since the parent can change its attachment state at any time and must keep its children in sync.
    * `AbstractNode.attached` indicates whether the node is attached \(i.e., has an owner\).
  * `AbstractNode.parent` references the parent abstract node.
    * `AbstractNode.adoptChild` updates a child's parent and depth. The child is attached if the parent has an owner.
    * `AbstractNode.dropChild` clears the child's parent. The child is detached if the parent has an owner.
  * `AbstractNode.depth` is an integer that increases with depth. All depths below a given node will be greater than that node's depth. Note that values need not match the actual depth of the node.
    * `AbstractNode.redepthChild` updates a child's depth to be greater than its parent.
    * `AbstractNode.redepthChildren` uses the concrete child model to call `AbstractNode.redepthChild` on each child.

