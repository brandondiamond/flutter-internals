# Widgets

## What are the widget building blocks?

* The main entry points are `RenderObjectWidget`, `StatefulWidget`, and `StatelessWidget`. Widgets that export data to one or more descendant widgets \(via notifications or another mechanism\) utilize `ProxyWidget` or its subclasses, `InheritedWidget` and `ParentDataWidget`. 
* In general, widgets either directly or indirectly configure render objects; these indirect “component” widgets \(e.g., stateful and stateless widgets\) participate in a building process, whereas render object widgets manage an associated render object directly \(creating it and updating it via `RenderObjectWidget.createRenderObject` and `RenderObjectWidget.updateRenderObject`, respectively\).
* Certain widgets wrap \(i.e., reproject\) a child widget \(`ProxyWidget`\), introducing heritable state \(`InheritedWidget`, `InheritedModel`\) or setting parent data used by an enclosing widget \(`ParentDataWidget`\).
  * `ProxyWidget` notifies clients \(via `ProxyElement.notifyClients`\) in response to widget changes \(via `ProxyElement.updated`, called by `ProxyElement.update`\).
  * `ParentDataWidget` utilizes this flow to update the first descendant render object’s parent data \(via `ParentDataElement._applyParentData`, which calls `RenderObjectElement._updateParentData`\); this will be triggered any time the widget is updated.
* `PreferredSizeWidget` is not widely used but an interesting example of adapting widgets to a specific need \(e.g., an `AppBar` expressing a size preference\).
* Numerous helper subclasses exist, most notably including `SingleChildRenderObjectWidget`, `MultiChildRenderObjectWidget`, `LeafRenderObjectWidget`. These provide storage for a child / children without implementing the underlying child model.
* Anonymous widgets can be created using `Builder` and `StatefulBuilder`.

## How do stateful widgets work?

* `StatefulWidget` is associated with `StatefulElement`, a `ComponentElement` that is almost identical to `StatelessElement`. The key difference is that the `StatefulElement` references the `State` of the corresponding `StatefulWidget`, and invokes lifecycle methods on that instance at key moments. For instance, when `StatefulElement.update` is invoked, the `State` instance is notified via `State.didUpdateWidget`.
* `StatefulElement` creates the associated `State` instance when it is constructed \(i.e., in `StatefulWidget.createElement`\). Then, when the `StatefulElement` is built for the first time \(via `StatefulElement._firstBuild`, called by `StatefulElement.mount`\), `State.initState` is invoked. Crucially, `State` instance and the `StatefulWidget` share the same element.
* Since `State` is associated with the underlying `StatefulElement`, if the widget changes, provided that `StatefulElement.updateChild` is able to reuse the same element \(because the widget’s runtime type and key both match\), `State` will be preserved. Otherwise, the `State` will be recreated.

## Why is changing tree depth expensive?

* Changing the depth necessarily causes a rebuild of the subtree, since there will never be an element match for the new child \(since it wasn’t a child previously\).
* This requires `Element.inflateWidget` to be invoked, which causes a fresh build, which causes layout, paint, etc.
* Adding a `GlobalKey` to the previous child can mitigate this issue since `Element.updateChild` is able to reuse elements that are stored in the `GlobalKey` registry \(allowing that subtree to simply be reinserted instead of rebuilt\).

