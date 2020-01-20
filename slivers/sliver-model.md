# Sliver `Model`


## What are the sliver building blocks?

* `RenderSliver` is the base class used by render objects that implement scrolling effects. Layout utilizes `SliverConstraints` to produce `SliverGeometry` \(`RenderSliver.geometry`\); painting is unchanged, with the origin positioned at the top-left corner. Hit testing is implemented similarly to `RenderBox`, relying instead on extents within the viewport’s main axis. 
* `RenderSliverHelpers` incorporates a variety of helpers, including support for implementing `RenderObject.applyPaintTransform` and `RenderObject.hitTestChildren` for slivers with box children.
* `RenderSliverSingleBoxAdapter`
* `RenderSliverToBoxAdapter` adapts a render box to the sliver protocol, allowing it to appear alongside and within other slivers.

## How do slivers manage children?

* Most slivers contain box children. Some, like `RenderSliverPadding`, have a single sliver child. Few slivers have multiple sliver children \(generally, this is the purview of viewports\).
* Some slivers manage children dynamically. This is quite complicated and discussed in a subsequent section.

## What helpers do slivers provide?

* `RenderSliver.calculateCacheOffset` and `RenderSliver.calculatePaintOffset` accept a region \(defined by a “from” and “to” relative to the sliver’s zero offset\) and determine the contained extent that is visible. This value is derived from the current scroll offset and remaining paint/cache extent from the provided constraints.
  * The result is only meaningful for slivers that paint exactly as much as is scrolled into view. For example, if a sliver paints more than it was scrolled \(e.g., a pinned sliver that always paints the same amount\), this calculation will be invalid.
* `RenderSliver.getAbsoluteSizeRelativeToOrigin`: the size of the sliver in relation to the sliver’s leading edge rather than the canvas’s origin. This size is derived from the cross axis and paint extents; the paint extent is negated if the sliver’s contents precede its leading edge from the perspective of the canvas.
* `RenderSliver.childScrollOffset`: the distance from the parent’s zero scroll offset to a child’s zero scroll offset; unaffected by scrolling. Defaults to returning zero \(i.e., child and parent are aligned\).
* `RenderSliver.childMainAxisPosition`: the distance from the parent’s visible leading edge to the child’s visible leading edge, in the axis direction. If the actual leading edge is not visible, the edge defined by the intersection of the sliver with the viewport’s leading edge is used instead. The child must be visible, though the parent need not be \(e.g., a persistent header that pins its child even when the anchoring sliver is scrolled away\).
  * Slivers that contain box children cannot measure distance using the child’s visible edge since box children are viewport naive. As a result, the box’s actual leading edge is always used.
* `RenderSliver.childCrossAxisPosition`: for vertical axes, this is the distance from the parent’s left side to the child’s left side. For horizontal axes, this is the distance from parent’s top side to the child’s top side.

## What new properties do slivers introduce?

* `RenderSliver.geometry` is analogous to `RenderBox.size` and contains the output of layout.
* `RenderSliver.centerOffsetAdjustment` is an offset applied to the viewport’s center sliver \(e.g., to center it\). Implicitly shifts neighboring slivers. Positive offsets shift the sliver opposite the axis direction.

## How is sliver parent data managed?

* There are two options for encoding parent data \(i.e., a sliver’s position within its parent\): using absolute coordinates from the parent’s visible top left corner \(via `SliverPhysicalParentData.paintOffset`\) or using a delta from the parent’s zero scroll offset to the nearest child edge \(via `SliverLogicalParentData.layoutOffset`\). The former optimizes painting at the expense of layout whereas the latter makes the opposite trade-off.
  * When thinking about paint offsets, it’s crucial to note the distinction between a sliver’s top-left corner and its visible top-left corner. If a sliver straddles the leading edge in a downward-scrollable viewport, its children are positioned relative to the top-left corner of the viewport, not the top-left corner that precedes the leading edge. This is always the edge that is nearest the top-left corner regardless of axis or growth direction.
  * Logical offsets are relative to the parent’s zero scroll offset. This is either the parent’s leading or trailing edge, depending on growth direction.

