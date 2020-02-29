# Elements

## What are elements?

* The element tree is anchored in the `WidgetsBinding` and established via `runApp` / `RenderObjectToWidgetAdapter`. Widgets are immutable representations of UI configuration data. Widgets are “inflated” into `Element` instances, which serve as their mutable counterparts. Among other things, elements model the relationship between widgets \(e.g., the widget tree\), store state / inherited relationships, and participate in the build process.
* Elements are assembled into tree \(via `Element.mount` and `Element.unmount`\). Elements may be temporarily deactivated and re-activated \(via `Element.deactivate` and `Element.activate`\). An element's parent is generally responsible for deactivating the element \(via `Element.deactivateChild`\). Deactivation temporarily removes the element \(and any associated render objects\) from the element tree; unmounting makes it permanent.
  * Elements generally transition through states via this sequence of invocations: `Element.mount` \(`initial` to `active`, `Element.activate` is implied\), `Element.deactivate` \(`active` to `inactive`, can be reactivated via `Element.activate)`, then finally `Element.unmount` \(`inactive` to `defunct`\).
* Many lifecycle events \(e.g., `initState`, `didChangeDependencies`, `didUpdateWidget`, etc.\) are triggered by changes to the element tree. In particular, all elements are associated with a `BuildOwner` that is responsible for tracking dirty elements and, during `WidgetsBinding.drawFrame`, re-building the widget / element tree. This drives lifecycle events within widget and state objects.
* Elements are initially mounted \(e.g., attached to the element tree\), which places them in the `active` state. They may then be updated multiple times. An element may be deactivated \(by the parent via `Element.deactivateChild`\); this removes any associated render objects from the render tree and adds the element to the build owner's list of inactive nodes. It also clears all inherited dependencies \(e.g., from `InheritedElement`\)
  * An element may be reactivated within the same frame \(e.g., due to tree grafting\), otherwise the element will be permanently unmounted by the build owner \(via `BuildOwner.finalizeTree` which calls `Element.unmount`\).
  * If the element is reactivated, the subtree will be restored and marked dirty, causing it to be rebuilt \(adopting any render objects which were previously dropped\). 
* `Element.updateChild` is used to update a child element when its configuration \(i.e., widget\) changes. If the new widget isn't compatible with the old one \(e.g., doesn't exist, has a different type, or has a different key\), a fresh element is inflated. Once an element is retrieved or inflated, the new configuration is applied via `Element.update`; this might alter an associated render object, notify dependents of a state change, or tweak the element itself.

## What are the element building blocks?

* Elements are broken down into `RenderObjectElements` and `ComponentElements`. `RenderObjectElements` are responsible for configuring render objects and keeping the render object tree and widget tree in sync. `ComponentElements` don't directly manage render objects but instead produce intermediate nodes via mechanisms like `Widget.build`. Both are triggered by `Element.performRebuild` which is itself triggered by `BuildOwner.buildScope`. The latter is run as part of the build process every time the engine requests a frame.

## How is the render tree managed by `RenderObjectElement`?

* Render object elements are responsible for managing an associated render object. `RenderObjectElement.update` will update this render object to match a new configuration \(i.e., widget\).
  * The render object is created \(via `RenderObjectWidget.createRenderObject`\) when its element is first mounted. The render object is retained throughout the life of the element, even when the element is deactivated \(and the render object detached\). 
    * A new render object is created if an element is inflated and mounted \(e.g., because the corresponding widget changed\); at this point, the old render object is destroyed. A slot token is used during this process so the render object can attach itself to its render object parent.
  * The render object is attached to the parent \(via `RenderObjectElement.attachRenderObject`\) when it's first mounted. If the element is detached \(which only happens as part of global key tree grafting\), it will be attached to its new parent when the graft is completed \(via `RenderObjectElement.inflateWidget`, which includes special logic for handling global keys\).
  * The render object is updated \(via `RenderObjectWidget.updateRenderObject`\) when the element is updated \(via `Element.update`\) or rebuilt \(via `Element.rebuild`\).
  * The render object is detached from its parent \(via `RenderObjectElement.detachRenderObject`\) when the element is deactivated. This is generally managed by the parent \(via `Element.deactivateChild`\) and occurs when children are explicitly removed or reparented due to tree grafting. Deactivating a child calls `Element.detachRenderObject` which recursively processes descendants until reaching the nearest render object element boundary. `RenderObjectElement` overrides this method to detach its render object, cutting off the recursive walk.
