# Persistent Headers


## How do persistent headers work?

* RenderSliverPersistentHeader provides a base class for implementing persistent headers within a viewport, adding support for varying between a minimum and maximum extent during scrolling. Some subclasses introduce pinning behavior, whereas others allow the header to scroll into and out of out of view. SliverPersistentHeader encapsulates this behavior into a stateless widget, delegating configuration \(and child building\) to a SliverPersistentHeaderDelegate instance.
  * Persistent headers can be any combination of pinnable and floating \(or neither\); those that float, however, can also play a snapping animation. All persistent headers expand and contract in response to scrolling; only floating headers do so in response to user scrolling anywhere in the viewport.
    * A floating header reappears whenever the user scrolls its direction. The header expands to its maximum extent as the user scrolls toward it, and shrinks as the user scrolls away.
    * A pinned header remains at the top of the viewport. Unless floating is also enabled, the header will only expand when approaching its actual position \(e.g., the top of the viewport\).
    * Snapping causes a floating header to animate to its expanded or contracted state when the user stops scrolling, regardless of scroll extent.
  * Persistent headers contain a single box child and track a maximum and minimum extent. The minimum extent is typically based on the box’s intrinsic dimensions \(i.e., the object’s natural size\). As per the render box protocol, by reading the child’s intrinsic size, the persistent header will be re-laid out if this changes.
  * RenderSliverPersistentHeader doesn’t configure parent data or determine how to position its child along the main axis. It does, however, provide support for hit testing children \(which requires subclasses to override RenderSliverPersistentHeader.childMainAxisPosition\). It also paints its child without using parent data, computing the child’s offset in the same way as RenderSliverSingleBoxAdapter \(i.e., to determine whether its viewport-naive box is offset on the canvas if partially scrolled out of view\).
  * Two major hooks are exposed:
    * RenderSliverPersistentHeader.updateChild: supports updating the contained box whenever the persistent header lays out or the box itself changes size. This is provided two pieces of layout information.
      * Shrink offset is the delta between the current and maximum extents \(i.e., how much more room there is to grow\). Always positive.
      * Overlaps content is true if the header’s leading edge is not at its layout position in the viewport.
    * RenderSliverPersistentHeader.layoutChild: invoked by subclasses to updates then lay out children within the largest visible portion of the header \(between maximum and minimum extent\). Subclasses provide a scroll offset, maximum extent, and overlap flag, all of which may differ from the incoming constraints.
      * Shrink offset is set to the scroll offset \(how far the header is before the viewport’s leading edge\) and is capped at the max extent. Conceptually, this represents how much of the header is off screen.
      * If the child has changed size or the shrink offset has changed, the box is given an opportunity to update its appearance \(via RenderSliverPersistentHeader.updateChild\). This is done within a layout callback since, in the common case, the child is built dynamically by the delegate. This is an example of interleaving build and layout.
        * This flow is facilitated by the persistent header widgets \(e.g., \_SliverPersistentHeaderRenderObjectWidget\), which utilize a custom render object element \(\_SliverPersistentHeaderElement\).
        * When the child is updated \(via RenderSliverPersistentHeader.updateChild\), the specialized element updates its child’s element \(via Element.updateChild\) using the widget produced by the delegate’s build method.
      * Finally, the child is laid out with its main extent loosely constrained to the portion of the header that’s visible -- with the minimum extent as a lower bound \(i.e., so that the box is never forced to be smaller than the minimum extent\).

## How is the expanding / contracting scrolling effect implemented?

