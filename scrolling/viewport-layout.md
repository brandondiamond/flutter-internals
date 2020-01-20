# Viewport Layout


## How does a viewport layout its children?

* `RenderViewport` kicks off layout via `RenderViewport.performLayout`.
  * Out-of-band data is incorporated into layout, supplementing constraints and geometry.
    * `minScrollExtent`: total scroll extent of reverse slivers, negated.
    * `maxScrollExtent`: total scroll extent of forward slivers.
    * `hasVisualOverflow`: whether clipping is needed when painting.
  * `RenderViewport.performLayout` applies the center sliver adjustment and repeatedly attempts layout until layout succeeds or a maximum number of attempts is reached.
    * Main and cross axis extents are derived from width and height given the viewport’s axis.
    * Attempt to layout all children \(`RenderViewport._attemptLayout`\), passing the current scroll offset \(`ViewportOffset.pixels`\) including any visual adjustment of the center sliver \(`RenderSliver.centerOffsetAdjustment`\). 
    * If the attempt fails due to a scroll offset correction being returned, that correction is applied \(`ViewportOffset.pixels` is adjusted additively, without notifying listeners\) and layout re-attempted.
    * If the attempt succeeds, the viewport must verify that the final content dimensions would not invalidate the viewport’s current pixel offset \(via `ViewportOffset.applyContentDimensions`\); if it does, layout must be re-attempted. 
      * The out-of-band minimum and maximum scroll extents are transformed into “slack” values, representing how much the viewport can be scrolled in either direction from the zero offset, excluding pixels visible in the viewport.
      * As an example, if layout was attempted assuming a large viewport offset, but layout indicates that there is no slack on either side of the viewport, this would be considered an invalid state -- the viewport shouldn’t have been scrolled so far. \[?\]
      * The typical implementation of this method \(`ScrollPositionWithSingleContext.applyContentDimensions`\) never requests re-layout. Intead, the current scroll activity is given an opportunity to react to any changes in slack \(`ScrollActivity.applyNewDimensions`\).
  * `RenderViewport._attemptLayout` lays out the reverse slivers then, if successful, the forward slivers; else, it returns a scroll offset correction. Before attempting layout, this method clears stale layout information \(e.g., out-of-band state\) and calculates a number of intermediate values used to determine the constraints passed to the first sliver in each sequence \(i.e., the arguments to `RenderViewportBase.layoutChildSequence`\).
    * All out-of-band data is cleared and the intermediate viewport metrics calculated.
      * `centerOffset`: the offset from the viewport’s outer leading edge to the centerline \(i.e., from the current scroll offset to the zero scroll offset\).
        * Negative when scrolled into the forward region, positive when scrolled into the reverse region.
        * If the anchor \(i.e., visual adjustment of the centerline\) is non-zero, applies an additional offset proportional to the main extent. This offset affects all other metrics, too.
      * `reverseDirectionRemainingPaintExtent`: total visible pixels available for painting reverse slivers.
      * `forwardDirectionRemainingPaintExtent`: total visible pixels available for painting forward slivers.
      * `fullCacheExtent`: the viewport’s visual extent plus the two adjacent cache extents \(`RenderViewportBase.cacheExtent`\).
        * Subsequent definitions refer to the region defined by the leading cache extent, the visible viewport, and the trailing cache extent as the “cacheable viewport.”
      * `centerCacheOffset`: the offset from the leading edge of the cacheable viewport to the centerline.
      * `reverseDirectionRemainingCacheExtent`: total visible and cacheable pixels available for painting reverse slivers.
      * `forwardDirectionRemainingCacheExtent`: total visible and cacheable pixels available for painting forward slivers.
    * If present, reverse slivers are laid out via `RenderViewportBase.layoutChildSequence`. The parameters to this method serve as initial values for the first reverse sliver; as layout progresses, they are incrementally adjusted for each subsequent sliver.
      * In the following descriptions, since reverse slivers are effectively laid out in the opposite axis direction, the meaning of “leading” and “trailing” are flipped. That is, what was previously the trailing edge of the viewport / sliver is now described as its leading edge.
      * child: the initial reverse sliver \(immediately preceding the center sliver\).
      * `scrollOffset`: the amount the first reverse sliver has been scrolled before the leading edge of the viewport.
        * If the forward region is visible, this is always zero. This is because the initial reverse sliver must necessarily be after the leading edge.
      * overlap: zero, since nothing has been painted.
      * `layoutOffset`: the offset to apply to the sliver’s layout position \(i.e., to push it past any pixels earmarked for other slivers\), effectively measured from the viewport’s leading edge \(though technically measured from the centerline\). Only increased by slivers that are on screen.
        * If the forward region is not visible, this is always zero. This is because layout may start from the leading edge unimpeded.
        * If the forward region is visible, this is `forwardDirectionRemainingPaintExtent`; those pixels are earmarked for forward sliver layout \(which, being visible, will reduce the available space in the viewport\). Thus, the first reverse sliver must be “bumped up” by a corresponding amount.
          * If the forward region is partially visible, the above represents how much of the visible viewport has been allocated for forward layout.
          * If the forward region is solely visible, the above still applies \(i.e., layout offset must be adjusted by `forwardDirectionRemainingPaintExtent` which, in this case, is the full extent\). 
          * No adjustment beyond this is necessary because the forward region appears after the reverse region and can therefore only interact with reverse layout by consuming paint extent.
      * `remainingPaintExtent`: total visible pixels available for reverse slivers \(`reverseDirectionRemainingPaintExtent`\).
      * advance: `childBefore` to iterate backwards.
      * `remainingCacheExtent`: total visible and cacheable pixels available for painting \(`reverseDirectionRemainingCacheExtent`\).
      * `cacheOrigin`: how far before the leading edge is cacheable \(always negative\). Forward region not eligible.
    * Forward slivers are laid out via `RenderViewportBase.layoutChildSequence`. The parameters to this method serve as initial values for the center sliver. These values are the opposite of the reverse values, with a handful of exceptions.
      * “Leading” and “trailing” once again refer to their intuitive definitions.
      * child: the center sliver \(i.e., the first forward sliver\).
      * `scrollOffset`: the amount the centerline has been scrolled before the leading edge of the viewport.
        * If the reverse region is visible, this is always zero. This is because the center sliver must necessarily be after the leading edge.
      * overlap: number of pixels needed to jump from the starting layout position to the first pixel not painted on by an earlier sliver \(before or after the sliver’s leading edge\).
        * If reverse slivers were laid out, the reverse region is treated as a self-contained block that fully consumes its portion of the viewport. Thus, there can be no overlap. \[?\]
        * If reverse slivers were not laid out, the initial overlap is how far the centerline is from the viewport’s leading edge \(i.e., the full extent of the reverse region, negated\). Intuitively, a negative overlap indicates that the first clean pixel is before the center sliver’s leading edge and not after.
          * `RenderViewportBase.layoutChildSequence` tracks the furthest offset painted so far \(`maxPaintOffset`\). Passing a negative value ensures that the reverse region \(e.g., due to overscroll\) is captured by this calculation; else it would remain empty. 
          * For example, failing to do this might cause a pinned app bar to separate from the top of the viewport during overscroll.
      * `layoutOffset`: the offset to apply to the sliver’s layout position \(i.e., to push it past any pixels earmarked for other slivers\), effectively measured from the viewport’s leading edge \(though technically measured from the centerline\). Only increased by slivers that are on screen.
        * If the reverse region is not visible, this is always zero. This is because layout may start from the leading edge unimpeded.
        * If the reverse region is visible, the layout offset must be bumped by the full extent of the reverse region -- not just the visible portion \(note that this differs from the reverse case\). Forward slivers are laid out after reverse slivers, and so must account for the full amount of potentially consumed layout.
          * If the reverse region is partially visible, the offset used is `forwardDirectionRemainingPaintExtent`. This represents how much of the visible viewport has been allocated for reverse layout.
          * If the reverse region is solely visible, the offset used is the full distance from the viewport’s outer leading edge to the centerline. That is, the entire reverse region is considered as having been used for layout.
          * Though this is an upper-bound approximation, this value is acceptable since forward slivers won’t be painted \(they are offscreen by assumption\). Consequently, layout offsets within the forward region serve mainly for ordering, not precise positioning. \[?\]
      * `remainingPaintExtent`: total visible pixels available for forward slivers \(`forwardDirectionRemainingPaintExtent`\).
        * Note that slivers preceding the leading edge will receive the full value even if they are off screen; it is up to the sliver’s interpretation of its scroll offset to determine whether it consumes pixels.
      * advance: `childAfter` to iterate forwards.
      * `remainingCacheExtent`: total visible and cacheable pixels available for painting \(`forwardDirectionRemainingCacheExtent`\).
      * `cacheOrigin`: how far before the leading edge is cacheable \(always negative\). Reverse region not eligible.