* Render objects may have children. However, there may be several intermediate nodes \(i.e., component elements\) in the element tree between that render object's `RenderObjectElement` and the elements associated with its children.
  * Slot tokens are passed down the element tree so that these `RenderObjectElement` nodes can interact with their render object's parent \(via`RenderObjectElement.insertChildRenderObject`, `RenderObjectElement.moveChildRenderObject`, `RenderObjectElement.removeChildRenderObject`\). Tokens are interpreted by the ancestor `RenderObjectElement` to uniquely identify each child.
* Elements generally use the associated widget's children as a base source of truth \(e.g., `MultiChildRenderObjectWidget.children`\). When the element is first mounted, each child is typically inflated and stored in an internal list \(e.g., `MultiChildRenderObjectElement._children`\) which is used when updating the element.
* Elements can be grafted from one part of the tree to another during a single frame. Such elements are “forgotten” by their parents \(via `RenderObjectElement.forgetChild`\) so that they are excluding from iteration and updating. The old parent removes the child when the element is eventually added to its new parent \(during inflation, since grafting is triggered by mutating the widget tree\).
* When updating, children must be updated, too. To avoid unnecessarily inflation \(and potential loss of state\), the new and old child lists are synchronized using a linear reconciliation scheme optimized for empty lists, matched lists, and lists with one mismatched region:

1. The leading elements and widgets are matched by key and updated.
2. The trailing elements and widgets are matched by key with updates queued \(update order is significant\).
3. A mismatched region is identified in the old and new lists.
4. Old elements are indexed by key.
5. Old elements without a key are updated with null \(deleted\).
6. The index is consulted for each new, mismatched widget.
7. New widgets with keys in the index update together \(re-use\).
8. New widgets without matches are updated with null \(inflated\).
9. Remaining elements in the index are updated with null \(deleted\).

## What are the render object element building blocks?

* `LeafRenderObjectElement`, `SingleChildRenderObjectElement`, and `MultiChildRenderObjectElement` provide support for common use cases and correspond to the similarly named widget helpers. The multi-child and single-child variants integrate with `ContainerRenderObjectMixin` and `RenderObjectWithChildMixin`, respectively.
* These use the previous child \(or null\) as the slot identifier; this integrates nicely with `ContainerRenderObjectMixin` which supports an “after” argument when managing children.

## How are components managed by `ComponentElement`?

* Building is an alternative to storing a static list of children in a widget \(even though this can be somewhat dynamic if the list is mutated in response to events\).
* Instead, a method or function is invoked to dynamically produce a list of children. Flutter's reactive programming paradigm is facilitated by re-invoking such procedures any time associated state changes \(e.g., in response to events\).
* This process is managed by marking a subtree as being "dirty" \(i.e., needing to be rebuilt\) whenever state changes. This causes each affected build method to be systematically reevaluated when the engine requests the next frame. Elements are updated using these new widgets, mutating the element tree and therefore the render tree \(and UI\).
* `ComponentElement.build` provides a hook for producing intermediate nodes in the element tree. `StatelessElement.build` invokes the widget’s build method, whereas `StatefulElement.build` invokes the state’s build method. Mounting and updating cause rebuild to be invoked. For `StatefulElement`, a rebuild may be scheduled spontaneously via `State.setState`. In both cases, lifecycle methods are invoked in response to changes to the element tree \(for example, `StatefulElement.update` will invoke `State.didUpdateWidget`\).
* If a component element rebuilds, the old element and new widget will still be paired, if possible. This allows `Element.update` to be used instead of `Element.inflateWidget`. Consequently, descendant render objects may be updated instead of recreated. Provided that the render object’s properties weren’t changed, this will likely circumvent the rest of the rendering process.

## How do element dependencies work \(inheritance\)?

* Elements are automatically constructed with a hashmap of ancestor inherited widgets; thus, if a dependency is eventually added \(via `Element.inheritFromWidgetOfExactType`\), it is not necessary to walk up the tree. When elements express a dependency, that dependency is added to the element’s map via `Element.inheritFromElement` and communicated to the inherited ancestor via `InheritedElement.updateDependency`.
* Locating ancestor render objects / widgets / states that do not utilize `InheritedElement` requires a linear walk up the tree.
* `InheritedElement` is a subclass of `ProxyElement`, which introduces a notification mechanism so that descendents can be notified after `ProxyElement.update` \(via `ProxyElement.updated` and `ProxyElement.notifyClients`\). This is useful for notifying dependant elements when dependencies change so that they can be marked dirty.