## How do slivers perform hit testing?

* Slivers follow a similar model to `RenderBox`, tracking whether a point is contained within the current sliver or one of its descendents. The current sliver determines whether this is the case by first testing its children and then itself \(`RenderSliver.hitTest` invokes `RenderSliver.hitTestChildren` then `RenderSliver.hitTestSelf`\).
* If a hit is detected, the sliver adds a `SliverHitTestEntry` to the `SliverHitTestResult`, and returns true. The sliver will then receive subsequent pointer events via `RenderSliver.handleEvent`.
  * `SliverHitTestEntry` associates a sliver with the coordinate that was hit.
  * `SliverHitTestResult` is a `HitTestResult` with a convenience method for mapping to sliver coordinates \(`SliverHitTestResult.addWithAxisOffset`\). This adapts a main and cross axis position in the parent’s coordinate space to the child’s coordinate space by subtracting the offset to the child. A paint offset is also used since `HitTestResult` is also responsible for mapping pointer events, which use canvas coordinates. \[?\]
* Hit testing will only be performed once layout has completed; painting will not have occurred yet.
* Coordinates are expressed as main and cross axis positions in the sliver’s own coordinate space. Any positions provided to children must be mapped into that child’s coordinate space.
  * Positions are relative to the current scroll offset; that is, a main axis position of zero corresponds to the first visible offset within the sliver, not the leading edge of the sliver.
  * The interactive region is defined by the current hit test extent anchored at the current scroll offset. The sliver’s full cross axis extent is considered interactive, unless the default hit testing behavior is overridden.
  * Slivers may paint anywhere in the viewport using a non-zero paint origin. Special care must be taken to transform coordinates if this is the case. For instance, a sliver that contains a sliver child that always paints a button at the viewport’s trailing edge will filter out events that lie outside of its own interactive bounds unless it is explicitly written to handle its child’s special painting behavior.

## How do slivers represent layout constraints?

