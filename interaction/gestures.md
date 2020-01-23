---
description: '`TODO`: `Expand`.'
---

# Gestures


## How are hit tests performed?

* The `RendererBinding` is `HitTestable`, which implies the presence of a `hitTest` method. The default implementation defers to `RenderView`, which itself implements `hitTest` to visit the entire render tree. Each render object is given an opportunity to add itself to a shared `HitTestResult`.
* `GestureBinding.dispatchEvent` \(via `HitTestDispatcher`\) uses the `PointerRouter` to pass the original event to all render objects that were hit \(`RenderObject` implements `HitTestTarget`, and therefore provides `handleEvent`\). If a `GestureRecognizer` is utilized, the event’s pointer is passed via `GestureRecognizer.addPointer` which registers with the `PointerRouter` to receive future events.
* Related events are sent to all original `HitTestTarget` as well as any routes registered with the `PointerRouter`. `PointerDownEvent` will close the gesture area, barring additional entries, whereas `PointerUpEvent` will sweep it, resolving competing gestures and preventing indecisive gestures from locking up input.

## How are gestures captured and propagated?

* `Window.onPointerDataPacket` captures pointer updates from the engine and generates `PointerEvents` from the raw data. `PointerEventConverter` is utilized to map physical coordinates from the engine to logical coordinates, taking into account the device’s pixel ratio \(via `PointerEventConverter.expand`\). 

## What does a gesture recognizer do?

* A pointer is added to the gesture recognizer by client code on `PointerDownEvent`. Gesture recognizers determine whether a pointer is allowed by overriding `GestureRecognizer.isPointerAllowed`. If so, the recognizer subscribes to future pointer events via the `PointerRouter` and adds the pointer to the gesture arena via `GestureArenaManager.add`.
* The recognizer will process incoming events, outcomes from the gesture arena \(`GestureArenaMember.accept`/`rejectGesture`\), spontaneous decisions about the gesture \(`GestureArenaEntry.resolve`\), and other externalities. Typically, recognizers watch the stream of `PointerEvents` via `HitTestTarget.handleEvent`, looking for terminating events like `PointerDown`, or criteria that will cause acceptance / rejection. If the gesture is accepted, the recognizer will continue to process events to characterize the gesture, invoking user-provided callbacks at key moments.
* The gesture recognizer must unsubscribe from the pointer when rejecting or done processing, removing itself from the `PointerRouter` \(`OneSequenceGestureRecognizer.stopTrackingPointer` does this\).

## What does the gesture arena do?

* The arena disambiguates multiple gestures in a way that allows single gestures to resolve immediately if there is no competition. A recognizer “wins” if it declares itself the winner or if it’s the last/sole survivor.

## Can gesture recognizers be grouped together?

* A `GestureArenaTeam` combines multiple recognizers together into a group. 
* Captained teams cause the captain recognizer to win when all unaffiliated recognizers reject or a constituent accepts.
* A non-captained team causes the first added recognizer to win when all unaffiliated recognizers reject. However, if a constituent accepts, that recognizer still takes the win.

## What auxiliary classes support gesture handling?

* There are two major categories of gesture recognizers, multi-touch recognizers \(i.e., `MultiTapGestureRecognizer`\) that simultaneously process multiple pointers \(i.e., tapping with two fingers will register twice\), and single-touch recognizer \(i.e., `OneSequenceGestureRecognizer`\) that will only consider events from a single pointer \(i.e., tapping with two fingers will register once\). 
* There is a helper “Drag” object that is used to communicate drag-related updates to other parts of the framework \(like `DragScrollActivity`\)
* There’s a `VelocityTracker` that generates fairly accurate estimates about drag velocity using curve fitting.
* There are local and global `PointerRoutes` in `PointerRouter`. Local routes are used as described above; global routes are used to react to any interaction \(i.e., to dismiss a tooltip\).

