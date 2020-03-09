# Boxes

## What are the render box building blocks?

* `RenderBox` models a box in 2D cartesian coordinates with a width, height, and offset. The box's top-left corner defines its origin, with the bottom-right corner corresponding to `(width, height)`.
* `BoxParentData` stores the child's offset in the parent’s coordinate space \(`BoxParentData.offset`\). By convention, this data may not be accessed by the child.
* `ContainerBoxParentData` extends `BoxParentData` with `ContainerParentDataMixin`. This combines the child offset with next and previous pointers \(`ContainerParentData.previousSibling`, `ContainerParentData.nextSibling`\) to support a doubly linked list of children.
* `BoxConstraints` describes immutable constraints in the Cartesian plane expressed as a maximum and minimum width and height ranging from zero to infinity, inclusive. Constraints are satisfied by concrete sizes that fall within this range.
  * Box constraints are classified in several ways:
    * `BoxConstraints.isNormal`: minimum is greater than zero and less than or equal to the maximum in both dimensions.
    * `BoxConstraints.tight, BoxConstraints.isTight`: minimum and maximum values are equal in both dimensions.
    * `BoxConstraints.loose`: minimum value is zero, even if maximum is also zero \(i.e., loose and tight\).
    * `BoxConstraints.hasBoundedWidth`, `BoxConstraints.hasBoundedHeight`: the corresponding maximum is not infinite.
      * Constraints are unbounded when the maximum is infinite.
    * `BoxConstraints.expanding`:  both maximum and minimum values are infinite in the same dimension \(i.e., tightly infinite\).
      * Expanding constraints imply that the corresponding dimension will be determined by other incoming constraints \(e.g., established by containing UI\). Dimensions must ultimately be finite.
    * `BoxConstraints.hasInfiniteWidth`, `BoxConstraints.hasInfiniteHeight`: the corresponding minimum is infinite \(thus, the maximum must also be infinite; this is the same as expanding\).
  * Box constraints can be evaluated \(`BoxConstraints.isSatisfiedBy`\), applied to a `Size` \(`BoxConstraints.constrain`\), tightened relative to constraints \(`BoxConstraints.tighten`\), loosened by setting minimums to zero \(`BoxConstraints.loosen`\), and intersected \(`BoxConstraints.enforce`\). Constraints can also be scaled standard algebraic operators.
* `BoxHitTestResult` is a `HitTestResult` subclass that captures each `RenderBox` \(a`HitTestTarget`\) that reported being hit in order of decreasing precedence.
  * Instances include box-specific helpers intended to help transform global coordinates to local coordinates \(e.g., `BoxHitTestResult.addWithPaintOffset`, `BoxHitTestResult.addWithPaintTransform`\).
* `BoxHitTestEntry` represents a box that was hit during hit testing. It captures the position of the collision in local coordinates \(`BoxHitTestEntry.localPosition`\).

## How do boxes model children?.

* `RenderObjectWithChildMixin` and `ContainerRenderObjectMixin` can both be used with a type argument of `RenderBox`. The `ContainerRenderObjectMixin` accepts a parent data type argument; `ContainerBoxParentData` combines `BoxParentData` with the `ContainerParentDataMixin` to satisfy the type constraint.
* `RenderBoxContainerDefaultsMixin` adds useful defaults to `ContainerRenderObjectMixin` for render boxes with children. Children must extend `RenderBox` and parent type must extend `ContainerBoxParentData`.
  * This includes support for hit testing \(`RenderBoxContainerDefaultsMixin.defaultHitTestChildren`\), painting \(`RenderBoxContainerDefaultsMixin.defaultPaint`\), and listing \(`RenderBoxContainerDefaultsMixin.getChildrenAsList`\) children.
* `RenderProxyBox` delegates all methods to a single `RenderBox` child. Proxies are useful for layering new functionality on top of a render box without altering layout or painting.
* `RenderShiftedBox` delegates all methods to a single `RenderBox` child without providing a default layout implementation. Subclasses override `RenderShiftedBox.performLayout` to assign a position to the child \(via `BoxParentData.offset`\). 

## How do render boxes handle layout?

* Boxes use the standard `RenderObject` layout protocol to map from incoming `BoxConstraints` to a concrete `Size` \(stored in `RenderBox.size`\). Layout typically sets the child's offset relative to the parent, too \(`BoxParentData.offset`\). This information may not be read by the child during layout.
* Boxes add support for intrinsic dimensions \(an ideal size, computed outside of the standard layout protocol\) as well as baselines \(a line to use for vertical alignment, typically only used when laying out text\). `RenderBox` instances track changes to these values and whether the parent has queried them; if so, when that box is marked as needing layout, the parent is marked as well.
  * Intrinsic dimensions are cached \(`RenderBox._cachedIntrinsicDimensions`\) whenever they're computed \(via `RenderBox._computeIntrinsicDimension`\). The cache is disabled during debugging.
  * Baselines are cached \(`RenderBox._cachedBaselines`\) whenever they're computed \(via `RenderBox.getDistanceToActualBaseline`\). Different baselines are computed for alphabetic and ideographic text.
