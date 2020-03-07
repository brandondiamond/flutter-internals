# Render Tree

## What are the render object building blocks?

* `RenderObject` provides the basic infrastructure for managing a tree of visual elements. Render objects define a general protocol for performing layout, painting, and compositing. This protocol is largely abstract, deferring to subclasses to determine the inputs and outputs of layout, how hit testing is performed \(though all render objects are `HitTestTargets`\), and how the render object hierarchy is modeled \(`RenderObject` extends `AbstractNode`, which doesn't specify a concrete child model\).
* `Constraints` represent the immutable inputs to layout. The definition is flexible provided that constraints can indicate whether they represent a single configuration \(`Constraints.isTight`\), are expressed in the canonical form \(`Constraints.isNormalized`\), and can be compared for equality \(via `Constraints.==` and `Constraints.hashCode`\).
* `ParentData` represents opaque data stored in a child by its parent. This data is typically considered an implementation detail of the parent and should therefore not be accessed by the child. Parent data is flexibly defined and only includes a single method \(`ParentData.detach`, which allows instances to react to their render object being removed from the tree\).

## How are render objects rendered?

* `RendererBinding` establishes a `PipelineOwner` as part of `RendererBinding.initInstances` \(i.e., the binding’s “constructor”\). The pipeline owner serves as the render tree's `AbstractNode.owner` \(set by `AbstractNode.attach` and propagated by `AbstractNode.adoptChild`\).
* `PipelineOwner` is analogous to the `BuildOwner`, tracking which render objects need compositing bits, layout, painting, or semantics updates. Objects mark themselves as being dirty by adding themselves to specific lists \(e.g.,`PipelineOwner._nodesNeedingLayout`, `PipeplineOwner._nodesNeedingPaint`\). All dirty objects are later cleaned by a corresponding “flush” method \(e.g, `PipelineOwner.flushLayout`, `PipelineOwner.flushPaint`\). These methods initiate the corresponding rendering process and are invoked every frame as needed \(via `RendererBinding.drawFrame` and `WidgetsBinding.drawFrame`\).
  * Rendering takes place by flushing dirty render objects in a specific sequence:
    * `PipelineOwner.flushLayout` lays out all dirty render objects \(laying out dirty children recursively based on the `RenderObject.performLayout` implementation\).
    * `PipelineOwner.flushCompositingBits` updates a cache used to indicate whether descendant render objects create new layers. If so, because painting is often interlaced \(i.e., a render object paints before and after its children\), certain operations may need to be implemented differently. If all descendants paint into the same layer, an operation \(like clipping\) can be performed using a single painting context. If not, some effects can only be achieved by inserting new layers. These bits determine which option will be used when painting.
    * `PipelineOwner.flushPaint` paints all dirty render objects \(painting dirty render objects recursively, based on the `RenderObject.paint` implementation\).
    * `PipelineOwner.flushSemantics` compiles semantic data for all dirty render objects.
* Whenever an object marks itself dirty, it will generally also invoke `PipelineOwner.requestVisualUpdate` to schedule a frame and eventually update the user interface. Requesting a visual update invokes the visual update handler that was bound when the `PipelineOwner` was constructed by `RendererBinding` \(`PipelineOwner.onNeedVisualUpdate`\) 
* The default handler invokes`RendererBinding.ensureVisualUpdate` which schedules a frame \(via`SchedulerBinding.scheduleFrame`\) if it's not possible to apply the visual update during the current frame \(e.g., during post frame callbacks\). 
* When a render object is attached to the `PipelineOwner` \(via `RenderObject.attach`\), it adds itself to all relevant dirty lists \(e.g., `RenderObject.markNeedsLayout`, `RenderObject.markNeedsPaint`\).
  * Typically, a newly created render object will be dirty at this point since most of the dirty bits are initialized to true \(e.g., `RenderObject._needsLayout`, `RenderObject._needsPaint`\). Several other bits depend on the configuration of the node \(e.g., `RenderObject._needsCompositing`\) but will be true if they're applicable.
  * The render object may also be dirty if it had been previously detached, mutated, and is now being re-attached.
  * As an exception, the very first painting, layout, and semantics passes are driven by a different flow \(via `RenderObject.scheduleInitialLayout`, `RenderObject.scheduleInitialPaint`, and `RenderObject.scheduleInitialSemantics`\). This flow occurs when the `RenderView` is first constructed \(via `RendererBinding.initRenderView`\). This process calls `RenderView.prepareInitialFrame`, which schedules initial layout and painting for the `RenderView` \(semantics follows a slightly different flow\). Marking the root node as being dirty causes the entire tree to perform layout, painting, and semantics as needed.
* During a hot reload, the renderer invokes `RenderView.reassemble` \(via `RendererBinding.performReassemble`\) which cascades down the render tree. By default, render objects mark themselves as being dirty for layout, compositing bits, painting, and semantics \(via `RenderObject.reassemble`\).

## How is the render tree structured?

* The render tree is composed of `RenderObject` instances, a subclass of `AbstractNode`. 
  * `RenderObject.adoptChild` or `RenderObject.dropChild` must be called whenever a child is added or removed, respectively. Modifying the child model marks the subtree dirty for layout, painting, semantics, and compositing bits.
  * `RenderObject.redepthChild` must be called whenever a child's depth changes.
  * Render objects must be attached to the `PipelineOwner` \(via`RenderObject.attach`\) and detached when they should no longer be rendered \(via `RenderObject.detach`\). 
    * `PipelineOwner` maintains a reference to a root node \(`PipelineOwner.rootNode`\). This field is set to the application's `RenderView` when initializing the `RendererBinding` \(via `RendererBinding.initRenderView` which sets `RendererBinding.renderView` and, via a setter, `PipelineOwner.rootNode`\). This automatically detaches the old render object \(if any\) and attaches the new one.
    * Children are automatically attached and detached when they are adopted or dropped.
    * However, since parents may be attached and detached arbitrarily, render objects are responsible for calling the corresponding method on their children.
  * Subclasses that introduce a child model \(e.g., `ContainerRenderObjectMixin`\) must override `RenderObject.redepthChildren` to call `RenderObject.redepthChild` on each child.
  * Render objects have a parent \(`RenderObject.parent`\) and an attached state \(`RenderObject.attached`\), indicating whether they'll contribute to rendering.
* Parent data may be associated with a child using a `ParentData` subclass; this data may only be assigned using `RenderObject.setupParentData`. By convention, this data is considered opaque to the child, though a protocol is free to alter this rule \(but shouldn’t\). The parent data may then be mutated \(e.g., via `RenderObject.performLayout`\). 
  * `ContainerRenderObjectMixin` uses this field to implement a linked list of children.
* All `RenderObjects` implement the visitor pattern via `RenderObject.RenderObject.visitChildren` which invokes a `RenderObjectVisitor` once per child.

## What is necessary when adding or removing children?

* When adding or removing a child, call `RenderObject.adoptChild` and `RenderObject.dropChild` respectively.
* The parent’s `RenderObject.attach` and `RenderObject.detach` methods must call their counterpart on each child.
* The parent’s `RenderObject.redepthChildren` and `RenderObject.visitChildren` methods must recursively call `RenderObject.redepthChild` and the visitor argument on each child, respectively.

## What are the render tree building blocks?

* `RenderObjectWithChildMixin` stores a single child reference within a render object.
  * A child setter \(`RenderObjectWithChildMixin.child`\) adopts and drops the child as appropriate \(i.e., when adding and removing a child\).
  * Updating the child marks the render object as needing layout and therefore painting.
  * Attaching, detaching, recomputing depths, and iterating are overridden to incorporate the child.
* `ContainerRenderObjectMixin` uses each child’s parent data \(which must implement `ContainerParentDataMixin`\) to maintain a doubly linked list of children \(via `ContainerParentDataMixin.nextSibling` and `ContainerParentDataMixin.previousSibling`\).
  * The underlying linked list supports insertions and removals \(via `ContainerParentDataMixin._insertIntoChildList` and `ContainerParentDataMixin._removeFromChildList`\). Insertions can be positional using an optional preceding child argument.
  * A variety of container-type methods are exposed \(e.g., `ContainerRenderObjectMixin.insert`, `ContainerRenderObjectMixin.add`, `ContainerRenderObjectMixin.move`, `ContainerRenderObjectMixin.remove`\). These adopt and drop children as needed. A move operation is provided to avoid unnecessarily dropping and re-adopting a child.
  * Updating \(or reordering\) the child list marks the render object as needing layout and therefore painting.
  * These adopt, drop, attach, and detach children appropriately. Additionally, when the child list changes, `RenderObject.markNeedsLayout` is invoked \(as this may alter layout\).
  * Attaching, detaching, recomputing depths, and iterating are overridden to incorporate all children.
  * The first child \(`ContainerRenderObjectMixin.firstChild`\), last child \(`ContainerRenderObjectMixin.lastChild`\), and total number of children \(`ContainerRenderObjectMixin.childCount`\) is accessible via the render object.

## How do render objects manage layout?

* Render objects mark themselves as needing layout \(via`RenderObject.markNeedsLayout`\); this schedules a layout pass during the next frame.
* Layout is performed by `PipelineOwner.flushLayout` when the corresponding dirty list is non-empty \(`PipelineOwner._nodesNeedingLayout`\). 
* `RenderObject.layout` is invoked with a `Constraints` subclass as input; some subclasses incorporate other out-of-band input, too. The layout method applies input to produce an arbitrary output \(generally a size, though the type is unconstrained\).
  * If a protocol uses out-of-band input, and this input changes, the affected render objects must be marked dirty \(e.g., a parent that uses its child baseline to perform layout must repeat layout whenever the child's baseline changes\).
* If a parent depends on the child’s layout, it must pass the `parentUsesSize` argument to layout.
* `RenderObjects` that solely determine their sizing using the input constraints  set `RenderObject.sizedByParent` to true and perform all layout in `RenderObject.performResize`.
* Layout cannot depend on anything other than the incoming constraints; this includes the render object's position \(which is generally determined by the parent render object and stored in `ParentData`\). 

## How do render objects manage painting?

* Render objects mark themselves as needing paint \(via`RenderObject.markNeedsPaint`\); this schedules a painting pass during the next frame.
* Painting is performed by `PipelineOwner.flushPaint` when the corresponding dirty list is non-empty \(`PipelineOwner._nodesNeedingPaint`\).
* Dirty render objects are sorted from back to front \(using `RenderObject.depth`\) then painted \(via `PaintingContext.repaintCompositedChild`\). This requires the render object to have an associated layer \(only render objects that are repaint boundaries are added to this list\).
  * If a render object's layer isn't attached \(i.e., isn't being composited\), painting must wait until it's reattached. The framework iterates through all ancestor render objects to find the nearest repaint boundary with an attached layer \(via `RenderObject._skippedPaintingOnLayer`\). Each is marked dirty to ensure that it's eventually painted.
* If necessary, a new `PaintingContext` is created for the render object and forwarded to `RenderObject._paintWithContext`.
* This method ensures that layout has been performed and, if so, invokes `RenderObject.paint` with the provided context.

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