* Sliver constraints can be converted to box constraints via `SliverConstraints.asBoxConstraints`. This method accepts minimum and maximum bounds for the sliver’s main extent \(e.g., if the main axis is horizontal, this would correspond to width\). The cross axis constraint is always tight.
* Constraints provide information about scrolling state from the perspective of the sliver being laid out so that it can select appropriate geometry.
  * `SliverConstraints.axis`, `SliverConstraints.axisDirection`, `SliverConstraints.crossAxisDirection`: the current orientation and directionality of the main and cross axes.
  * `SliverConstraints.growthDirection`: how a sliver’s contents are ordered relative to the main axis direction \(i.e., first-to-last vs. last-to-first\).
  * `SliverConstraints.normalizedGrowthDirection`: the growth direction that would produce the same child ordering assuming a “standard” axis direction \(`AxisDirection.down` or `AxisDirection.right`\). This is the opposite of `SliverConstraints.growthDirection` if axis direction is up or left.
  * `SliverConstraints.cacheOrigin`: how far before the current scroll offset to begin painting to support caching \(i.e., the number of pixels to “reach back” from the first visible offset in the sliver\). Always between -`SliverConstraints.scrollOffset` and zero.
  * `SliverConstraints.overlap`: a delta from the current scroll offset to the furthest pixel that has not yet been painted by an earlier sliver, in the axis direction. For example, if an earlier sliver painted beyond its layout extent, this would be the offset to “escape” that painting. May be negative if the furthest pixel painted precedes the current sliver \(e.g., due to overscroll\).
  * `SliverConstraints.precedingScrollExtent`: the total scrolling distance consumed by preceding slivers \(i.e., the sum of all preceding scroll extents\). Scroll extent is approximate for slivers that could not fully paint in the available space; thus, this value may be approximate. If a preceding sliver is built lazily \(e.g., a list that builds children on demand\) and has not finished building, this value will be infinite. It may also be infinite if a preceding sliver is infinitely scrollable.
  * `SliverConstraints.remainingPaintExtent`: the number of pixels available for painting in the viewport. May be infinite \(e.g., for shrink-wrapping viewports\). Zero when all pixels have been painted \(e.g., when a sliver is beyond the viewport’s trailing edge\).
    * Whether this extent is actually consumable by the sliver depends on the sliver’s intended behavior.
  * `SliverConstraints.remainingCacheExtent`: the number of pixels available for painting in the viewport and cache regions, starting from the scroll offset adjusted by the cache origin. Always larger than remaining paint extent.
    * Whether this extent is actually consumable by the sliver depends on the sliver’s intended behavior.
  * `SliverConstraints.scrollOffset`: the offset of the first visible part of the sliver, as measured from the sliver’s leading edge in the axis direction. Equivalently, this is how far the sliver’s leading edge precedes its parent’s leading edge.
    * All slivers after the leading edge have a scroll offset of zero, including those after the trailing edge.
    * The scroll offset may be larger than the sliver’s actual extent if the sliver is far before the parent’s leading edge.
  * `SliverConstraints.userScrollDirection`: the `ScrollDirection` in relation to the axis direction and the content’s logical ordering \(as determined by growth direction\).
    * Note that this describes the direction in which the content is actually moving on the user’s screen, not the intuitive direction that the user is scrolling \(e.g., scrolling down a webpage translates the content up; thus, the scroll direction would be `ScrollDirection.reverse`. If the content were reversed within that web page, it would be `ScrollDirection.forward`\).
  * `SliverConstraints.viewportMainAxisExtent`: the number of pixels that the viewport can display in its main axis.

## How do slivers represent geometry?

