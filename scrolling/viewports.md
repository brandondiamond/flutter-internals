# Viewports


## What are the general viewport building blocks?

* `ViewportOffset` is an interface that tracks the current scroll offset \(`ViewportOffset.pixels`\) and direction \(`ViewportOffset.userScrollDirection`, which is relative to the positive axis direction, ignoring growth direction\); it also offers a variety of helpers \(e.g., `ViewportOffset.animateTo`\). The offset represents how much content has been scrolled off screen, or more precisely, the number of logical pixels by which all children have been shifted opposite the viewport’s scrolling direction \(`Viewport.axisDirection`\). For a web page, this would be how many pixels of content are above the browser’s viewport. This interface is implemented by `ScrollPosition`, tying together viewports and scrollables. Pixels can be negative when scrolling before the center sliver. \[?\]

## What are the viewport widget building blocks?

* Viewport is a layout widget that is larger on the inside. The viewport is associated with a scroll offset \(`ViewportOffset`\), an interface that is implemented by `ScrollPosition` and typically fulfilled by `ScrollPositionWithSingleContext`. As the user scrolls, this offset is propagated to descendant slivers via layout. Finally, slivers are repainted at their new offsets, creating the visual effect of scrolling.
* `ShrinkWrappingViewport` is a variant of viewport that sizes itself to match its children in the main axis \(instead of expanding to fill the main axis\).
* `NestedScrollViewViewport` is a specialized viewport used by `NestedScrollView` to coordinate scrolling across two viewports \(supported by auxiliary widgets like `SliverOverlapAbsorberHandle`\).
* `ScrollView` couples a viewport with a scrollable, deferring to its subclass to provide slivers; as such, the `ScrollView` provides a foundation for building scrollable UI. `CustomScrollView` accepts an arbitrary sliver list whereas `BoxScrollView` -- and its subclasses `ListView` and `GridView` -- apply a single layout model \(e.g., list or grid\) to a collection of slivers.

## What are the viewport rendering building blocks?

* `RenderAbstractViewport` provides a common interface for all viewport subtypes. This allows the framework to locate and interact with viewports in a generic way \(via `RenderAbstractViewport.of`\). It also provides a generic interface for determining offsets necessary to reveal certain children.
* `RenderViewportBase` provides shared code for render objects that host slivers. By establishing an axis and axis direction, `RenderViewportBase` maps the offset-based coordinate space used by slivers into cartesian space according to a managed offset value \(`ViewportOffset.pixels`\). `RenderViewportBase.layoutChildSequence` serves as the foundation for sliver layout \(and is typically invoked by `performLayout` in subclasses\). `RenderViewportBase` also establishes the cache extent \(the area to either side of the viewport that is laid out but not visible\) as well as entry points for hit testing and painting.
* `RenderViewport` displays a subset of its sliver children based on its current viewport offset. A center sliver \(`RenderViewport.center`\) is anchored at offset zero. Slivers before center \(“reverse children”\) grow opposite the axis direction \(`GrowthDirection.reverse`\) whereas the center along with subsequent slivers \(“forward children”\) grow forward \(`GrowthDirection.forward`\); both groups are anchored according to the same axis direction \(this is why both start from the same edge\), though conceptually reverse slivers are laid out in the opposite axis direction \(e.g., their “leading” and “trailing” edges are flipped\).
  * The anchor point can be adjusted, changing the visual position of offset zero \(`RenderViewport.anchor` is in the range \[0, 1\], with zero corresponding to the axis origin \[?\]\).
  * Conceptually, children are ordered: RN-R0, center, FN-F0.
* `RenderShrinkWrappingViewport` similar to `RenderViewport` except sized to match the total extent of visible children within the bounds of incoming constraints.
* `RenderNestedScrollViewViewport`

## What are the attributes of a viewport?

* Throughout this section, words like “main extent,” “cross extent,” “before,” “after,” “leading,” and “trailing” are used to eliminate spatial bias from descriptions. This is because viewports can be oriented along either axis \(e.g., horizontal, vertical\) with varying directionality \(e.g., down, right\). Moreover, the ordering of children along the axis is subject to the current growth direction \(e.g., forward or reverse\). 
* Viewports have two sets of dimensions: outer and inner. The portion of the viewport that occupies space on screen has a main axis and cross axis extent \(e.g., height and width\); these are the viewport’s “outer”, “outside”, or “physical” dimensions. The inside of the viewport, which matches or exceeds the outer extent, includes all the content contained within the viewport; these are described using “inner”, “inside”, or “logical” dimensions. The inner edges correspond to the edges of the viewport’s contents; the outer edges correspond to the edges of the visible content. When otherwise unspecified, the viewport’s leading / trailing edges generally refer to its outer \(i.e., physical\) edges.
* The viewport is comprised of a bidirectional list of slivers. “Forward slivers” include a “center sliver,” and are laid out in the default axis direction \(`GrowthDirection.forward`\). “Reverse slivers” immediately precede the center sliver and are laid out opposite the axis direction \(`GrowthDirection.reverse`\). The line between forward and reverse slivers, at the center sliver’s leading edge, is called the “centerline,” coincident with the zero scroll offset. Within this document, the region within the viewport comprised of reverse slivers is called the “reverse region” with its counterpart being the “forward region.” Note that the viewport’s inner edges fully encompass both regions.
* The viewport positions the center sliver at scroll offset zero by definition. However, the viewport also tracks a current scroll offset \(`RenderViewport.offset`\) to determine which of its sliver children are in view and therefore should be rendered. This offset represents the distance between the center sliver’s leading edge \(i.e., scroll offset zero\) and the viewport’s outer leading edge, and increases opposite the axis direction.
  * Equivalently, this represents the number of pixels by which the viewport’s contents have been shifted opposite its axis direction.
  * For example, if the center sliver’s leading edge is aligned with the viewport’s leading edge, the offset would be zero. If its trailing edge is aligned with the viewport’s leading edge, the offset would be the sliver’s extent. If its leading edge is aligned with the viewport’s trailing edge, the offset would be the viewport’s extent, negated.

