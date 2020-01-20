# Widgets


## What are the widget building blocks?

* The main entry points are `RenderObjectWidget`, `StatefulWidget`, and `StatelessWidget`. Widgets that export data to one or more descendant widgets \(via notifications or another mechanism\) utilize `ProxyWidget` or its subclasses, `InheritedWidget` and `ParentDataWidget`. 
* In general, widgets either directly or indirectly configure render objects; these indirect “component” widgets \(e.g., stateful and stateless widgets\) participate in a building process, whereas render object widgets manage an associated render object directly \(creating it and updating it via `RenderObjectWidget.createRenderObject` and `RenderObjectWidget.updateRenderObject`, respectively\).
* Certain widgets wrap \(i.e., reproject\) a child widget \(`ProxyWidget`\), introducing heritable state \(`InheritedWidget`, `InheritedModel`\) or setting parent data used by an enclosing widget \(`ParentDataWidget`\).
  * `ProxyWidget` notifies clients \(via `ProxyElement.notifyClients`\) in response to widget changes \(via `ProxyElement.updated`, called by `ProxyElement.update`\).
  * `ParentDataWidget` utilizes this flow to update the first descendant render object’s parent data \(via `ParentDataElement.\_applyParentData`, which calls `RenderObjectElement.\_updateParentData`\); this will be triggered any time the widget is updated.
* `PreferredSizeWidget` is not widely used but an interesting example of adapting widgets to a specific need \(e.g., an `AppBar` expressing a size preference\).
* Numerous helper subclasses exist, most notably including `SingleChildRenderObjectWidget`, `MultiChildRenderObjectWidget`, `LeafRenderObjectWidget`. These provide storage for a child / children without implementing the underlying child model.
* Anonymous widgets can be created using Builder and `StatefulBuilder`.

## How does building work?

* Every widget is associated with an element, a mutable object corresponding to the widget’s instantiation within the tree. Only those widgets associated with `ComponentElement` \(e.g., `StatelessWidget`, `StatefulWidget`, `ProxyWidget`\) participate in the build process; `RenderObjectWidget` subclasses, generally associated with `RenderObjectElements`, do not \(these simply update their render object when building\). `ComponentElement` instances only have a single child, typically that returned by their widget’s build method.
* When the element tree is first affixed to the render tree via `RenderObjectToWidgetAdapter.attachToRenderTree`, the `RenderObjectToWidgetElement` \(itself a `RootRenderObjectElement`\) assigns a `BuildOwner` for the element tree. The `BuildOwner` is responsible for tracking dirty elements \(`BuildOwner.scheduleBuildFor`\), establishing build scopes wherein elements can be rebuilt / descendent elements can be marked dirty \(`BuildOwner.buildScope`\), and unmounting inactive elements at the end of a frame \(`BuildOwner.finalizeTree`\). It also maintains a reference to the root `FocusManager` and triggers reassembly after a hot reload.
* When a `ComponentElement` is mounted \(e.g., after being inflated\), an initial build is performed immediately \(via `ComponentElement.\_firstBuild`, which calls `ComponentElement.rebuild`\).
* Later, elements can be marked dirty using `Element.markNeedsBuild`. This is invoked any time the UI might need to be updated reactively \(or explicitly, in response to `State.setState`\). This method adds the element to the dirty list and, via `BuildOwner.onBuildScheduled`, schedules a frame via `SchedulerBinding.ensureVisualUpdate`.
  * Other operations trigger a rebuild directly \(i.e., without marking the tree dirty first\). These include `ProxyElement.update`, `StatelessElement.update`, `StatefulElement.update`, and `ComponentElement.mount` \(for the initial build, only\).
  * In contrast, builds are scheduled by `State.setState`, `Element.reassemble`, `Element.didChangeDependencies`, `StatefulElement.activate`, etc.
  * Proxy elements use notifications to indicate when underlying data has changed. In the case of `InheritedElement`, each dependant’s `Element.didChangeDependencies` is invoked which, by default, marks that element as dirty.
* Once per frame, `BuildOwner.buildScope` will walk the element tree in depth-first order, only considering those nodes that have been marked dirty. By locking the tree and iterating in depth first order, any nodes that become dirty while rebuilding must necessarily be lower in the tree; this is because building is a unidirectional process -- a child cannot mark its parent as being dirty. Thus, it is not possible for build cycles to be introduced and it is not possible for elements that have been marked clean to become dirty again.
* As the build progresses, `ComponentElement.performRebuild` delegates to the `ComponentElement.build` method to produce a new child widget for each dirty element. Next, `Element.updateChild` is invoked to efficiently reuse or recreate an element for the child. Crucially, if the child’s widget hasn’t changed, the build is immediately cut off. Note that if the child widget did change and `Element.update` is needed, that child will itself be marked dirty, and the build will continue down the tree.
* Each Element maintains a map of all `InheritedElement` ancestors at its location. Thus, accessing dependencies from the build process is a constant time operation.
* If `Element.updateChild` invokes `Element.deactivateChild` because a child is removed or moved to another part of the tree, `BuildOwner.finalizeTree` will unmount the element if it isn’t reintegrated by the end of the frame.

## How do stateful widgets work?

* `StatefulWidget` is associated with `StatefulElement`, a `ComponentElement` that is almost identical to `StatelessElement`. The key difference is that the `StatefulElement` references the State of the corresponding `StatefulWidget`, and invokes lifecycle methods on that instance at key moments. For instance, when `StatefulElement.update` is invoked, the State instance is notified via `State.didUpdateWidget`.
* `StatefulElement` creates the associated State instance when it is constructed \(i.e., in `StatefulWidget.createElement`\). Then, when the `StatefulElement` is built for the first time \(via `StatefulElement.\_firstBuild`, called by `StatefulElement.mount`\), `State.initState` is invoked. Crucially, State instance and the `StatefulWidget` share the same element.
* Since State is associated with the underlying `StatefulElement`, if the widget changes, provided that `StatefulElement.updateChild` is able to reuse the same element \(because the widget’s runtime type and key both match\), State will be preserved. Otherwise, the State will be recreated.

## Why is changing tree depth expensive?

* Changing the depth necessarily causes a rebuild of the subtree, since there will never be an element match for the new child \(since it wasn’t a child previously\).
* This requires `Element.inflateWidget` to be invoked, which causes a fresh build, which causes layout, paint, etc.
* Adding a `GlobalKey` to the previous child can mitigate this issue since `Element.updateChild` is able to reuse elements that are stored in the `GlobalKey` registry \(allowing that subtree to simply be reinserted instead of rebuilt\).

