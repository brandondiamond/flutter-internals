# Elements


## What are elements?

* The element tree is anchored in the `WidgetsBinding` and established via `runApp` / `RenderObjectToWidgetAdapter`. Widgets are immutable representations of UI configuration data. Widgets are “inflated” into `Element` instances, which serve as their mutable counterparts. Among other things, elements model the relationship between widgets \(e.g., the widget tree\), store state / inherited relationships, and participate in the build process.
* Many lifecycle events are triggered by changes to the element tree. In particular, all elements are associated with a `BuildOwner` that is responsible for tracking dirty elements and, during `WidgetsBinding.drawFrame`, re-building the widget / element tree. This drives lifecycle events within widget and state objects.
* Elements are initially mounted. They may then be updated multiple times. An element may be deactivated \(by the parent via `Element.deactivateChild`\); it can be activated within the same frame, else it will be unmounted.
* `Element.updateChild` is used to alter the configuration \(widget\) of a given child, potentially inflating a new element if none exists, the new and old widgets do not have the same type, or the widgets have different keys. Updating an element may update the element’s children.

## What are the element building blocks?

* Elements are broken down into `RenderObjectElements` and `ComponentElements`. `RenderObjectElements` are responsible for configuring render objects and keeping the render object tree and widget tree in sync. `ComponentElements` do not directly manage render objects but instead produce intermediate nodes via mechanisms like `Widget.build`. Both are triggered by `Element.performRebuild` which is itself triggered by `BuildOwner.buildScope`.

## How is the render tree managed by `RenderObjectElement`?

* Render object elements are responsible for managing an associated render object. `RenderObjectElement.update` will cause the associated render object to be updated to match a new configuration \(widget\). This render object may have children; however, there may be several intermediate elements between its render object element and any descendent render object elements \(due to intervening component elements\). Slot tokens are passed down the element tree so that the render object elements corresponding to these children can integrate with the parent render object. This is managed via `RenderObjectElement.insertChildRenderObject`, `RenderObjectElement.moveChildRenderObject`, and `RenderObjectElement.removeChildRenderObject`.
* Elements can be moved during a frame. Such elements are “forgotten” so that they are excluding from iteration / updates, with the actual mutation taking place when the element is added to its new parent.
* When updating, children must be updated, too. To avoid unnecessarily inflation \(and potential loss of state\), the new and old child lists are synchronized using a linear reconciliation scheme optimized for empty lists, matched lists, and lists with one mismatched region:
    1. The leading elements and widgets are matched by key and updated
    2. The trailing elements and widgets are matched by key with updates queued \(update order is significant\)
    3. A mismatched region is identified in the old and new lists
    4. Old elements are indexed by key
    5. Old elements without a key are updated with null \(deleted\)
    6. The index is consulted for each new, mismatched widget
    7. New widgets with keys in the index update together \(re-use\)
    8. New widgets without matches are updated with null \(inflated\)
    9. Remaining elements in the index are updated with null \(deleted\)

## What are the render object element building blocks?

* `LeafRenderObjectElement`, `SingleChildRenderObjectElement`, and `MultiChildRenderObjectElement` provide support for common use cases and correspond to the similarly named widget helpers. The multi-child and single-child variants integrate with `ContainerRenderObjectMixin` and `RenderObjectWithChildMixin`, respectively.
* These use the previous child \(or null\) as the slot identifier; this integrates nicely with `ContainerRenderObjectMixin` which supports an “after” argument when managing children.

## How are components managed by `ComponentElement`?

* `ComponentElement.build` provides a hook for producing intermediate nodes in the element tree. `StatelessElement.build` invokes the widget’s build method, whereas `StatefulElement.build` invokes the state’s build method. Mounting and updating cause rebuild to be invoked. For `StatefulElement`, a rebuild may be scheduled spontaneously via `State.setState`. In both cases, lifecycle methods are invoked in response to changes to the element tree \(for example, `StatefulElement.update` will invoke `State.didUpdateWidget`\).
* If a component element rebuilds, the old element and new widget will still be paired, if possible. This allows `Element.update` to be used instead of `Element.inflateWidget`. Consequently, descendant render objects may be updated instead of recreated. Provided that the render object’s properties weren’t changed, this will likely circumvent the rest of the rendering process.

## How do element dependencies work \(inheritance\)?

* Elements are automatically constructed with a hashmap of ancestor inherited widgets; thus, if a dependency is eventually added \(via `Element.inheritFromWidgetOfExactType`\), it is not necessary to walk up the tree. When elements express a dependency, that dependency is added to the element’s map via `Element.inheritFromElement` and communicated to the inherited ancestor via `InheritedElement.updateDependency`.
* Locating ancestor render objects / widgets / states that do not utilize `InheritedElement` requires a linear walk up the tree.
* `InheritedElement` is a subclass of `ProxyElement`, which introduces a notification mechanism so that descendents can be notified after `ProxyElement.update` \(via `ProxyElement.updated` and `ProxyElement.notifyClients`\). This is useful for notifying dependant elements when dependencies change so that they can be marked dirty.

