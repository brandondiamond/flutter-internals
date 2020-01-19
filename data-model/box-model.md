# Box Model

* What are the render box building blocks?
  * RenderBox models a box in 2D cartesian coordinates with a width, height, and offset. The origin of the box is the top-left corner \(0,0\), with the bottom-right corner corresponding to \(width, height\).
  * BoxParentData stores a child’s position in the parent’s coordinate space. It is opaque to the child by convention.
  * BoxConstraints provide cartesian constraints in the form of a maximum and minimum width and height between 0 and infinity, inclusive. Constraints are satisfied for all dimensions simultaneously within the given inclusive range.
    * Constraints are normalized when min is less or equal to max.
    * Constraints are tight when max and min are equal.
    * Constraints are loose when min is 0, even if max is also 0 \(loose and tight\).
    * Constraints are bounded when max is not infinite.
    * Constraints are unbounded when max is infinite.
    * Constraints are expanding when tightly infinite.
    * Constraints are infinite when min is infinite \(max must also be infinite; thus, this is also expanding\).
  * BoxHitTestResult is an aggregator that collects the entries associated with a single hit test query. BoxHitTestResult includes several box-specific helpers, i.e., BoxHitTestResult.addWithPaintOffset, BoxHitTestResult.addWithPaintTransform, etc.
  * BoxHitTestEntry represents a box that was hit during hit testing. It captures the position of the collision in local coordinates \(BoxHitTestEntry.localPosition\).
* What are the render box tree building blocks? 
  * RenderProxyBox supports a single RenderBox child, delegating all methods to said child. RenderShiftedBox is identical, but offsets the child using BoxParentData.offset and requires a layout implementation. Both are built with RenderObjectWithChildMixin.
  * RenderObjectWithChildMixin and ContainerRenderObjectMixin can both be used with a type argument of RenderBox. The ContainerRenderObjectMixin accepts a parent data type argument, as well; ContainerBoxParentData satisfies the type constraints and supports box parent data.
  * RenderBoxContainerDefaultsMixin adds useful defaults to ContainerRenderObjectMixin for render boxes. This includes support for hit testing and painting children, as well as computing baselines.
* How do render boxes handle layout?
  * RenderBox layout is the same as RenderObject layout, with the input being BoxConstraints and the output being RenderBox.size \(Size\). Layout typically also sets the position \(BoxParentData.offset\) of any children, though the child may not read this data.
  * The RenderBox protocol includes support for intrinsic dimensions \(a size that is computed outside of the layout protocol\) as well as baselines \(the bottom of the box for vertical alignment\). RenderBox instances track changes to these values and whether the parent has queried them; if so, when that box is marked as needing layout, the parent is marked as well.
  * Marking a render box as needing layout implies that the box needs paint. Building a render box implies both.
  * Render boxes can have non-box children. In this case, the constraints passed from the parent to the child will need to be adapted from BoxConstraints to the new protocol.
* How do render boxes handle paint?
  * Painting is the same as RenderObject painting. The offset provided corresponds to the origin of the render object in the canvas’s coordinate space \(which may not be the same\).
  * If the render box applies a transform when painting, including painting at a different offset than the one provided, RenderBox.applyPaintTransform must apply the same transformation. RenderBox.globalToLocal and RenderBox.localToGlobal rely on this transform to map cartesian coordinates.
    * By default, RenderBox.applyPaintTransform will apply the child’s offset to the matrix as a translation.
* How do render boxes handle hit testing?
  * All RenderBoxes must implement RenderBox.hitTest, which by default delegates to RenderBox.hitTestChildren and RenderBox.hitTestSelf, in that order. Note that items added to the BoxHitTestResult first are treated as being on top of those added later. All entries in the BoxHitTestResult are fed events from the corresponding event stream via RenderBox.handleEvent.
* What are intrinsic dimensions?
  * Conceptually, the intrinsic dimensions of a box are its natural dimensions -- the size it “wants” to be. Given that this definition is subjective, it is left as an implementation detail what exactly this means for a given render object. Note that intrinsic dimensions are often defined in terms of child intrinsic dimensions and are therefore expensive to calculate \(typically traversing an entire subtree\).
  * Intrinsic dimensions often do not match the dimensions resulting from layout \(except when using the IntrinsicHeight and IntrinsicWidth widgets, which attempt to layout their child using its intrinsic dimensions\).
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
* What are baselines?
  * Baselines are used to vertically align children independent of font size, padding, etc. When a real baseline isn’t available, the bottom of the render box is used. Baselines are expressed as an offset from the box’s y-coordinate; if there are multiple children, the first sets the baseline.
  * Only a render box’s parent can invoke RenderBox.getDistanceToBaseline, and only after that render box has been laid out in the parent’s RenderBox.performLayout method.