## How do viewports manage parent data?

* There are two major classes of parent data used by slivers: `SliverPhysicalParentData` \(used by `RenderViewport`\) and `SliverLogicalParentData` \(used by `RenderShrinkWrappingViewport`\). These differ in how they represent the child’s position. The former stores absolute coordinates from the parent’s visible top left corner whereas the latter stores the distance from the parent’s zero scroll offset to the child’s nearest edge. Physical coordinates are more efficient for children that must repaint often but incur a cost during layout. Logical coordinates optimize layout at the expense of added cost during painting.
* Viewports use two subclasses to support multiple sliver children, `SliverPhysicalContainerParentData` and `SliverLogicalContainerParentData`. These are identical to their superclasses \(where the “parent” isn’t a sliver but the viewport itself\), mixing in `ContainerParentDataMixin`&lt;`RenderSliver`&gt;.

## When might the center sliver not appear at the leading edge?

* The center sliver may be offset by `RenderSliver.centerOffsetAdjustment` \(added to the current `ViewportOffset.pixels` value\). This effectively shifts the zero scroll offset \(e.g., to visually center the center sliver\).
* The zero scroll offset can itself be shifted by a proportion of the viewport’s main extent via `RenderViewport.anchor`. Zero positions the zero offset at the viewport’s leading edge; one positions the offset at the trailing edge \(and `0.5` would position it at the midpoint\).
* These adjustments are mixed into the calculation early on \(see `RenderViewport.performLayout` and `RenderViewport._attemptLayout`\). Conceptually, it is easiest to ignore them other than to know that they shift the centerline’s visual position.
* The center sliver may also paint itself at an arbitrary offset via `SliverGeometry.paintOrigin`, though this won’t actually move the zero offset.

## What are some of the quirks of viewport layout?

* Forward and reverse slivers are laid out separately and are generally isolated from one another \[?\]. Reverse slivers are laid out first, then forward slivers. Reverse slivers and forward slivers share the same axis direction \(i.e., the generated constraints reference the same `SliverConstraints.axisDirection`\), though reverse sliver calculations \(e.g., for painting or layout offset\) effectively flip this direction. Thus, it is most intuitive to think of reverse slivers as having their leading and trailing edges flipped, etc.
* Layout offset is effectively measured from the viewport’s outer leading edge to the nearest edge of the sliver \(i.e., the offset is relative to the viewport’s current view\).
  * More accurately, a sliver’s layout offset is measured from the zero scroll offset of its parent which, for a viewport, coincides with the centerline. However, since layout offset is iteratively computed by summing layout extents \(in `RenderViewportBase.layoutChildSequence`\) and these extents are zero unless a sliver is visible, this formal definition boils down to the practical definition described above. \[?\]
  * This property explains why `RenderViewportBase.computeAbsolutePaintOffset` is able to produce paint offsets trivially from layout offsets \(this is surprising since layout offsets are ostensibly measured from the zero scroll offset whereas paint offsets are measured from the box’s top left corner\).
  * Even though layout offsets after the trailing edge are approximate \(due to an implementation detail of `RenderViewportBase.layoutChildSequence`\), this direct mapping remains safe as out-of-bounds painting will be clipped.
* Scroll offset can be interpreted in two ways. 
  * When considering a sliver or viewport in isolation, scroll offset refers to a one dimensional coordinate space anchored at the object’s leading edge and extending toward its trailing edge. 
  * When considering a sliver in relation to a parent container, scroll offset represents the first offset in the sliver’s coordinate space that would be visible in its parent \(e.g. offset zero implies that the leading edge of the sliver is visible; offset N implies that all except the leading N pixels are visible -- if N is greater than the sliver’s extent, some of those pixels are empty space\). Conceptually, a scroll offset represents how far a sliver’s leading precedes the viewport.
    * When zero, the sliver has been fully scrolled after the leading edge \(and possibly after the trailing edge\).
    * When less than the sliver’s scroll extent, a portion of the sliver precedes the leading edge.
    * When greater, the sliver is entirely before the leading edge.
