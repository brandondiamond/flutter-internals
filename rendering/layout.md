# Layout

## What is a relayout boundary?

* A relayout boundary is a logical divide between nodes in the render tree. Nodes below the boundary can never invalidate the layout of nodes above the boundary.
* The boundary is represented as a render object \(`RenderObject._relayoutBoundary`\) that meets a set of criteria allowing it to layout independently of its parent. That is, the subtree rooted at a relayout boundary node can never invalidate nodes outside of that subtree.
  * A render object defines a relayout boundary \(i.e., cannot influence its parent's layout\) if its parent explicitly ignores its size \(`!parentUsesSize`\),  fully determines its size \(`sizedByParent`\),  provides tight constraints, or isn't a render object \(e.g., `RenderView`\).
  * The last two cases imply that the child cannot change size regardless of what happens in its subtree.
  * The `parentUsesSize` parameter prevents an object from becoming a relayout boundary. This ensures that the render object's parent will always be marked dirty if the object itself is marked dirty. In fact, all ancestors up to the nearest relayout boundary are marked dirty when this occurs.
* Relayout boundaries limit how many ancestor nodes must be marked dirty when a render object needs layout. The walk can be cut off at the nearest ancestor relayout boundary since nodes beyond that point cannot be affected by descendant layout.
* Relayout boundaries are ignored when actually performing layout. Render objects always layout their children. However, children are expected to cut off the walk if they haven't been marked dirty or are receiving identical constraints \(i.e., nothing has changed so layout isn't needed\). `RenderObject.layout` applies this logic automatically.

## How is layout marked dirty?

* `PipelineOwner` maintains a list of dirty nodes \(`PipelineOwner._nodeNeedingLayout`\) that are cleaned once per frame \(via `PipelineOwner.flushLayout`\).
* Marking render objects as needing layout allows operations to batched \(i.e., layout can be processed in a single walk rather than potentially laying out nodes multiple times\).
* When a render object is marked dirty \(via `RenderObject.markNeedsLayout`, which sets `RenderObject._needsLayout`\) , all ancestors up to and including the nearest relayout boundary are marked dirty \(via `RenderObject.markParentNeedsLayout`\).
  * Only the nearest enclosing relayout boundary is added to `PipelineOwner._nodeNeedingLayout`. Performing layout on this node will recursively layout all descendants. This also schedules the next frame \(via `PipelineOwner.requestVisualUpdate`\) so that layout changes will be applied \(via `PipelineOwner.flushLayout`\).
  * It's possible that nodes later in the dirty list will be cleaned while laying out earlier nodes; `PipelineOwner.flushLayout` ignores any clean nodes.
* A render object and its parent must be notified if the `sizedByParent` bit changes \(via `RenderObject.markNeedsLayoutForSizedByParentChange`\). Updating this bit can add or remove a relayout boundary from the render tree \(the change is committed subsequently by `RenderObject.layout`\). Most objects do not update this bit dynamically.
  * The parent must be marked dirty because nodes in the dirty list are laid out using `RenderObject._layoutWithoutResize`, which doesn't update relayout boundaries. Thus, since the child may change status, its parent must be laid out, too.
  * Changing this bit also implies that the child will adopt new dimensions, potentially invalidating the parent's layout.

## How is layout performed?

* `PipelineOwner.flushLayout` triggers layout for all nodes in the dirty list \(`PipelineOwner._nodesNeedingLayout`\) in order of increasing depth. The nodes in this list correspond to relayout boundaries and are laid out using `RenderObject._layoutWithoutResize` instead of `RenderObject.layout`. 
  * `RenderObject.performResize` is never necessary for a node in the dirty list. Consider a node that is sized by its parent:
    * If the parent isn't dirty, the incoming constraints won't change. Thus, resizing is unnecessary.
    * If the parent is dirty, since nodes are cleaned from top to bottom, it will be laid out first. This will recursively layout the original node \(potentially resizing it\), causing it to be skipped later in the process.
  * `RenderObject._layoutWithoutResize` never updates cached constraints \(`RenderObject.constraints`\) or the relayout boundary \(`RenderObject._relayoutBoundary`\). 
    * Incoming constraints won't change since ancestor nodes haven't been marked dirty.
    * Relayout boundary won't change since the relationship between this node and its parent hasn't changed. If it had, the parent would have been marked dirty, too \(via `RenderObject.markNeedsLayoutForSizedByParentChange`\).
