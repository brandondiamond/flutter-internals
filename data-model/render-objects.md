# Render Objects

## What are the render object building blocks?

* `RenderObject` provides the basic infrastructure for managing a tree of visual elements. Render objects define a general protocol for performing layout, painting, and compositing. This protocol is largely abstract, deferring to subclasses to determine the inputs and outputs of layout, how hit testing is performed \(though all render objects are `HitTestTargets`\), and how the render object hierarchy is modeled \(`RenderObject` extends `AbstractNode`, which doesn't specify a concrete child model\).
* `Constraints` represent the immutable inputs to layout. The definition is flexible provided that constraints can indicate whether they represent a single configuration \(`Constraints.isTight`\), are expressed in the canonical form \(`Constraints.isNormalized`\), and can be compared for equality \(via `Constraints.==` and `Constraints.hashCode`\).
* `ParentData` represents opaque data stored in a child by its parent. This data is typically considered an implementation detail of the parent and should therefore not be accessed by the child. Parent data is flexibly defined and only includes a single method \(`ParentData.detach`, which allows instances to react to their render object being removed from the tree\).

## How are render objects rendered?

* `RendererBinding` establishes a `PipelineOwner` as part of `RendererBinding.initInstances` \(i.e., the binding’s “constructor”\). The pipeline owner serves as the render tree's `AbstractNode.owner` \(set by `AbstractNode.attach` and propagated by `AbstractNode.adoptChild`\).
* `PipelineOwner` is analogous to the `BuildOwner`, tracking which render objects need compositing bits, layout, painting, or semantics updates. Objects mark themselves as being dirty by adding themselves to specific lists \(e.g.,`PipelineOwner._nodesNeedingLayout`, `PipeplineOwner._nodesNeedingPaint`\). All dirty objects are later cleaned by a corresponding “flush” method \(e.g, `PipelineOwner.flushLayout`, `PipelineOwner.flushPaint`\). These methods initiate the corresponding rendering process and are invoked every frame as needed \(via `RendererBinding.drawFrame` and `WidgetsBinding.drawFrame`\).
  * Rendering takes place by flushing dirty render objects in a specific sequence:
    * `PipelineOwner.flushLayout` lays out all dirty render objects \(laying out dirty children recursively, based on each `RenderObject.performLayout` implementation\).
    * `PipelineOwner.flushCompositingBits` updates a cache used to indicate whether descendant render objects create new layers. If so, because painting is often interlaced \(i.e., a render object paints before and after its children\), certain operations may need to be implemented differently. If all descendants paint into the same layer, an operation \(like clipping\) can be performed using a single painting context. If not, some effects can only be achieved by inserting new layers. These bits determine which option will be used when painting.
    * `PipelineOwner.flushPaint` paints all dirty render objects \(painting dirty children recursively, based on each `RenderObject.paint` implementation\).
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
* `RenderObject.layout` is invoked with a `Constraints` subclass as input; some subclasses incorporate other out-of-band input, too. The layout method applies input to produce geometry \(generally a size, though the type is unconstrained\).
  * If a protocol uses out-of-band input, and this input changes, the affected render objects must be marked dirty \(e.g., a parent that uses its child baseline to perform layout must repeat layout whenever the child's baseline changes\).
* If a parent depends on the child’s layout, it must pass the `parentUsesSize` argument to layout.
* `RenderObjects` that solely determine their sizing using the input constraints  set `RenderObject.sizedByParent` to true and perform all layout in `RenderObject.performResize`.
* Layout cannot depend on anything other than the incoming constraints; this includes the render object's position \(which is generally determined by the parent render object and stored in `ParentData`\). 

## How are render objects composited into layers?

* Compositing is the process by which painting \(via `RenderObject.paint`\) is divided into layers \(`Layer`\) before being rasterized and uploaded to the engine. Some render objects share the current layer whereas others introduce one or more new layers.
* A repaint boundary corresponds to a render object that always paints into a new `OffsetLayer` \(stored in `RenderObject.layer`\). This allows the subtree rooted at this node to paint separately from its parent. Moreover, the offset layer can be re-used \(and translated\) to avoid unnecessary repainting.
  * Other render objects that produce new layers can store the root-most layer in `RenderObject.layer` to allow future painting operations to update the existing layer rather than creating a new one.
  * This is possible because `Layer` preserves a reference to any underlying `EngineLayer`, and `SceneBuilder` accepts an `oldLayer` argument when building a new scene.
* Certain graphical operations are implemented differently depending on whether they're being applied within a single layer or across multiple layers \(e.g., clipping\).
  * Render objects often interlace their own painting with their children's painting. 
  * Therefore, to apply these graphical operations correctly, the framework must track which render objects introduce new layers into the scene.
* `needsCompositing` indicates whether the subtree rooted at a render object \(e.g., the object and its descendants\) may introduce new layers. This indicates to the `PaintingContext` that certain operations \(e.g., clipping\) will need to be implemented differently.
  * Render objects that always insert layers \(e.g., a video\) should override `alwaysNeedsCompositing`.
* The compositing bits should be marked dirty \(via `RenderObject.markNeedsCompositingBitsUpdate`\) whenever the render tree is mutated; children that are added may introduce layers whereas children that are removed may allow a single layer to be shared. `RenderObject.adoptChild` and `RenderObject.dropChild` always call this method.
  * All ancestors are marked dirty until reaching a repaint boundary or a node with a repaint boundary parent. In this case, all upstream objects will already have the correct compositing bits \(i.e., repaint boundary status cannot change; thus, all upstream compositing bits will have been configured when the render object was added\).
* Updates are performed by `PipelineOwner.flushCompositingBits` when the corresponding dirty list is non-empty \(`PipelineOwner._nodesNeedingCompositingBitesUpdate`\). The update must be applied before painting.
* Dirty render objects are sorted from back to front \(using `RenderObject.depth`\) then updated \(via `RenderObject._updateCompositingBits`\). This method performs a depth-first traversal of the render tree, marking each render object as needing compositing if any of its children need compositing \(or it's a repaint boundary or always needs compositing\).
  * If the compositing bits are updated, the node is marked as needing repainting, too.

## How do render objects manage painting?

* Render objects mark themselves as needing paint \(via`RenderObject.markNeedsPaint`\); this schedules a painting pass during the next frame.
* Painting is performed by `PipelineOwner.flushPaint` when the corresponding dirty list is non-empty \(`PipelineOwner._nodesNeedingPaint`\).
* Dirty render objects are sorted from back to front \(using `RenderObject.depth`\) then painted \(via `PaintingContext.repaintCompositedChild`\). This requires the render object to have an associated layer \(only render objects that are repaint boundaries are added to this list\).
  * If a render object's layer isn't attached \(i.e., isn't being composited\), painting must wait until it's reattached. The framework iterates through all ancestor render objects to find the nearest repaint boundary with an attached layer \(via `RenderObject._skippedPaintingOnLayer`\). Each is marked dirty to ensure that it's eventually painted.
* If necessary, a new `PaintingContext` is created for the render object and forwarded to `RenderObject._paintWithContext`.
* This method ensures that layout has been performed and, if so, invokes `RenderObject.paint` with the provided context.
* `RenderObject.paintBounds` provides an estimate of how large a region this render object will paint \(as a `Rect` in local coordinates\). This doesn't need to match the render object's layout geometry.
* `RenderObject.applyPaintTransform` captures how a render object transforms its children by applying the same transformation to a provided matrix \(e.x., if the child is painted at `(x, y)`, the matrix is translated by `(x, y)`\). This allows the framework to assemble a sequence of transformations to map between any two render object's coordinate systems \(via `RenderObject.getTransformTo`\). This process allows local coordinates to be mapped to global coordinates and vice versa.

## How do render objects manage hit testing?

* Render objects implement `HitTestTarget` and can therefore process incoming pointer events \(via `RenderObject.handleEvent`\). However, render objects do not support hit testing: there is no builtin mechanism for a render object to subscribe to a pointer. This functionality is introduced by `RenderBox`.
* The `GestureBinding` receives pointer events from the engine \(via `Window.onPointerDataPacket`\). These are queued \(via `GestureBinding._handlePointerDataPacket`\), which is flushed immediate \(via `GestureBinding._flushPointerEventQueue`\) or when events are unlocked \(via `GestureBinding.unlocked`, which is called by the `BaseBinding.lockEvents`\).
  * Events are locked during reassembly and the initial warmup frame.
* `GestureBinding._handlePointerEvent` processes each queued event in sequence using a table of hit test results \(`GestureBinding._hitTests`\) and a pointer router \(`GestureBinding._pointerRouter`\). 
  * The pointer router is a `PointerRouter` instance that maps an integral pointer ID to an event handler callback \(`PointerRoute`, a function that accepts a `PointerEvent`\). The router also captures a transform matrix to map from screen coordinates to local coordinates.
  * The hit testing table maps from an integral pointer ID to a record of all successful hits \(`HitTestResult`\). Each entry \(`HitTestEntry`\) records the recipient object \(`HitTestEntry.target`, a `HitTestTarget`\) and the transform needed to map from global coordinates to local coordinates \(`HitTestEntry.transform`\).
  * A `HitTestTarget` is an object that can handle pointer events \(via `HitTestTarget.handleEvent`\). Render objects are hit test targets.
* Initiating events \(`PointerUpEvent`, `PointerSignalEvent`\) trigger a hit test \(via `GestureBinding.hitTest`\). Ongoing events \(`PointerMoveEvent`\) are forwarded to the path associated with the initiating event. Terminating events \(`PointerUpEvent`, `PointerCancelEvent`\) remove the entry. 
  * Hit test results are stored in the hit test table so that they can be retrieved for subsequent events associated with the same pointer. Hit tests are performed on `HitTestable` objects \(via `HitTestable.hitTest`\). 
  * `GestureBinding` defines a default override that adds itself to the `HitTestResult`; invoking this also invokes any overrides mixed into the bindings. The binding processes events to resolve gesture recognizers using the gesture arena \(via `GestureBinding.handleEvent`\).
  * `RendererBinding` provides an override that tests the render view first \(via `RenderView.hitTest`, since `RenderView` is `HitTestable`\). This is the entry point for hit testing in the render tree.
  * The render view tests its render box child \(via `RenderBox.hitTest`\) before adding itself to the result.
  * `RenderBox` doesn't implement `HitTestable`, but does provide a compatible interface \(used by `RenderView`\). `RenderObject` is not hit testable and does not participate in hit testing by default.
* If the hit succeeds, the event is unconditionally forwarded to each `HitTestEntry` in the `HitTestResult` \(via `GestureBinding.dispatchEvent`\) in depth-first order.
  * Each entry along the path is mapped to a target which handles the event \(via `HitTestTarget.processEvent`\). As a `HitTestTarget`, render objects are eligible to process pointers. However, there is no mechanism for them to participate in hit testing.
  * Events are also forwarded via the pointer router to support gesture recognition, a more select event handling mechanism.

## How do render objects handle transformations?

* Visual transforms are implemented in `RenderObject.paint`. The same transform is applied logically \(i.e., when hit testing, computing semantics, mapping coordinates, etc.\) via `RenderObject.applyPaintTransform`. This method applies the child transformation to the provided matrix \(`providedMatrix * childTransform`\).
* `RenderObject.transformTo` chains paint transformation matrices mapping from the current coordinate space to an ancestor’s coordinate space. 
  * If an ancestor isn't provided, the transformation will map to global coordinates \(i.e., the coordinates of the render object just below the `RenderView`; to obtain physical screen coordinates, these coordinates must be further transformed by `RenderView.applyPaintTransform`\). 

## How does the view affect rendering?

* `RenderView` receives a `ViewConfiguration` from the platform that specifies the screen's dimensions \(`ViewConfiguration.size`\) and the pixel depth \(`ViewConfiguration.devicePixelRatio`\).
* `RenderView.performLayout` adopts this size \(`RenderView.size`\)  and, if present, lays out the render box child with tight constraints corresponding to the screen's dimensions \(`ViewConfiguration.size`\).
* `RenderView.paint` and `RenderView.hitTest` delegate to the child, if present; the render view always adds itself to the hit test result.
* When the configuration is set \(via `RenderView.configuration`\), the layer tree's root is replaced with a `TransformLayer` \(via `RenderView._updateMatricesAndCreateNewRootLayer`\) that converts physical pixels to logical pixels. This allows all descendant render objects to be isolated from changes to screen resolution.
* `RenderView.applyPaintTransform` applies the resolution transform \(`RenderView._rootTransform`\) to map from logical global coordinates to physical screen coordinates.
* `RenderView.compositeFrame` implements the final stage of the rendering pipeline, compositing all painted layers and uploading the resulting scene to the engine for rasterization \(via `Window.render`\).
  * If system UI customization is enabled \(`RenderView.automaticSystemUiAdjustment`\), `RenderView._updateSystemChrome` will query the layer tree to find `SystemUiOverlayStyle` instances associated with the very top and very bottom of the screen \(this is the mechanism used to determine how to tint system UI like the status bar or Android's bottom navigation bar\). If found, the appropriate settings are communicated to the platform \(using `SystemChrome` platform methods\).

## What protocols are implemented in the framework?

* `RenderBox`, `RenderSliver`.