* Scroll extent represents how much space a sliver might consume; it need only be accurate when the constraints would permit the sliver to fully paint \(i.e., the desired paint extent fits within the remaining paintable space\).
  * Slivers preceding the leading edge or appearing within the viewport must provide valid scroll extents if they might conceivably be painted.
  * Slivers beyond the trailing edge may approximate their scroll extents since no pixels remain for painting.
* Overlap is the pixel offset \(in the main axis direction\) necessary to fully “escape” any earlier sliver’s painting. More formally, this is the distance from the sliver’s current position to the first pixel that hasn’t been painted on by an earlier sliver. Typically, this pixel is after the sliver’s offset \(e.g., because a preceding sliver painted beyond its layout extent\). However, in some cases, the overlap can be negative indicating that the first such pixel is before the sliver’s offset \(i.e., earlier than the sliver’s offset\).
* All slivers preceding the viewport’s trailing edge receive unclamped values for the remaining paintable and cacheable extents, even if those slivers are located far offscreen. Slivers implement a variety of layout effects and therefore may consume visible \(or cacheable\) pixels at their discretion.

## What other services does the viewport provide?

* Viewports support the concept of maximum scroll obstruction \(`RenderViewportBase.maxScrollObstructionExtentBefore`\), a region of the viewport that is covered by “pinned” slivers and that effectively reduces the viewport’s scrollable bounds. This is a secondary concept used only when computing the scroll offset to reveal a certain sliver \(`AbstractRenderViewport.getOffsetToReveal`\).
* Viewports provide a mechanism for querying the paint and \(approximate\) scroll offsets of their children. The implementation depends on the type of viewport; efficiency may also be affected by the parent model \(e.g., `RenderViewport` uses `SliverPhysicalContainerParentData`, allowing paint offsets to be returned immediately\).

## How are viewport children ordered?

* A logical index is assigned to all children. The center child is assigned zero; subsequent children \(forward slivers\) are assigned increasing indices \(e.g., 1, 2, 3\) whereas preceding children \(reverse slivers\) are assigned decreasing indices \(e.g., -1, -2, -3\).
* Children are stored sequentially \(R reverse slivers + the center sliver + F forward slivers\), starting with the “last” reverse sliver \(-R\), proceeding toward the “first” reverse sliver \(-1\), then the center sliver \(0\), then ending on the last forward sliver \(F+1\).
  * The first child \(`RenderViewport.firstChild`\) is the “last” reverse sliver.
* Viewports define a painting order and a hit-test order. Reverse slivers are painted from last to first \(-1\), then forward slivers are painted from last to first \(center\). Hit-testing proceeds in the opposite direction: forward slivers are tested from the first \(center\) to the last, then reverse slivers are tested from the first \(-1\) to the last.

## What do shrink-wrapping viewports do differently?

* Whereas an ordinary viewport expands to fill the main axis, shrink-wrapping viewports are sized to minimally contain their children. Consequently, as the viewport is scrolled, its size will change to accommodate the visible children which may require layout to be repeated. Shrink-wrapping viewports do not support reverse children.
* The shrink-wrapping viewport uses logical coordinates instead of physical coordinates since it performs layout frequently.

## What is a scroll offset correction?

* A pixel offset directly applied to the `ViewportOffset.pixels` value allowing descendent slivers to adjust the overall scrolling position. This is done to account for errors when estimating overall scroll extent for slivers that build content dynamically.
  * Such slivers often cannot measure their actual extent without building and laying out completely first. Doing this continuously would be prohibitively slow and thus relative positions are used \(i.e., position is reckoned based on where neighbors are without necessarily laying out all children\).
* The scroll offset correct immediately halts layout, propagating to the nearest enclosing viewport. The value is added directly to the viewport’s current offset \(e.g., a positive correction increases `ViewportOffset.pixels`, translating content opposite the viewport’s axis direction -- i.e., scrolling forward\).
  * Conceptually, this allows slivers to address logical inconsistencies that arise due to, e.g., estimating child positions by determining a scroll offset that would have avoided the problem, then reattempting layout using this new offset.
  * Adjusting the scroll offset will not reset other child state \(e.g., the position of children in a `SliverList`\); thus, when such a sliver requests a scroll offset correction, the offset selected is one that would cause any existing, unaltered state to be consistent.
  * For example, a `SliverList` may not have enough room for newly revealed children when scrolling backwards \(e.g., because the incoming scroll offset indicates an inadequate number of pixels preceding the viewport’s leading edge\). The `SliverList` calculates the scroll offset correction that would have avoided this logical inconsistency, adjusting it \(and any affected layout offsets\) to ensure that children appear at the same visual location in the viewport.
* If a chain of corrections occurs, layout will eventually fail.