## How are child sequences laid out?

* `RenderViewportBase.layoutChildSequence` is the engine that drives sliver layout. It processes a sequence of children one by one, passing initial constraints and adjusting layout state as geometry is produced.
  * Layout state is initialized. The initial layout offset is captured and the effective scroll direction computed \(this direction will be flipped to account for reverse growth\).
    * `maxPaintOffset`: the layout offset corresponding to the farthest pixel painted so far. Adjusted by incoming overlap \(e.g., to allow slivers to avoid overlapping with earlier slivers; or, when negative, to allow slivers to fill empty space due to overscroll\).
  * For every child, invoke `RenderSliver.performLayout`, passing an appropriate `SliverConstraint` instance.
    * `scrollDirection`: the effective scroll direction.
    * `scrollOffset`: the incoming \(e.g., center sliver’s\) scroll offset, decremented by all preceding slivers’ scroll extents. Negative values are clamped at zero since fully exposed slivers have a zero scroll offset.
      * Conceptually, this is the amount that the current sliver’s leading edge precedes the viewport’s leading edge as reckoned using earlier slivers’ extents.
    * overlap: the difference between the farthest pixel painted thus far \(`maxPaintOffset`\) and the current layout offset.
      * Conceptually, this is the offset needed to jump to the nearest clean pixel from the sliver’s layout position. Can be negative.
    * `precedingScrollExtent`: sum of all preceding slivers’ scroll extents, including those that are off screen.
      * Note that a sliver may consume infinite scroll extent, so this quantity may be infinite.
    * `remainingPaintExtent`: the original paintable extent decreased by the total layout extent consumed so far \(paint extent is not used because painting can overlap whereas layout cannot\).
      * Conceptually, this is how much the sliver may choose to paint starting from its scroll offset.
    * `parentUsesSize`: this is always true; if a sliver must lay out, so too must the viewport.
    * `correctedCacheOrigin` \(intermediate value, negative\): ensure the cache origin falls within the sliver, clamping it to the sliver’s leading edge. 
    * `cacheExtentCorrection` \(intermediate value, negative\): how much of the cache region could not be filled by the sliver \(i.e., because it precedes its leading edge\).
    * `cacheOrigin`: the corrected cache origin.
      * Even if the sliver significantly precedes the viewport’s leading edge \(i.e., has a large positive scroll offset\), the cache origin is added to the sliver’s scroll offset \(i.e., the offset needed to reach the viewport’s leading edge from the sliver’s leading edge\) and will therefore correctly index into the leading cache region.
    * `remainingCacheExtent`: the original cacheable extent \(the leading and trailing cache regions plus the viewport’s main extent\) decreased by the total cache extent consumed so far.
      * Conceptually, this is how much the sliver may choose to paint starting from the corrected cache origin. Always larger than `remainingPaintExtent`.
      * This is adjusted by `cacheExtentCorrection` to discount the portion of the cache that is inaccessible to the sliver. The sliver is effectively starting further into the cacheable region, not bumping the cacheable region out to ensure that the cacheable region ends at the same place.
  * If layout returns an offset correction, control returns to the caller which, in most cases, applies the correction and reattempts layout.
  * Layout state is mutated to account for the geometry produced by the sliver.
    * `effectiveLayoutOffset`: the layout offset as adjusted by the sliver’s `SliverGeometry.paintOrigin` \(negative values shift the offset toward the leading edge; the default is zero\).
      * Conceptually, this is where the sliver has been visually positioned in the viewport. Note that it will still consume `SliverGeometry.layoutExtent` pixels at its original layout offset and not the effective layout offset.
    * `maxPaintOffset`: update if the farthest pixel painted by this sliver \(the effective layout offset increased by `SliverGeometry.paintExtent`\) exceeds the previous maximum.
    * `scrollOffset`: decrease by the scroll extent of the sliver. Unlike `sliverScrollOffset`, which is clamped at zero, this quantity is permitted to become negative \(representing the proportion of total scroll extent after the viewport’s leading edge\).
    * `layoutOffset`: increase by the layout extent of the sliver. Layout extent must be less than paint extent, which itself must be less than remaining paint extent. Thus, only those slivers consuming space in the visible viewport will increase layout offset.
    * Out of band data is updated to reflect the total forward and reverse scroll extent encountered so far \(`_maxScrollExtent` and `_minScrollExtent`, respectively\), as well as whether clipping is needed \(`_hasVisualOverflow`).
  * Update cache state to if any cache was consumed \(i.e., `SliverGeometry.cacheExtent` &gt; 0\).
    * `remainingCacheExtent`: reduce the cacheable extent by the number of pixels that were either consumed or unreachable due to preceding the sliver’s leading edge \(`cacheExtentCorrection`\).
    * `cacheOrigin`: if the leading cacheable extent has not been exhausted, update the origin to point to the next eligible pixel; this offset must be negative. Otherwise, set the cache origin to zero. Subsequent slivers may still attempt to fill the trailing cacheable extent if space permits.
  * Update the sliver’s parent data to reflect its position.
    * If the sliver precedes the viewport’s leading edge or is otherwise visible, the effective layout offset correctly represents this sliver’s position.
    * Otherwise, construct an approximate position by adding together the initial layout offset and the proportion of total scroll extent after the viewport’s leading edge.
      * The latter term is equivalent to the initial scroll offset decreased by the total scroll extent processed so far. Once negative, this value represents how far slivers have extended into the visible portion of the viewport \(and beyond\).
      * Layout offsets only increase for slivers that are painted \(and have a `SliverGeometry.layoutExtent` that is non-zero\); thus, only the portion of scroll extent falling within the visible region can possibly increment layout offset. As a result, the above quantity represents an upper-bound offset that can be used to reliably order, if not paint, offscreen slivers.
    * Viewports that use physical parent data \(e.g., `RenderViewport`\) must compute the absolute paint offset to store. Otherwise, the layout offset may be stored directly.