* `RenderBox.markNeedsLayout` marks the parent as needing layout if the box's intrinsic dimension or baseline caches have been populated. If so, both are cleared so that new values are computed after the next layout pass.
* By default, boxes that are sized by their parents adopt the smallest possible size given the incoming constraints \(via `RenderBox.performResize`\).
* Render boxes can have non-box children. In this case, the constraints passed from the parent to the child will need to be adapted from `BoxConstraints` to the appropriate type.

## How do render boxes handle painting?

* `RenderBox.paintBounds` describes the region that will be painted by a box. This determines the size of the buffer used for painting and is expressed in local coordinates. It need not match `RenderBox.size`.
* Painting is the same as `RenderObject` painting. The offset provided corresponds to the origin of the render object in the canvas’s coordinate space \(which may not be the same\).
* If the render box applies a transform when painting, including painting at a different offset than the one provided, `RenderBox.applyPaintTransform` must apply the same transformation. `RenderBox.globalToLocal` and `RenderBox.localToGlobal` rely on this transform to map cartesian coordinates.
  * By default, `RenderBox.applyPaintTransform` will apply the child’s offset to the matrix as a translation.

## How do render boxes handle hit testing?

* All `RenderBoxes` must implement `RenderBox.hitTest`, which by default delegates to `RenderBox.hitTestChildren` and `RenderBox.hitTestSelf`, in that order. Note that items added to the `BoxHitTestResult` first are treated as being on top of those added later. All entries in the `BoxHitTestResult` are fed events from the corresponding event stream via `RenderBox.handleEvent`.

## What are intrinsic dimensions?

* Conceptually, the intrinsic dimensions of a box are its natural dimensions -- the size it “wants” to be. The precise definition depends on the semantics of the box in question. Note that intrinsic dimensions are often defined in terms of child intrinsic dimensions and are therefore expensive to calculate \(typically traversing an entire subtree\).
* Intrinsic dimensions often do not match the dimensions resulting from layout \(except when using the `IntrinsicHeight` and `IntrinsicWidth` widgets, which attempt to layout their child using its intrinsic dimensions\).
  * Intrinsic dimensions are generally ignored unless one of these widgets are used \(or another widget that explicitly incorporates intrinsic dimensions into its own layout\).
* The render box model describes intrinsic dimensions in terms of minimum and maximum values for width and height.
  * Minimum intrinsic width is the smallest width before the box cannot paint correctly without clipping.
    * Intuition: making the box thinner would clip its contents.
    * If width is determined by height \(ignoring constraints\), the incoming height \(which may be infinite, i.e., unconstrained\) should be used. Otherwise, ignore the height.
  * Minimum intrinsic height is the same concept for height.
  * Maximum intrinsic width is the smallest width such that further expansion would not reduce minimum intrinsic height \(for that width\).
    * Intuition: making the box wider won’t help fit more content.
    * If width is determined by height \(ignoring constraints\), the incoming height \(which may be infinite, i.e., unconstrained\) should be used. Otherwise, ignore the height.
  * Maximum intrinsic height is the same concept for height.
* The specific meaning of intrinsic dimensions depends on the implementation.
  * Text is width-in-height-out.
    * Max intrinsic width: the width of the string without line breaks \(increasing the width would not shrink the preferred height\).
    * Min intrinsic width: the width of the widest word \(decreasing the width would clip the word or cause an invalid break\).
    * Height intrinsics derived from width intrinsics for the given width.
  * Viewports ignore incoming constraints and aggregate child dimensions without clipping.
  * Aspect ratio boxes are entirely determined when one dimension is provided.
    * Use the incoming dimension to compute the other. If unconstrained, recurse.
  * When intrinsic dimensions cannot be computed or are too expensive, return zero.

## What are baselines?

* Baselines are used to vertically align children independent of font size, padding, etc. When a real baseline isn’t available, the bottom of the render box is used. Baselines are expressed as an offset from the box’s y-coordinate; if there are multiple children, the first sets the baseline.
* Only a render box’s parent can invoke `RenderBox.getDistanceToBaseline`, and only after that render box has been laid out in the parent’s `RenderBox.performLayout` method.

