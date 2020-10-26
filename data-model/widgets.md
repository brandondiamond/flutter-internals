# Widgets

## What are the widget building blocks?

* Widgets provide an immutable description of the user interface. Though widgets themselves are immutable, they may be freely replaced, removed, or rearranged \(note that updating a widget's child typically requires the parent widget to be replaced, too\). Creating and destroying widgets is efficient since widgets are lightweight, immutable instances that are, ideally, compile-time constants.
* The immutable widget tree is used to create and configure \(i.e., inflate\) a mutable element tree which manages a separate render tree; this final tree is responsible for layout, painting, gestures, and compositing. The element tree is efficiently synchronized with widget changes, reusing and mutating elements where possible \(that is, though a widget may be replaced with a different instance, provided the two instances have the same runtime type and key, the original element will be updated and not recreated\). Modifying the element tree typically updates the render tree which, in turns, changes what appears on the screen.
* The main widget types are `RenderObjectWidget`, `StatefulWidget`, and `StatelessWidget`. Widgets that export data to one or more descendant widgets \(via notifications or another mechanism\) utilize `ProxyWidget` or one of its subclasses \(e.g., `InheritedWidget` or `ParentDataWidget`\). 
* In general, widgets either directly or indirectly configure render objects by modifying the element tree. Most widgets created by application developers \(via `StatefulWidget` and `StatelessWidget`\) delegate to a constellation of descendant widgets, typically via a build method \(e.g., `StatelessWidget.build`\). Others \(e.g., `RenderObjectWidget`\) manage a render object directly \(creating it and updating it via `RenderObjectWidget.createRenderObject` and `RenderObjectWidget.updateRenderObject`, respectively\).
* Certain widgets wrap an explicit child widget via`ProxyWidget`, introducing heritable state \(e.g.,`InheritedWidget`, `InheritedModel`\) or configuring auxiliary data \(e.g.,`ParentDataWidget`\).
  * `ProxyWidget` notifies clients \(via `ProxyElement.notifyClients`\) in response to widget changes \(via `ProxyElement.updated`, called by `ProxyElement.update`\).
  * `ParentDataWidget` updates the nearest descendant render objects' parent data \(via `ParentDataElement._applyParentData`, which calls `RenderObjectElement._updateParentData`\); this process is triggered any time the corresponding widget is updated.
* There are also bespoke widget subclasses that support less common types of configuration. For instance,`PreferredSizeWidget` extends `Widget` to capture a preferred size allowing subclasses \(e.g., `AppBar`, `TabBar`, `PreferredSize` \) to express sizing information to their containers \(e.g., `Scaffold`\).
* `LeafRenderObjectWidget`, `SingleChildRenderObjectWidget`, and `MultiChildRenderObjectWidget` provide storage for render object widgets with zero or more children without constraining how the underlying render object is created or updated. These widgets correspond to`LeafRenderObjectElement`, `SingleChildRenderObjectElement`, and `MultiChildRenderObjectElement`, respectively, which manage the underlying child model in the element and render trees.
* Anonymous widgets can be created using `Builder` and `StatefulBuilder`.

## stateless widgets 如何工作？

* `StatelessWidget` is a trivial subclass of `Widget` that defines a `StatelessWidget.build` method and configures a `StatelessElement`.
* `StatelessElement` is a `ComponentElement` subclass that invokes `StatelessWidget.build` in response to `StatelessElement.build` \(e.g., delegates building to its widget\).

## How do stateful widgets work?

* `StatefulWidget` is associated with `StatefulElement`, a `ComponentElement` that is almost identical to `StatelessElement`. The key difference is that the `StatefulElement` retains a reference to the `State` of the corresponding `StatefulWidget`, invoking methods on that instance rather than the widget itself. For instance, when `StatefulElement.update` is invoked, the `State` instance is notified via `State.didUpdateWidget`.
* `StatefulElement` creates the associated `State` instance when it is constructed \(i.e., in `StatefulWidget.createElement`\). Then, when the `StatefulElement` is built for the first time \(via `StatefulElement._firstBuild`, called by `StatefulElement.mount`\), `State.initState` is invoked. Crucially, `State` instance and the `StatefulWidget` reference the same element.
* Since `State` is associated with the underlying `StatefulElement`, if the widget changes, provided that `StatefulElement.updateChild` is able to reuse the same element \(because the widget’s runtime type and key both match\), `State` will be preserved. Otherwise, the `State` will be recreated.

## Why is changing tree depth expensive?

* Flutter doesn't have the ability to compare trees. That is, only an element's immediate children are considered when matching widgets and elements \(via `RenderObjectElement.updateChildren`\).
* When increasing the tree depth \(i.e., inserting an intermediate node\), the existing parent will be configured with a child corresponding to the intermediate widget. In most cases, this widget will not correspond to a previous child \(i.e., `Widget.canUpdate` will return false\). Thus, the new element will be freshly inflated. Since the intermediate node is the new owner of its parent's children, each of those children will also be inflated \(the intermediate node doesn't have access to the existing elements\). This will proceed down the entire subtree.
* When decreasing the tree depth, the parent will once again be assigned new children which likely won't sync with old children. Thus, the new children will need to be inflated, cascading down the entire subtree.
* Adding a `GlobalKey` to the previous child can mitigate this issue since `Element.updateChild` is able to reuse elements that are stored in the `GlobalKey` registry \(allowing that subtree to simply be reinserted instead of rebuilt\).

## How do notifications work?

* Notification support is not built directly into the widget abstraction, but layered on top of it.
* `Notification` is an abstract class that searches up the element tree, visiting each widget subclass of `NotificationListener` \(`Notification.dispatch` calls `Notification.visitAncestor`, which performs this walk\).
  * The notification invokes `NotificationListener._dispatch` on each suitable widget, comparing the notification's static type with the callback's type parameter. If there's a match \(i.e., the notification is a subtype of the callback's type parameter\), the listener is invoked.
  * If the listener returns true, the walk terminates. Otherwise, the notification continues to bubble up the tree.