* RenderSliverScrollingPersistentHeader expands to its maximum extent when scrolled into view, and shrinks to its minimum extent before being scrolled out of view.
  * Lays out the child in the largest visible portion of the header \(up to the maximum extent\), then returns its own geometry.
    * SliverGeometry.scrollExtent: always the max extent since this is how much scrolling is needed to scroll past the header.
    * SliverGeometry.paintOrigin: paints before itself if there’s a negative overlap \(e.g., to fill empty space in the viewport from overscroll\).
    * SliverGeometry.paintExtent, SliverGeometry.layoutExtent: the largest visible portion of the header, clamped to the remaining paint extent.
    * SliverGeometry.maxPaintExtent: the maximum extent; it’s not possible to provide more content.
  * Tracks the child’s position across the scroll \(i.e., the distance from the header’s leading edge to the child’s leading edge\). The child is aligned with the trailing edge of the header to ensure it scrolls into view first.
    * Calculated as the portion of the maximum extent in view \(which is negative when scrolled before the leading edge\), minus the child’s extent after layout.

## How does pinning work?

* RenderSliverPinnedPersistentHeader is similar to its scrolling sibling, but remains pinned at the top of the viewport regardless of offset. It also avoids overlapping earlier slivers \(e.g., useful for building stacking section labels\).
  * The pinned header will generally have a layout offset of zero when scrolled before the viewport’s leading edge. As a consequence, any painting that it performs will be coincident with the viewport’s leading edge. This is how pinning is achieved.
    * Recall that layout offset within a viewport only increases when slivers report a non-zero layout extent. Slivers generally report a zero layout extent when they precede the viewport’s leading edge \(i.e., when viewport scroll offset exceeds their extent\); thus, when the header precedes the viewport’s leading edge, it will likely have a zero layout offset. Additionally, recall that layout offset is used when the viewport paints its children. Since the header will have a zero layout offset, at least in the case of RenderViewport, the sliver will be painted at the viewport’s painting origin.
    * If an earlier, off-screen sliver consumes layout, this will bump out where the header paints. This might cause strange behavior.
    * If another pinned header precedes this one, it will not consume layout. However, by utilizing the overlap offset, the current header is able to avoid painting on top of the preceding header.
  * Next, it lays out the child in the largest visible portion of the header \(up to the maximum extent\), then returns its own geometry.
    * SliverGeometry.scrollExtent: always the max extent since this is how much scrolling is needed to scroll past the header \(though, practically speaking, it can never truly be scrolled out of view\).
    * SliverGeometry.paintOrigin: always paints on the first clean pixel to avoid overlapping earlier slivers.
    * SliverGeometry.paintExtent: always paints the entire child, even when scrolled out of view. Clamped to remaining paint extent.
    * SliverGeometry.layoutExtent: the pinned header will consume the portion of its maximum extent that is actually visible \(i.e., it only takes up space when its true layout position falls within the viewport\). Otherwise \(e.g., when pinned\), it occupies zero layout space and therefore may overlap any subsequent slivers. Clamped to remaining paint extent.
    * SliverGeometry.maxScrollObstructionExtent: the minimum extent, since this is the most that the viewport can be obscured when pinned.
  * The child is always positioned at zero as this sliver always paints at the viewport’s leading edge.

## How does floating work?