* A child will perform layout \(via `RenderObject.layout`\) if it's been marked dirty, has received new constraints, or has just become a relayout boundary. Otherwise, it will mark itself clean and cut off the walk.
* * Note that layout optimizations are applied in the marking phase rather than the layout phase. Relayout boundaries provide a limit on the nodes that are marked dirty when a descendant is marked dirty; they have no other direct effect.
* Objects that are sized by their parent \(`RenderObject.sizedByParent`\) adopt a new size using the incoming constraints \(via `RenderObject.performResize`\). All objects, including those that were resized, then perform layout \(via `RenderObject.performLayout`\); this method is responsible for laying out any children \(via `RenderObject.layout`\) and updating their parent data as needed.
  * This may happen before or after the parent depending on whether the parent uses the children's layout.
  * Objects that are not sized by their parent must adopt a new size via `RenderObject.performLayout`.
* Once completed, `RenderObject.layout` marks the object as being clean for layout and dirty for semantics and painting.
* Layout captures the most recent set of constraints \(`RenderObject.constraints`\) and updates a reference to the nearest ancestor render object that defines a relayout boundary \(`RenderObject._relayoutBoundary`\).

## How is layout optimized?

* Certain criteria allow the layout pass to avoid laying out children:
  * If a child is not dirty and the constraints are unchanged, it needn’t be laid out, cutting off the walk.
  * If a parent does not use a child’s size, it needn’t be laid out with the child; the child will necessarily conform to the original input constraints, and thus nothing changes for the parent.
  * If a parent passes tight constraints to its child, it needn’t be laid out with the child; the child cannot change size.
  * If a child is “`sizedByParent`” \(the provided constraints solely determine the size\), and the same constraints are passed again, the parent needn’t be laid out with the child; the child cannot change size.
* The nearest upstream relayout boundary \(`RenderObject._relayoutBoundary`\) is maintained as layout progresses down the tree. When layout is performed \(via `RenderObject.performLayout`\), the render object determines whether it meets any of the optimization criteria described above. If so, it sets itself as the nearest relayout boundary. Otherwise, it adopts its parent's relayout boundary.
  * Updating the relayout boundary causes layout to proceed even if the object is clean and incoming constraints are unchanged.
  * All descendant nodes must update their understanding of the nearest relayout boundary. This is accomplished by recursively visiting to clear the relayout boundary and mark the render object as needing layout \(via `RenderObject._cleanRelayoutBoundary`\). The walk stops at descendants that themselves demarcate a relayout boundary \(i.e., `RenderObject._relayoutBoundary == this`\).
  * These nodes must be marked dirty because their layout may now be stale \(i.e., assumptions that may have led to layout being skipped are no longer true\). Layout is also responsible for propagating the relayout boundary down the tree of affected render objects.

## How can UI be built in response to layout?

* `RenderObject.invokeLayoutCallback` allows a builder callback to be invoked during the layout process. Only subtrees that are still dirty may be manipulated which ensures that nodes are laid out exactly once. This allows children to be built on-demand during layout \(needed for viewports\) and nodes to be moved \(needed for global key reparenting\).
* Several invariants ensure that this is valid:
  * Building is unidirectional -- information only flows down the tree, visiting each node once.
  * Layout visits each dirty node once.
  * Building below the current node cannot invalidate earlier layout; build information only flows down the tree.
  * Building after a node is laid out cannot invalidate that node; nodes are never revisited.
  * Therefore, it is safe to build within the unvisited subtree rooted at a given render object.

## What are the layout primitives for single children?

