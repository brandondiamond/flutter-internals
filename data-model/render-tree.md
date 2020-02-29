# Render Tree

## What are the render object building blocks?

* `RenderObject` provides the basic infrastructure for modeling hierarchical visual elements in the UI and includes a generalized protocol for performing layout, painting, and compositing. This protocol is unopinionated, deferring to subclasses to determine the inputs and outputs of layout, how hit testing is performed \(though all render objects are `HitTestTargets`\), and even how the render object hierarchy is modeled.
* Constraints represent the immutable inputs to layout. They must indicate whether they offer no choice \(`Constraints.isTight`\), are the canonical representation \(`Constraints.isNormalized`\), and provide “==” and `hashCode` implementations.
* `ParentData` is opaque data stored in a child `RenderObject` by its parent. The only requirement is that `ParentData` instances implement a `ParentData.detach` method to react to their render object being removed from the tree.

## How is the render object lifecycle managed?

* The `RendererBinding` establishes a `PipelineOwner` as part of `RendererBinding.initInstances` \(i.e., the binding’s “constructor”\). The `PipelineOwner` is analogous to the `BuildOwner`, tracking which render objects need compositing bits, layout, painting, and semantics updates. Objects mark themselves “dirty” by adding themselves to lists, e.g., `PipelineOwner._nodesNeedingLayout`, `PipeplineOwner._nodesNeedingPaint`, etc. All dirty objects are later cleaned by the corresponding “flush” method: `PipelineOwner.flushLayout`, `PipelineOwner.flushPaint`, etc. These methods form the majority of the rendering pipeline and are invoked every frame via `RendererBinding.drawFrame`.
* Whenever an object marks itself dirty, it will generally also invoke `PipelineOwner.requestVisualUpdate` to schedule a frame and update the UI. This calls the `PipelineOwner.onNeedVisualUpdate` handler which schedules a frame via `RendererBinding.ensureVisualUpdate` and `SchedulerBinding.scheduleFrame`. The rendering pipeline \(as facilitated by `RendererBinding.drawFrame` or `WidgetsBinding.drawFrame`\) will invoke the `PipelineOwner`’s various flush methods to drive each stage.
* When the render object is attached to its `PipelineOwner`, it will mark the render object as dirty in all necessary dimensions \(e.g., `RenderObject.markNeedsLayout`, `RenderObject.markNeedsPaint`\). Generally, all are invoked since the internal flags that are consulted are initialized to true \(e.g., `RenderObject._needsLayout`, `RenderObject._needsPaint`\).

## How is the render tree structured?

* The render tree consists of `AbstractNode` instances \(`RenderObject` is a subclass\). When a child is added or removed, `RenderObject.adoptChild` and `RenderObject.dropChild` must be called, respectively. When altering the depth of a child, `RenderObject.redepthChild` must be called to keep `RenderObject.depth` in sync.
* Every node also needs to use `RenderObject.attach` and `RenderObject.detach` to set and clear the pipeline owner \(`RenderObject.owner`\), respectively. Every `RenderObject` has a parent \(`RenderObject.parent`\) and an attached state \(`RenderObject.attached`\).
* Parent data is a `ParentData` subclass optionally associated with a render object. By convention, the child is not to access this data, though a protocol is free to alter this rule \(but shouldn’t\). When adding a child, the parent data is initialized by calling `RenderObject.setupParentData`. The parent data may then be mutated in, e.g., `RenderObject.performLayout`.
* All `RenderObjects` implement the visitor pattern via `RenderObject.RenderObject.visitChildren` which invokes a `RenderObjectVisitor` once per child.

## What is necessary when altering the child model?

* When adding or removing a child, call `RenderObject.adoptChild` and `RenderObject.dropChild` respectively.
* The parent’s `RenderObject.attach` and `RenderObject.detach` methods must call their counterpart on each child.
* The parent’s `RenderObject.redepthChildren` and `RenderObject.visitChildren` methods must recursively call `RenderObject.redepthChild` and the visitor argument on each child, respectively.

## What are the render tree building blocks?