* Slivers define several extents \(i.e., regions within the viewport\) with scroll, layout, and paint being particularly important. Scroll extent corresponds to the amount of scrolling necessary to go from the sliver’s leading edge to its trailing edge. Layout extent corresponds to the number of pixels physically occupied by the sliver in the visible viewport. Paint extent corresponds to the number of pixels painted on by the sliver in the visible viewport. Paint extent and layout extent typically go to zero as the sliver is scrolled out of the viewport, whereas scroll extent generally remains fixed.
* A sliver’s actual position within its parent \(sometimes called its “layout position”\) is encoded in one of two ways: as an absolute paint offset that can be directly used when painting or as a layout offset relative to the parent’s zero scroll offset \(`SliverPhysicalParentData` and `SliverLogicalParentData`, respectively\). Though a sliver may have a particular position, it is free to paint anywhere in the viewport.
* Sliver geometry captures several such extents as well as the sliver’s current relationship with the viewport.
  * `SliverGeometry.paintOrigin`: where to begin painting relative to the sliver’s current layout position, expressed as an offset in the axis direction \(e.g., a negative value would precede the current position\).
    * The sliver still consumes layout extent at its original layout position. Subsequent layout is unaffected by the paint origin.
    * Taken into account when determining the next sliver’s overlap constraint, particularly if this sliver paints further than any preceding sliver.
  * `SliverGeometry.paintExtent`: the number of contiguous pixels consumed by this sliver in the visible region. Measured from the sliver’s layout position, as adjusted by the paint origin \(`SliverGeometry.paintOrigin`, typically zero\).
    * Paint extent ≤ remaining paint extent: it is inefficient to paint out of bounds.
    * Typically, but not necessarily, goes to zero when scrolled out of the viewport.
    * The paint extent combined with the paint origin determines the next sliver’s overlap constraint.
  * `SliverGeometry.maxPaintExtent`: the number of pixels this sliver could paint in an unconstrained environment \(i.e., a shrink-wrapping viewport\). Approximate.
  * `SliverGeometry.layoutExtent`: the number of contiguous pixels consumed by this sliver in the visible region for layout \(i.e., the amount of space this sliver will physically occupy in the viewport\). Measured from the sliver’s layout position regardless of paint origin. Layout positions are advanced by summing layout extents.
    * Layout extent ≤ paint extent: cannot take up more space than is painted.
    * Defaults to paint extent. 
    * Typically, but not necessarily, goes to zero when scrolled out of the viewport. Can still take up space even when logically offscreen.
  * `SliverGeometry.scrollExtent`: the amount of scrolling needed to reach this sliver’s trailing edge from its leading edge. Used to calculate all subsequent sliver’s scroll offsets.
    * Approximate, except when sliver is able to fully paint.
    * Typically, but not necessarily, constant.
    * Typically, but not necessarily, an upper bound on paint extent.
  * `SliverGeometry.cacheExtent`: the number of contiguous pixels consumed by this sliver in the cacheable region. Measured from the sliver’s layout position, as adjusted by the cache origin \(i.e., the offset into the available cache region from the sliver’s current scroll offset\).
    * Layout extent ≤ paint extent ≤ cache extent: the cacheable viewport includes the visible region; thus, any visible pixels consumed count toward the sliver’s cache extent.
    * Slivers that paint at positions other than their layout position will need to calculate how much of the cache region was actually consumed.
  * `SliverGeometry.hitTestExtent`: the number of contiguous pixels painted by this sliver that are interactive in the visible region. Measured from the sliver’s layout position. Does not take the paint origin into account.
    * Hit test extent ≤ paint extent: can only handle events in painted region.
    * Defaults to paint extent.
    * Typically, but not necessarily, goes to zero when scrolled out of the viewport.
    * Subject to interpretation by `RenderSliver.hitTest` which might, for instance, take into account paint origin; if applicable, the parent sliver must be written to forward hit events that fall outside of its own interactive bounds.
  * `SliverGeometry.maxScrollObstructionExtent`: only meaningful for slivers that affix themselves to the leading or trailing edges of the viewport \(e.g., a pinned header\), effectively reducing the amount of content visible within the viewport \(i.e., the viewport’s outer dimension\). If applicable, this is the extent of that reduction, in pixels.
  * `SliverGeometry.hasVisualOverflow`: whether a clip is necessary due to visual overflow. The viewport will add a clip if any of its children report visual overflow.
  * `SliverGeometry.visible`: whether this sliver should be painted. Defaults to true when paint extent is greater than zero.
  * `SliverGeometry.scrollOffsetCorrection`: a correction indicating that the incoming scroll offset was invalid \(e.g., \[?\]\) and therefore must be corrected. If set, no further geometry need be returned as layout will be recomputed from scratch. Must be propagated to the containing viewport.

## How is a box embedded within a sliver?

* `SliverToBoxAdapter` is a single-child render object widget that incorporates a `RenderSliverToBoxAdapter` into the render tree.
* `RenderSliverSingleBoxAdapter` lays the groundwork for managing a single render box child.
  * Children are positioned using physical coordinates \(`SliverPhysicalParentData`\); these coordinates are measured from the parent’s visible top-left corner to the child’s visible top-left corner.
  * It provides a hook for updating the child’s position using the parent’s constraints and geometry \(`RenderSliverSingleBoxAdapter.setChildParentData`\). This calculation takes into account that render boxes may be partially off screen and must therefore be offset during painting to ensure that the correct slice of the box is visible.
    * For instance, assuming a typical downward-oriented viewport, a non-zero scroll offset indicates how much of the parent is currently above the top of the viewport. To paint the correct slice of the viewport, the render box must be translated above the canvas by that same amount.
    * To crystallize this, consider an upward-oriented viewport. The child’s paint offset is zero as its parent is scrolled into view \(the box’s visible top-left corner is coincident with its parent’s visible top-left corner\). Once the sliver has been fully scrolled across the viewport and begins to scroll off the viewport’s top edge, its parent’s first visible top-left corner effectively becomes the top of the viewport and remains there as scrolling continues \(i.e., this is now the top-left visible corner of the sliver\). Therefore, to correctly paint the box shifting out of view, it must be offset to account for the shifted frame of reference.
  * The child’s main axis position is calculated using the box’s actual edge since render box is viewport naive. Thus, the box’s main axis position may be negative if it precedes the viewport’s leading edge.
  * Hit testing and painting trivially wrap the box’s own implementation.
