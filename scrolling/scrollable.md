# Scrollable


## What are the building blocks of scrolling?

* Scrollable provides the interaction model for scrolling without specifying how the actual viewport is managed \(a `ViewportBuilder` must be provided\). UI concerns are customized directly or via an inherited `ScrollConfiguration` that exposes an immutable `ScrollBehavior` instance. This instance is used to build platform-specific chrome \(i.e., a scrolling indicator\) and provides ambient `ScrollPhysics`, a class that describes how scrolling UI will respond to user gestures.
* `ScrollPhysics` is consulted throughout the framework to construct physics simulations for ballistic scrolling, to validate and adjust user interaction, to manage momentum across interactions, and to identify overscroll regions.
* `ScrollableState` connects the Scrollable to a `ScrollPosition` via a `ScrollController`. This controller is responsible for producing the `ScrollPosition` from a given `ScrollContext` and `ScrollPhysics`; it also provides the `initialScrollOffset`.
  * For example, `PageView` injects a page-based scrolling mechanism by having its `ScrollController` \(`PageController`\) return a custom scroll position subclass.
* The `ScrollContext` exposes build contexts for notifications and storage, a ticker provider for animations, and methods to interact with the scrollable; its analogous to `BuildContext` for an entire scrollable widget.
* `ScrollPosition` tracks scroll offset as pixels \(reporting changes via Listenable\), applies physics to interactions via `ScrollPhysics`, and through subclasses like `ScrollPositionWithSingleContext` \(which implement `ScrollActivityDelegate` and makes concrete much of the actual scrolling machinery\), starts and stops `ScrollActivity` instances to mutate the represented scroll position.
  * The actual pixel offset and mechanisms for reacting to changes in the associated viewport are introduced via the `ViewportOffset` superclass.
  * Viewport metrics are mixed in via `ScrollMetrics`, which redundantly defines pixel offset and defines a number of other useful metrics like the amount of content above and below the current viewport \(`extentBefore`, `extentAfter`\), the pixel offset corresponding to the top and bottom of the current viewport \(`minScrollExtent`, `maxScrollExtent`\) and the viewport size \(`viewportDimension`\).
  * The scroll position may need to be corrected \(via `ScrollPosition.correctPixels` \[replaces pixels outright\] / `ViewportOffset.correctBy` \[applies a delta to pixels\]\) when the viewport is resized, as triggered by shrink wrapping or relayout. Every time a viewport \(via `RenderViewport`\) is laid out, the new content extents are checked by `ViewportOffset.applyContentDimensions` to ensure the offset won’t change; if it does, layout must be repeated.
  * `ViewportOffset.applyViewportDimension` and `ViewportOffset.applyContentDimensions` are called to determine if this is the case; any extents provided represent viewport slack -- how far the viewport can be scrolled in either direction beyond what is already visible. Activities are notified via `ScrollActivity.applyNewDimensions`\(\).
    * The original pixel values corresponds to certain children being visible. If the dimensions of the viewport change, the pixel offset required to maintain that same view may change. For example, consider a viewport sized to a single letter displaying “A,” “B,” and “C” in a column. When “B” is visible, pixels will correspond to “A”’s height. Suppose the viewport expands to fit the full column. Now, pixels will be zero \(no offset is needed\). \[?\]
    * The same is true if the viewport’s content changes size. Again, consider the aforementioned “A-B-C” scenario with “B” visible. Instead of the viewport changing size, suppose “A” is resized to be zero pixels tall. To keep “B” in view, the pixel offset must be updated \(from non-zero to zero\). \[?\]
* `ScrollController` provides a convenient interface for interacting with one or more `ScrollPositions`; in effect, it calls the corresponding method in each of its positions. As a Listenable, the controller aggregates notifications from its positions.
* `ScrollNotifications` are emitted by scrollable \(by way of the active `ScrollActivity`\). As a `LayoutChangedNotification` subclass, these are emitted after build and layout have already occurred, thus only painting can be performed in response without introduce jank.
  * Listening to a scroll position directly avoids the delay, allowing layout to be performed in response to offset changes. It’s not clear why this is faster - both paths seem to trigger at the same time \[?\]

## How is the scroll position updated in general?

* The `ScrollPositionWithSingleContext` starts and manages `ScrollActivity` instances via drag, `animateTo`, `jumpTo`, and more.
* `ScrollActivity` instances update the scroll position via `ScrollActivityDelegate`; `ScrollPositionWithSingleContext` implements this interface and applies changes requested by the current activity \(`setPixels`, `applyUserOffset`\) and starts follow-on activities \(`goIdle`, `goBalastic`\).
* Any changes applied by the activity are processed by the scroll position, then passed back to the activity which generates scroll notifications \(e.g., `dispatchScrollUpdateNotification`\).
* `DragScrollActivity`, `DrivenScrollActivity`, and `BallisticScrollActivity` apply user-driven scrolling, animation-driven scrolling, and physics-driven scrolling, respectively.
* `ScrollPosition.beginActivity` starts activities and tracks all state changes. This is possible because the scroll position is always running an activity, even when idle \(`IdleScrollActivity`\). These state changes generate scroll notifications via the activity.

## How is the scroll position updated by dragging?

* The underlying Scrollable uses a gesture recognizer to detect and track dragging if `ScrollPhysics.shouldAcceptUserOffset` allows. When a drag begins, the Scrollable’s scroll position is notified via `ScrollPosition.drag`.
* `ScrollPositionWithSingleContext` implements this method to create a `ScrollDragController` which serves as an integration point for the Scrollable, which receives drag events, and the activity, which manages scroll state / notifications. The controller is returned as a Drag instance, which provides a mechanism to update state as events arrive.
* As the user drags, the drag controller forwards a derived user offset back to `ScrollActivityDelegate.applyUserOffset` \(`ScrollPositionWithSingleContext`\) which applies `ScrollPhysics.applyPhysicsToUserOffset` and, if valid, invokes `ScrollActivityDelegate.setPixels`. This actually updates the scroll offset and generates scroll notifications.
* When the drag completes, a ballistic simulation is started via `ScrollActivityDelegate.goBallistic`. This delegates to the scroll position’s `ScrollPhysics` instance to determine how to react.
* Interestingly, the `DragScrollActivity` delegates most of its work to the drag controller and is mainly responsible for forwarding scroll notifications.

## How is the scroll position updated by `animateTo`?

* The `DrivenScrollActivity` is much more straightforward. It starts an animation controller which, on every tick, updates the current pixel value via `setPixels`. When animating, if the container over-scrolls, an idle activity is started. If the animation completes successfully, a ballistic activity is started instead.

## How is scrolling behavior and state managed?

*  The `ScrollPosition` writes the current scroll offset to `PageStorage` if `ScrollPosition.keepScrollOffset` is true.

## How are the scrollable, the viewport, and any contained slivers associated?

* `ScrollView` is a base class that builds a scrollable and a viewport, deferring to its subclass to specify how its slivers are constructed. The subclass overrides `buildSlivers` to do this \(`ScrollView.build` creates the Scrollable, which uses `ScrollView.buildViewport` as its `viewportBuilder`, which uses `ScrollView.buildSlivers` to obtain the sliver children\).