* `RenderObjectWithChildMixin` allocates storage for a single child within the object instance itself \(`RenderObjectWithChildMixin.child`\).
* `ContainerRenderObjectMixin` uses each child’s parent data \(which must implement `ContainerParentDataMixin`\) to build a doubly linked list via `ContainerParentDataMixin.nextSibling` and `ContainerParentDataMixin.previousSibling`.
  * A variety of container-type methods are included: `ContainerRenderObjectMixin.insert`, `ContainerRenderObjectMixin.add`, `ContainerRenderObjectMixin.move`, `ContainerRenderObjectMixin.remove`, etc.
  * These adopt, drop, attach, and detach children appropriately. Additionally, when the child list changes, `RenderObject.markNeedsLayout` is invoked \(as this may alter layout\).

## How do render objects manage layout?

* When a render object has been marked as dirty via `RenderObject.markNeedsLayout`, `RenderObject.layout` will be invoked with constraints as input and a size as output. Both the constraints and the size are implementation-dependent; the constraints must implement `Constraints` whereas the size is entirely arbitrary.
* If a parent depends on the child’s geometry, it must pass the `parentUsesSize` argument to layout. The implementation of `RenderObject.markNeedsLayout` / `RenderObject.sizedByParent` will need to call `RenderObject.markParentNeedsLayout` / `RenderObject.markParentNeedsLayoutForSizedByParentChange`, respectively.
* `RenderObjects` that solely determine their sizing using the input constraints must set `RenderObject.sizedByParent` to true and perform all layout in `RenderObject.performResize`.
* Layout cannot depend on position \(typically stored using parent data\). The position, if applicable, is solely determined by the parent. However, some `RenderObject` subtypes may utilize additional out-of-band information when performing layout. If this information changes, and the parent used it during the last cycle, `RenderObject.markParentNeedsLayout` must be invoked.

## How do render objects manage hit testing?

* `RenderView`’s child is a `RenderBox`, which must therefore define `RenderBox.hitTest`. To add a custom `RenderObject` to the render tree, the top level `RenderView` must be replaced \(which would be a massive undertaking\) or a `RenderBox` adapter added to the tree. It is up to the implementer to determine how `RenderBox.hitTest` is adapted to a custom `RenderObject` subclass; indeed, that render object can implement any manner of `hitTest`-like method of its choosing. Note that all `RenderObjects` are `HitTestTargets` and therefore will receive pointer events via `HitTestTarget.pointerEvent` once registered via `HitTestEntry`.

## How are render objects composited into layers?

* Render objects track a “`needsCompositing`” bit \(as well as an “`alwaysNeedsCompositing`” bit and a “`isRepaintBoundary`” bit, which are consulted when updating “`needsCompositing`”\). This bit indicates to the framework that a render object will paint into its own layer.
* This bit is not set directly. Instead, it must be marked dirty whenever it might possibly change \(e.g., when adding or removing a child\). `RenderObject.markNeedsCompositingBitsUpdate` walks up the render tree, marking objects as dirty until it reaches a previously dirtied render object or a repaint boundary.
* Later, just before painting, `PipelineOwner.flushCompositingBits` visits all dirty render objects, updating the “`needsCompositing`” bit by walking down the tree, looking for any descendent that needs compositing. This walk stops upon reaching a repaint boundary or an object that always needs compositing. If any descendent node needs compositing, all nodes along the walk need compositing, too.
* It is important to create small “repaint sandwiches” to avoid introducing too many layers. \[?\]
* Composited render objects are associated with an optional `ContainerLayer`. For render objects that are repaint boundaries, `RenderObject.layer` will be `OffsetLayer`. In either case, by retaining a reference to a previously used layer, `Flutter` is able to better utilize retained rendering -- i.e., recycle or selectively update previously rasterized bitmaps. This is possible because `Layers` preserve a reference to any underlying `EngineLayer`, and `SceneBuilder` accepts an “`oldLayer`” argument when building a new scene.

## How do render objects handle transformations?

* Visual transforms are implemented in `RenderObject.paint`. The same transform is applied logically \(i.e., when hit testing, computing semantics, mapping coordinates, etc\) via `RenderObject.applyPaintTransform`. This method applies the child transformation to the provided matrix.
* `RenderObject.transformTo` chains paint transformation matrices mapping from the current coordinate space to the provided ancestor’s coordinate space. \[`WIP`\]

## What other protocols are implemented in the framework?

* `RenderBox`, `RenderSliver`.

