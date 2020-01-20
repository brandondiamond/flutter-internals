# Dynamic Slivers


## What are a few common types of dynamic slivers?

* `RenderSliverList` positions children of varying extents in a linear array along the main axis. All positioning is relative to adjacent children in the list. Since only visible children are materialized, and earlier children may change extent, positions are occasionally corrected to maintain a consistent state. 
* `RenderSliverGrid` positions children in a two dimensional arrangement determined during layout by a `SliverGridDelegate`. The positioning, spacing, and size of each item is generally static, though the delegate is free to compute an arbitrarily complex layout. The current layout strategy does not support non-fixed extents.

## How do lists without variable item extent perform layout?

* When extent is not fixed, position cannot be directly computed from a child’s index. Instead, position must be measured by laying out all preceding children. Since this would be quite expensive and eliminate the benefit of viewport culling, an optimization is used whereby previously laid out children serve to anchor newly revealed children \(i.e., positions are determined relative to previously laid out children\).
  * Note that this can lead to inconsistencies if off-screen children change size or are removed; scroll offset corrections are applied to address these errors.
* Layout proceeds by establishing the range of children currently visible in the viewport; that is, the range of children beginning from the child starting at or intersecting the viewport’s leading edge to the child ending at or intersecting its trailing edge. Unlike a fixed extent list, this range cannot be computed directly but must be measured by traversing and building children starting from the first known child. Layout is computed as follows:
  * Ensure that there’s a first known child \(since, as mentioned, layout is performed relative to previously laid out children\).
    * If there isn’t, create \(but do not layout\) this initial child with an offset and index of zero. If this fails, the list has zero extent and layout is complete.
  * If the first known child does not precede or start at the viewport’s leading edge, build children toward the leading edge until such a child is identified.
    * Leading children must be built as the walk progresses since they must not have existed \(else, there would have been a different first known child\). As part of building, the item is inserted into the list’s child model.
    * If there are no more children to build, the last successfully processed child is positioned at offset zero \(i.e., bumped up to the top of the list\).
      * If the incoming scroll offset is already zero, then the walk ends; this child satisfies the search criteria.
      * If the scroll offset is non-zero, a scroll offset correction is needed.
        * A non-zero offset implies that a portion of the list precedes the leading edge. Given that there aren’t even enough children to reach the leading edge, this cannot be the case.
        * The correction ensures that the incoming scroll offset will be zero when layout is reattempted. Since the earliest child is now positioned at offset zero, the inconsistency is corrected.
    * The newly built child’s scroll offset is computed by subtracting its paint extent from the last child’s scroll offset.
      * If the resulting offset is negative, a scroll offset correction is needed.
        * A negative offset implies that there is insufficient room for the child. The last child’s offset represents the number of pixels available before that child; if this is less than the new child’s extent, the list is too small and the incoming scroll offset must be invalid.
        * All preceding children that do not fit \(including the one that triggered this case\) are built and measured to determine their total extent. The earliest such child is positioned at offset zero. 
        * A scroll offset correction is calculated to allow sufficient room for the overflowing children while ensuring that the last processed child appears at the same visual location.
          * This quantity is the total extent needed minus the last walked item’s scroll offset.
    * Position the child at the calculated scroll offset.
  * The first child in the list must now precede or start at the viewport’s leading edge. Ensure that it has been laid out \(e.g., if the preceding walk wasn’t necessary\).
  * Advance to find the child starting at or intersecting with the viewport’s leading edge \(there may have already been several such children in the child list\). Then, advance to find the child ending at or intersecting with the viewport’s trailing edge.
    * Advance by identifying the next child in the list while incrementing an index counter to detect gaps.
      * If the next child hasn’t been built or a gap is detected, build and layout the child at the current index.
        * If no more children can be built, report failure.
      * If the next child hasn’t been laid out, lay it out now.
    * While advancing, position each child directly after the preceding child.
    * If advancing fails before the leading edge is reached, remove all but the latest such child \(via garbage collection\). Maintain this child as it captures the list’s total dimensions \(i.e., its position plus its paint extent corresponds to the list’s scroll extent\).
      * Complete layout with zero paint extent, using the last item to compute overall scroll and maximum paint extents.
  * Count the children that wholly precede the viewport’s leading edge. Once the trailing child is found, count all children following it. Remove the corresponding children via garbage collection since they are no longer visible.
  * Return the resulting geometry:
    * `SliverGeometry.scrollExtent`: estimated maximum extent \(this is correct for fixed extent lists\).
    * `SliverGeometry.maxPaintExtent`: estimated maximum extent \(this is the most that can be painted\).
    * `SliverGeometry.paintExtent`: the portion of reified children that are actually visible \(via `RenderSliver.calculatePaintOffset`\).
    * `SliverGeometry.hasVisualOverflow`: true if the trailing child extends beyond the viewport’s leading edge, or the list precedes the viewport’s leading edge \(i.e., incoming scroll offset is greater than zero\).
  * If the list was fully scrolled, it will not have had an opportunity to lay out children. However, it is still necessary to report underflow to the manager.

