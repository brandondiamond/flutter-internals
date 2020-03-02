# Elements

## What are elements?

* The element tree is anchored in the `WidgetsBinding` and established via `runApp` / `RenderObjectToWidgetAdapter`.
* `Widget` instances are immutable representations of UI configuration data that are “inflated” into `Element` instances. Elements therefore serve as widgets' mutable counterparts and are responsible for modeling the relationship between widgets \(e.g., the widget tree\), storing state and inherited relationships, and participating in the build process, etc.
* All elements are associated with a `BuildOwner` singleton. This instance is responsible for tracking dirty elements and, during `WidgetsBinding.drawFrame`, re-building the element tree as needed. This process triggers several lifecycle events \(e.g., `initState`, `didChangeDependencies`, `didUpdateWidget`\).
* Elements are assembled into a tree \(via `Element.mount` and `Element.unmount`\). Whereas these operations are permanent, elements may also be temporarily removed and restored \(via `Element.deactivate` and `Element.activate`, respectively\).
  * Elements transition through several lifecycle states \(`_ElementLifecycle`\) in response to the following methods: `Element.mount` \(`initial` to `active`, `Element.activate` is implied\), `Element.deactivate` \(`active` to `inactive`, can be reactivated via `Element.activate)`, then finally `Element.unmount` \(`inactive` to `defunct`\).
  * Note that deactivating or unmounting an element is a recursive process, generally facilitated by the build owner's inactive elements list \(`BuildOwner._inactiveElements`\). All descendant elements are affected \(via`_InactiveElements._deactivateRecursively` and`_InactiveElements._unmount`\).
* Elements are attached \(i.e., mounted\) to the element tree when they're first created. They may then be updated \(via `Element.update`\) multiple times as they become dirty \(e.g., due to widget changes or notifications\). An element may also be deactivated; this removes any associated render objects from the render tree and adds the element to the build owner's list of inactive nodes. This list \(`_InactiveElements`\) automatically deactivates all nodes in the affected subtree and clears all dependencies \(e.g., from `InheritedElement`\).
  * Parents are generally responsible for deactivating their children \(via `Element.deactivateChild`\). Deactivation temporarily removes the element \(and any associated render objects\) from the element tree; unmounting makes this change permanent.
  * An element may be reactivated within the same frame \(e.g., due to tree grafting\), otherwise the element will be permanently unmounted by the build owner \(via `BuildOwner.finalizeTree` which calls `Element.unmount`\).
  * If the element is reactivated, the subtree will be restored and marked dirty, causing it to be rebuilt \(re-adopting any render objects which were previously dropped\). 
* `Element.updateChild` is used to update a child element when its configuration \(i.e., widget\) changes. If the new widget isn't compatible with the old one \(e.g., doesn't exist, has a different type, or has a different key\), a fresh element is inflated. Once an element is retrieved or inflated, the new configuration is applied via `Element.update`; this might alter an associated render object, notify dependents of a state change, or mutate the element itself.

## What are the element building blocks?

* Elements are broken down into `RenderObjectElement` and `ComponentElement`. `RenderObjectElements` are responsible for configuring render objects and keeping the render object tree and widget tree in sync. `ComponentElements` don't directly manage render objects but instead produce intermediate nodes via mechanisms like `Widget.build`. Both processes are driven by `Element.performRebuild` which is itself triggered by `BuildOwner.buildScope`. The latter is run as part of the build process every time the engine requests a frame.

## How is the render tree managed by `RenderObjectElement`?

* Render object elements are responsible for managing an associated render object. `RenderObjectElement.update` applies updates to this render object to match a new configuration \(i.e., widget\).
  * The render object is created \(via `RenderObjectWidget.createRenderObject`\) when its element is first mounted. The render object is retained throughout the life of the element, even when the element is deactivated \(and the render object is detached\). 
    * A new render object is created if an element is inflated and mounted \(e.g., because a new widget couldn't update the old one\); at this point, the old render object is destroyed. A slot token is used during this process so the render object can attach and detach itself from the render tree \(which can vary from the element tree\).
  * The render object is attached to the render tree when its element is first mounted \(via `RenderObjectElement.attachRenderObject`\). If the element is later deactivated \(due to tree grafting\), it will be re-attached when the graft is completed \(via `RenderObjectElement.inflateWidget`, which includes special logic for handling grafting by global key\).
  * The render object is updated \(via `RenderObjectWidget.updateRenderObject`\) when its element is updated \(via `Element.update`\) or rebuilt \(via `Element.rebuild`\).
  * The render object is detached from its parent \(via `RenderObjectElement.detachRenderObject`\) when the element is deactivated. This is generally managed by the parent \(via `Element.deactivateChild`\) and occurs when children are explicitly removed or reparented due to tree grafting. Deactivating a child calls `Element.detachRenderObject` which recursively processes descendants until reaching the nearest render object element boundary. `RenderObjectElement` overrides this method to detach its render object, cutting off the recursive walk.
* Render objects may have children. However, there may be several intermediate nodes \(i.e., component elements\) between its `RenderObjectElement` and the elements associated with its children. That is, the element tree typically has many more nodes than the render tree.
  * Slot tokens are passed down the element tree so that these `RenderObjectElement` nodes can interact with their render object's parent \(via`RenderObjectElement.insertChildRenderObject`, `RenderObjectElement.moveChildRenderObject`, `RenderObjectElement.removeChildRenderObject`\). Tokens are interpreted in an implementation-specific manner by the ancestor `RenderObjectElement` to distinguish render object children.
* Elements generally use their widget's children as the source of truth \(e.g., `MultiChildRenderObjectWidget.children`\). When the element is first mounted, each child is inflated and stored in an internal list \(e.g., `MultiChildRenderObjectElement._children`\); this list is later used when updating the element.
* Elements can be grafted from one part of the tree to another within a single frame. Such elements are “forgotten” by their parents \(via `RenderObjectElement.forgetChild`\) so that they are excluding from iteration and updating. The old parent removes the child when the element is added to its new parent \(this happens during inflation, since grafting requires that the widget tree be updated, too\).
* Elements are responsible for updating any children. To avoid unnecessarily inflation \(and potential loss of state\), the new and old child lists are synchronized using a linear reconciliation scheme optimized for empty lists, matched lists, and lists with one mismatched region:

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

* `LeafRenderObjectElement`, `SingleChildRenderObjectElement`, and `MultiChildRenderObjectElement` provide support for common use cases and correspond to the similarly named widget helpers \(`LeafRenderObjectWidget`, `SingleChildRenderObjectWidget`, `MultiChildRenderObjectWidget`\)
  * The multi-child and single-child variants integrate with `ContainerRenderObjectMixin` and `RenderObjectWithChildMixin` in the render tree.
* These use the previous child \(or null\) as the slot identifier; this is convenient since `ContainerRenderObjectMixin` manages children using a linked list.

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