* `Container` provides a consistent interface to a number of more fundamental layout building blocks. Unless otherwise specified, each box is sized to match its child.
* `ConstrainedBox` applies an extra set of constraints in addition to the incoming constraints. The extra constraints are clamped to the incoming constraints \(via `BoxConstraints.enforce`\) and thus may not violate them. The box becomes as small as possible without a child.
* `SizedBox` delegates to `ConstrainedBox`, providing tight constraints for the specified dimensions. The result is that the child will be sized to these dimensions subject to the incoming constraints. The box is still sized even without a child.
* `UnconstrainedBox` lays out its child with unbounded constraints \(in one or more dimensions\). It then sizes itself to match the child subject to incoming constraints \(i.e., to prevent any actual overflow\). An alignment is applied if the child is a different size than its parent. If there would have been overflow, the child’s painting is clipped accordingly. The box becomes as small as possible without a child.
* `OverflowBox` allows one or more components of the incoming constraints to be completely overwritten. The child is laid out using the altered constraints and therefore may overflow. An alignment is applied if the child is a different size than its parent. In all other ways, this box delegates to the child.
* `SizedOverflowBox` assumes a specific size, subject to the incoming constraints. Its child is laid out using the original constraints, however, and therefore may overflow the parent \(e.g., if the incoming constraints permit a size of up to 100x100, and the parent specifies a size of 50x50, the child can select a larger size, overflowing its parent\). Since this box has an explicit size, its given intrinsic dimensions matching this size.
* `FractionallySizedBox` sizes its child in proportion to the incoming constraints; it sizes itself to the child, but without violating the original constraints. In particular, the maximum constraints are multiplied by a corresponding factor to obtain a tight bound \(if a factor isn’t specified, that dimension’s constraints are left unchanged\). The child is laid out using the new constraints and may therefore overflow the parent. The parent is sized to the child subject to the incoming constraints, and therefore cannot overflow; note that painting and hit testing will not be automatically clipped. Intrinsic dimensions are defined as the child’s intrinsic dimension scaled by the corresponding factor.
* `LimitedBox` applies a maximum width or height only when the corresponding constraint is unbounded. That is, when the incoming constraints are forwarded to the child, the maximum constraint in each dimension is set to a specific value if it is otherwise unbounded. This is typically used to place expanding children into containers with infinite dimensions \(e.g., viewports\).
* `Padding` applies spacing around its child \(via `EdgeInsetsGeometry`\). This is achieved by providing the child with constraints deflated by the padding. The parent assumes the size of the child \(which is zero if there is no child\) inflated by the padding.
  * `EdgeInsetsGeometry` describes offsets in the four cardinal directions. `EdgeInsets` specifies the offsets with absolute positioning \(e.g., left, top, right, bottom\) whereas `EdgeInsetsDirectional` is text-orientation sensitive \(e.g., start, top, end, bottom\). Both allow offsets to be combined, subtracted, inflated, and deflated, as well as queried along an axis.
  * Padding defines intrinsic dimensions by reducing the incoming dimension by the total applicable padding before querying the child; the total padding is then added to the resulting dimension.
* `Transform` applies a transformation matrix during painting. It does not alter layout, delegating entirely to `RenderProxyBox` for these purposes. If the transform can solely be described as a translation, it delegates to its parent for painting, as well. Otherwise, it composites the corresponding transformation before painting its child. Optionally, this transform may be applied before hit testing \(via `Transform.transformHitTests`\).

## What higher level widgets support layout of single children?

* `FittedBox` applies a `BoxFit` to its child. `BoxFit` specifies how a child box is to be inscribed within a parent box \(e.g., `BoxFit.fitWidth` scales the width to fit the parent and the height to maintain the aspect ratio, possibly overflowing the parent; `BoxFit.scaleDown` aligns the child within the parent then scales to ensure that the child does not overflow\). The child is laid out without constraint, then scaled to be as large as possible while maintaining the original aspect ratio and respecting the incoming constraints. The box fit determines a transformation matrix that is applied during painting.
  * The box fit is calculated by `applyBoxFit`, which considers each box’s size without respect to positioning. The result is a `FittedSizes` instance, which describes a region within the child \(the source\) and a region within the parent \(the destination\). In particular, the source measures the portion of the child visible within the parent \(e.g., if this is smaller than the child, the child will be clipped\). The destination measures the portion of the parent occupied by the child \(e.g., if this is smaller than the parent, a portion of the parent will remain visible\).
  * The sizes are transformed into rectangles using the current alignment \(via `Alignment.inscribe`\). Finally, a transformation is composed to translate and scale the child relative to the parent as per the calculated rectangles.