* `RenderSliverToBoxAdapter` extends `RenderSliverSingleBoxAdapter` and does the actual work of adapting the sliver layout protocol to the render box equivalent.
  * Boxes receive tight cross axis constraints and unbounded main axis constraints \(i.e., the box is free to select its main dimension without constraint\). 
  * This extent serves as the sliver’s scroll extent and max paint extent \(i.e., because the size of the box is precisely how much this sliver can scroll and paint\).
  * The adapter calculates how much of the desired extent would be visible given the sliver’s current scroll offset and the remaining space in the viewport \(via `RenderSliver.calculatePaintOffset`\).
  * This quantity is used as the paint, layout, and hit-testing extents \(i.e., because this is how much of the box is visible, consuming space, and interactive, respectively\).
  * If the box cannot be rendered in full \(i.e., there isn’t enough room in the viewport or the sliver isn’t past the leading edge\), a clip is applied by indicating that there is visual overflow.

## What are some examples of basic slivers?

* `RenderSliverPadding`: adds padding around a child sliver.
  * Lays out child with the incoming constraints modified to account for padding.
    * `SliverGeometry.scrollExtent`: decreased to reflect leading padding pushing content toward the viewport’s leading edge.
    * `SliverGeometry.cacheOrigin`: offset to reflect padding consuming cache space.
    * `SliverGeometry.remainingPaintExtent`, `SliverGeometry.remainingCacheExtent`: reduced by any leading padding that is visible.
    * `SliverGeometry.crossAxisExtent`: reduced by the total cross axis padding.
  * Calculate the padding sliver’s geometry using the child’s geometry.
    * `SliverGeometry.paintExtent`: either the total visible main axis padding plus the child’s layout extent or, if larger, the leading padding plus the child’s paint extent \(this would allow the child’s painting to overflow trailing padding\). Clamped to remaining paint extent.
    * `SliverGeometry.scrollExtent`:the child’s scroll extent plus main axis padding.
    * `SliverGeometry.layoutExtent`: the total visible main axis padding plus the child’s layout extent. Clamped to the paint extent \(i.e., cannot consume more space than is painted\).
    * `SliverGeometry.hitTestExtent`: the total visible main axis padding plus the child’s paint extent or, if larger, the leading padding plus the child’s hit testing extent \(allowing the child’s hit testing region to overflow\).
  * Finally, a position is selected according to axis direction, growth direction, and the leading padding that is currently visible.
* `RenderSliverFillRemaining` is a single box adapter that causes its child to fill the remaining space in the viewport. Subsequent slivers are never seen as this sliver consumes all usable space.
  * This sliver calculates the remaining paint extent in the main axis, ensuring that this is passed as a tight constraint to the box child \(unlike `RenderSliverToBoxAdapter` which leaves the main extent unconstrained\). The child is subsequently laid out.
  * The sliver then calculates its own geometry.
    * `SliverGeometry.scrollExtent`: the full extent of the viewport. This is an upper bound since it isn’t actually possible to scroll beyond this sliver: it consumes all viewport space.
    * `SliverGeometry.paintExtent`, `SliverGeometry.layoutExtent`: matches the extent of the child that fits within the viewport.
  * Finally, the box is positioned \(using the incoming constraints and calculated geometry\) such that it is correctly offset during painting if it cannot fit in the viewport.