## What are the building blocks of grid layout?

* `SliverGridGeometry` captures the geometry of an item within a grid. This encompasses a child’s scroll offset, cross axis offset, main axis extent, and cross axis extent.
* `SliverGridParentData` extends `SliverMultiBoxAdaptorParentData` to include the child’s cross axis offset. This is necessary since multiple children within a grid can share the same scroll offset while appearing at different cross axis offsets.
* `SliverGridLayout` encapsulates positioning, sizing, and spacing logic. It is consulted during layout to determine the position and size of each grid item. This information is provided by returning the minimum and maximum index for a given scroll offset \(via `SliverGridLayout.getMinChildIndexForScrollOffset` and  `SliverGridLayout.getMaxChildIndexForScrollOffset`\), as well as the grid geometry for a given index \(via `SliverGridLayout.getGeometryForChildIndex`\).
* `SliverGridDelegate` builds a `SliverGridLayout` subclass on demand \(i.e., during grid layout\). This allows the calculated layout to adjust to incoming sliver constraints.
* `SliverGridRegularTileLayout` calculates a layout wherein children are equally sized and spaced. As such, all aspects of layout are computed directly \(i.e., without measuring adjacent children\).
* `SliverGridDelegateWithFixedCrossAxisCount` configures a `SliverGridRegularTileLayout` such that the same number of children appear at a given scroll offset. Children are sized to share the available cross axis extent equally.
* `SliverGridDelegateWithMaxCrossAxisExtent` configures a `SliverGridRegularTileLayout` such that tiles are no larger than the provided maximum cross axis extent. A candidate extent is calculated that divides the available space evenly \(i.e., without a remainder\) and that is as large as possible.

## How do grids perform layout?

* Grids are laid out similarly to fixed extent lists. Once a layout is computed \(via `SliverGridDelegate.getLayout`\), it is treated as a static description of the grid \(i.e., positioning is absolute\). As a result, children do not need to be reified to measure extent and position. Layout proceeds as follows:
  * Compute a new `SliverGridLayout` via `SliverGridDelegate`, providing the incoming constraints..
  * Target first and last indices are computed based on the children that would be visible given the scroll offset and the remaining paint extent \(via `SliverGridLayout.getMinChildIndexForScrollOffset` and `SliverGridLayout.getMaxChildIndexForScrollOffset`, respectively\).
    * Shrink wrapping viewports have infinite extent. In this case, there is no last index.
  * Any children that were visible but are now outside of the target index range are garbage collected \(via `RenderSliverMultiBoxAdaptor.collectGarbage`\). This also cleans up any expired keep alive children.
  * If there are no children attached to the grid, insert \(but do not lay out\) an initial child at the first index.
    * If this child cannot be built, layout is completed with scroll extent and maximum paint extent set to the calculated max scroll offset \(via `SliverGridLayout.computeMaxScrollOffset`\); all other geometry remains zero.
  * All children still attached to the grid fall in the visible index range and there is at least one such child.
  * If indices have become visible that precede the first child’s index, the corresponding children are built and laid out \(via `RenderSliverMultiBoxAdaptor.insertAndLayoutLeadingChild`\).
    * These children will need to be built since they could not have been attached to the grid by assumption.
    * If one of these children cannot be built, layout will fail. This is likely a bug.
  * Identify the child with the largest index that has been built and laid out so far. This is the trailing child.
    * If there were leading children, this will be the leading child adjacent to the initial child. If not, this is the initial child itself \(which is now laid out if necessary\).
  * Lay out every remaining child until there is no more room \(i.e., the target index is reached\) or no more children \(i.e., a child cannot be built\). Update the trailing child as layout progresses.
    * The trailing child serves as the “after” argument when inserting children \(via `RenderSliverMultiBoxAdaptor.insertAndLayoutChild`\).
    * The children may have already been attached to the grid. If so, the child is laid out without being rebuilt.
    * Layout offsets for both the main and cross axes are assigned according the geometry reported by the `SliverGridLayout` \(via `SliverGridLayout.getGeometryForChildIndex`\).
  * Compute the estimated maximum extent using the first and last index that were actually reified as well as the enclosing leading and trailing scroll offsets.
  * Return the resulting geometry:
    * `SliverGeometry.scrollExtent`: estimated maximum extent \(this is correct for grids with fixed extent\).
    * `SliverGeometry.maxPaintExtent`: estimated maximum extent \(this is the most that can be painted\).
    * `SliverGeometry.paintExtent`: the visible portion of the range defined by the leading and trailing scroll offsets.
    * `SliverGeometry.hasVisualOverflow`: always true, unfortunately.
  * If the list was fully scrolled, it will not have had an opportunity to lay out children. However, it is still necessary to report underflow to the manager.