* `Baseline` aligns a child such that its baseline is coincident with its own \(as specified\). The child’s baseline is obtained via `RenderBox.getDistanceToBaseline` and an offset calculated to ensure the child’s baseline is coincident with the target line \(itself expressed as an offset from the box’s top\). As a result, though the parent is sized to contain the child, the offset may cause the parent to be larger \(e.g., if the child is shifted below the parent’s top\).
* `Align` allows a child to be aligned within a parent according to a relative position specified via `AlignmentGeometry`. During layout, the child is positioned according to the alignment \(via `RenderAligningShiftedBox.alignChild`, which invokes `AlignmentGeometry.alongOffset`; this maps the alignment to logical coordinates using simple geometry\). Note that the child must undergo layout beforehand since its size is needed to compute its center; the incoming constraints are loosened before being forwarded to the child. Next, the parent is sized according to whether a width or height factor was specified; if so, the corresponding dimension will be set to a multiple of that of the child \(subject to the incoming constraints\). Otherwise, it will expand to fill the constraints. If the incoming constraints are unbounded, the parent will instead shrinkwrap the child.
  * `Alignment` is a subclass of `AlignmentGeometry` that describes a point within a container using units relative to the container’s size and center. The child’s center is positioned such that it is coincident with this point. Each unit is measured as half the container’s corresponding dimension \(e.g., the container is effectively 2 units wide and 2 units tall\). The center of the container corresponds to \(0, 0\), the top left to \(-1, -1\), and the bottom right to \(1, 1\). Units may overflow this range to specify different multiple of the parent’s size.
  * `AlignmentDirectional` is identical but flips the x-axis for right-to-left languages.
  * `FractionalOffset` represents a relative position as a fraction of the container’s size, measured from its top left corner. The child’s top-left corner is positioned such that it is coincident with this point. The center of the container corresponds to \(`0.5`, `0.5`\), the top left to \(0, 0\), and the bottom right to \(1, 1\).
* `IntrinsicHeight` and `IntrinsicWidth` integrate the intrinsic dimension and layout protocols. Intrinsic dimensions reflect the natural size of an object; this definition varies based on what the object actually represents. Prior to laying out, the child’s intrinsic dimension is queried \(e.g., via `RenderBox.getMaxIntrinsicWidth`\). Next, the incoming constraints are tightened to reflect the resulting value. Finally, the child is laid out using the tight constraints. In this way, the child’s intrinsic size is used in determining its actual size \(the two will only match if permitted by the incoming constraints\).
  * Boxes often define their intrinsic dimensions recursively. Consequently, an entire subtree may need to be traversed to produce one set of dimensions. This can quickly lead to polynomial complexity, particularly if intrinsic dimensions are computed several times during layout \(which itself traverses the render tree\).

## What higher level widgets support layout of multiple children?

* `Stack`
* `LayoutBuilder` makes it easy to take advantage of interleaving the build / layout phase via callbacks. The callback has access to the incoming constraints \(i.e., because the framework has completed building and is currently performing layout\) but may still build additional UI \(via `RenderObject.invokeLayoutCallback`\). Once building is complete, layout continues through the subtree and beyond.
* `CustomSingleChildLayout` allows its layout and its child’s layout to be customized without implementing a new render box. Deferring to a delegate, the associated `RenderCustomSingleChildLayout` invokes `SingleChildLayoutDelegate.getSize` to size the parent, then `SingleChildLayoutDelegate.getConstraintsForChild` to constrain the child, and finally `SingleChildLayoutDelegate.getPositionForChild` to position the child.
* `CustomMultiChildLayout` is similar, but supports multiple children. To identify children, each must be tagged with a `LayoutId`, a `ParentDataWidget` that stores an identifier in each child’s `MultiChildLayoutParentData` \(recall that render objects establish their children’s parent data type\). `MultiChildLayoutDelegate` behaves similarly to its single-child variant; override `MultiChildLayoutDelegate.performLayout` to size and position each child via `MultiChildLayoutDelegate.layoutChild` and `MultiChildLayoutDelegate.positionChild`, respectively \(the latter must be called once per child in any order\).
* `ConstrainedLayoutBuilder`
* `LayoutBuilder`
* `SliverLayoutBuilder`