* RenderSliverFloatingPersistentHeader is similar to its scrolling sibling, but reattaches to the viewport’s leading edge as soon as the user scrolls in its direction. It then shrinks and detaches if the user scrolls away.
  * The floating header tracks scroll offset, detecting when the user begins scrolling toward the header. The header maintains an effective scroll offset that matches the real scroll offset when scrolling away, but that enables floating otherwise.
    * It does this by jumping ahead such that the sliver’s trailing edge \(as measured using the effective offset and maximum extent\) is coincident with the viewport’s leading edge. This is the “floating threshold.” All subsequent scrolling deltas are applied to the effective offset until the user scrolls the header before the floating threshold. At this point, normal behavior is resumed.
  * RenderSliverFloatingPersistentHeader.performLayout detects the user’s scroll direction and manages an effective scroll offset. The effective scroll offset is used for updating geometry and painting, allowing the header to float above other slivers.
    * The effective scroll offset matches the actual scroll offset the first time layout is attempted or whenever the user is scrolling away from the header and the header isn’t currently floating.
    * Otherwise, the header is considered to be floating. This occurs when the user scrolls toward the header, or the header’s actual or effective scroll offset is less than its maximum extent \(i.e., the header’s effective trailing edge is at or after the viewport’s leading edge\).
      * When floating first begins \(i.e., because the user scrolled toward the sliver\), the effective scroll offset jumps to the header’s maximum extent. This effectively positions its trailing edge at the viewport’s leading edge.
      * As scrolling continues, the delta \(i.e., the change in actual offset\) is applied to the effective offset. As a result, geometry is updated as though the sliver were truly in this location.
      * The effective scroll offset is permitted to become smaller, allowing the header to reach its maximum extent as it scrolls into view.
      * The effective scroll offset is also permitted to become larger such that the header shrinks. Once the header is no longer visible, the effective scroll offset jumps back to the real scroll offset, and the header is no longer floating.
    * Once the effective scroll offset has been updated, the child is laid out using the effective scroll offset, the maximum extent, and an overlap flag.
      * The header may overlap content whenever it is floating \(i.e., its effective scroll offset is less than its actual scroll offset\).
    * Finally, the header’s geometry is computed and the child is positioned.
  * RenderSliverFloatingPersistentHeader.updateGeometry computes the header’s geometry as well as the child’s final position using the effective and actual scroll offsets.
    * SliverGeometry.scrollExtent: always the max extent since this is how much scrolling is needed to scroll past the header.
    * SliverGeometry.paintOrigin: always paints on the first clean pixel to avoid overlapping earlier slivers.
    * SliverGeometry.paintExtent: the largest visible portion of the header \(using its effective offset\), clamped to the remaining paint extent.
    * SliverGeometry.layoutExtent: the floating header will consume the portion of its maximum extent that is actually visible \(i.e., it only takes up space when its true layout position falls within the viewport\). Otherwise \(e.g., when floating\), it occupies zero layout space and therefore may overlap any subsequent slivers. Clamped to remaining paint extent.
    * SliverGeometry.maxScrollObstructionExtent: the maximum extent, since this is the most that the viewport can be obscured when floating.
  * The child’s position is calculated using the sliver’s effective trailing edge \(i.e., as measured using the sliver’s effective scroll offset\). When the sliver’s actual position precedes the viewport’s leading edge, its layout offset will typically be zero, and thus the sliver will paint at the viewport’s leading edge.

## How does pinning and floating work together?

* RenderSliverFloatingPinnedPersistentHeader is identical to its parent \(RenderSliverFloatingPersistentHeader\) other than in how it calculates its geometry and child’s position. Like its parent, the header will reappear whenever the user scrolls toward it. However, the header will remained pinned at the viewport’s leading edge even when the user scrolls away.
  * The calculated geometry is almost identical to its parent. The key difference is that the pinned, floating header always paints at least its minimum extent \(room permitting\). Additionally, its child is always positioned at zero since painting always occurs at the viewport’s leading edge.
    * SliverGeometry.scrollExtent: always the max extent since this is how much scrolling is needed to scroll past the header \(though it will continue to paint itself\).
    * SliverGeometry.paintOrigin: always paints on the first clean pixel to avoid overlapping earlier slivers.
    * SliverGeometry.paintExtent: the largest visible portion of the header \(using its effective offset\). Never less than the minimum extent \(i.e., the child will always be fully painted\), never more than the remaining paint extent. Unlike the non-floating pinned header, this will vary so that the header visibly grows to its maximum extent.
    * SliverGeometry.layoutExtent: the pinned, floating header will consume the portion of its maximum extent that is actually visible \(i.e., it only takes up space when its true layout position falls within the viewport\). Otherwise \(e.g., when floating\), it occupies zero layout space and therefore may overlap any subsequent slivers. Clamped to the paint extent, which may be less than the remaining paint extent if still growing between minimum and maximum extent.
    * SliverGeometry.maxScrollObstructionExtent: the maximum extent, since this is the most that the viewport can be obscured when floating.