## How do shrink-wrapping viewports handle layout?

* The out-of-band data tracked by shrink-wrapping viewports is extended.
  * `_shrinkWrapExtent`: the total maximum paint extent \(`SliverGeometry.maxPaintExtent`\) of all slivers laid out so far. Maximum paint extent represents how much a sliver could paint without constraint. Thus, this extent represents the amount of space needed to fully paint all children.
  * There is no minimum extent since there are no reverse slivers.
* There are several major differences when performing layout.
  * `RenderShrinkWrappingViewport.performLayout` is the entrypoint to viewport layout, repeating layout until there are no offset corrections and final dimensions are deemed valid.
    * If there are no children, the viewport expands to fill the cross axis, but is as small as possible in the main axis; it then returns. Else, the extents are initialized to be as big as the constraints permit.
    * Layout is attempted \(`RenderShrinkWrappingViewport._attemptLayout`\) using the unadulterated viewport offset \(`ViewportOffset.pixels`\). Any center offset adjustment is ignored. If layout produces a scroll offset correction, that correction is applied, and layout reattempted.
    * Effective extent is computed by clamping the maximum extent needed to paint all children \(`_shrinkWrapExtent`) to the incoming constraints.
    * Final layout dimensions \(inner and outer\) are verified. If either would invalidate the viewport offset \(`ViewportOffset.pixels`\), layout must be reattempted.
      * The outer dimension is set to the effective extent and verified via `ViewportOffset.applyViewportDimensions`.
      * The inner dimension is verified via `ViewportOffset.applyContentDimensions`, which requires “slack” values representing how far before and after the zero offset the viewport can scroll. Since there are no reverse slivers, the minimum extent is zero. The maximum extent is the difference between the total scrollable extent and the total paintable extent of all children \(clamped to zero\). That is, how much additional scrolling is needed even after all children are painted.
    * Last, the viewport is sized to match its effective extent, clamped by the incoming constraints.
  * `RenderShrinkWrappingViewport._attemptLayout` is a thin wrapper around `RenderViewportBase.layoutChildSequence` that configures the layout parameters passed to the first child in the sequence.
    * child: the first child, since there are no reverse slivers.
    * `scrollOffset`: incoming offset, clamped above zero \(i.e., the amount the viewport is scrolled\).
    * `scrollOffset`: incoming offset, clamped below zero \(i.e., the amount the viewport is over-scrolled\).
    * `layoutOffset`: always zero.
    * `remainingPaintExtent`: the main axis extent \(i.e., the full viewport\)
    * advance: `childAfter` to iterate forwards.
    * `remainingCacheExtent`: the main axis extent plus two cache extents.
    * `cacheOrigin`: `cacheExtent` pixels before the layout offset.

