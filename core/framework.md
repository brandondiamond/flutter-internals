# Framework

## How is the app bootstrapped?


## How is the app bootstrapped?

* `runApp` kicks off binding initialization by invoking the `WidgetsFlutterBinding`/`RenderingFlutterBinding.ensureInitialized` static method. This calls each binding’s `initInstances` method, allowing each to initialize in turn.
  * This flow is built using mixin chaining: each of the concrete bindings \(`e.g`., `WidgetsFlutterBinding`\) extends `BaseBinding`, the superclass constraint shared by all binding mixins \(`e.g`., `GestureBinding`\). Consequently, common methods \(like `BaseBinding.initInstances`\) can be chained together via super invocations. These calls are linearized from left-to-right, starting with the superclass and proceeding sequentially through the mixins; this strict order allows later bindings to depend on earlier ones \[?\].
* `RendererBinding.initInstances` creates the `RenderView`, passing an initial `ViewConfiguration` \(describing the size and density of the render surface\). It then prepares the first frame \(via `RenderView.prepareInitialFrame`\); this schedules the initial layout and initial paint \(via `RenderView.scheduleInitialLayout` and `RenderView.scheduleInitialPaint`; the latter creates the root layer, a `TransformLayer`\). This marks the `RenderView` as dirty for layout and painting but does not actually schedule a frame.
  * This is important since users may wish to begin interacting with the framework \(by initializing bindings via `BaseBinding.ensureInitialized`\) before starting up the app \(via `runApp`\). For instance, a plugin may need to block on a backend service before it can be used.
* Finally, the `RendererBinding` installs a persistent frame callback to actually draw the frame \(`WidgetsBinding` overrides the method invoked by this callback to add the build phase\). Note that nothing will invoke this callback until the `Window.onDrawFrame` handler is installed. This will only happen once a frame has actually been scheduled.
* Returning to `runApp`, `WidgetsBinding.scheduleAttachRootWidget` asynchronously creates a `RenderObjectToWidgetAdapter`, a `RenderObjectWidget` that inserts its child \(`i.e`., the app’s root widget\) into the provided container \(`i.e`., the `RenderView`\). 
  * This asynchronicity is necessary to avoid scheduling two builds back-to-back; while this isn’t strictly invalid, it is inefficient and may trigger asserts in the framework.
  * If the initial build weren’t asynchronous, it would be possible for intervening events to re-dirty the tree before the warm up frame is scheduled. This would result in a second build \(without an intervening layout pass, etc.\) when rendering the warm-up frame. By ensuring that the initial build is scheduled asynchronously, there will be no render tree to dirty until the platform is initialized.
  * For example, the engine may report user settings changes during initialization \(via the `\_updateUserSettingsData` hook\). This invokes callbacks on the window \(`e.g`., `Window.onTextScaleFactorChanged`\), which are forwarded to all `WidgetsBindingObservers` \(`e.g`., via `RendererBinding.handleTextScaleFactorChanged`\). As an observer, `WidgetsApp` reacts to the settings data by requesting a rebuild.
* It then invokes `RenderObjectToWidgetAdapter.attachToRenderTree` to bootstrap and mount an element to serve as the root of the element hierarchy \(`RenderObjectToWidgetElement`\). If the element already exists, which will only happen if `runApp` is called again, its associated widget is updated \(`RenderObjectToWidgetElement.\_newWidget`\) and marked as needing to be built.
  * `RenderObjectToWidgetElement.updateChild` is invoked when this element is mounted or rebuilt, inflating or updating the child widget \(`i.e`., the app’s root widget\) accordingly. Once a descendent `RenderObjectWidget` is inflated, the corresponding render object \(which must be a `RenderBox`\) will be inserted into the `RenderView` \(via `RenderObjectToWidgetElement.insertChildRenderObject`\). The resulting render tree is managed in the usual way going forward.
  * A reference to this element is stored in `WidgetsBinding.renderViewElement`, serving as the root of the element tree. As a `RootRenderObjectElement`, this element establishes the `BuildOwner` for its descendants.
* Finally, after scheduling the first frame \(via  `SchedulerBinding.instance.ensureVisualUpdate`, which will lazily install the frame callbacks\), `runApp` invokes `SchedulerBinding.scheduleWarmUpFrame`, manually pumping the rendering pipeline. This gives the initial frame extra time to render as it’s likely the most expensive.
* `SchedulerBinding.ensureFrameCallbacksRegistered` lazily installs frame callbacks as part of `SchedulerBinding.scheduleFrame`. Frames are typically scheduled in response to `PipelineOwner.requestVisualUpdate` \(due to UI needing painting, layout, or a rebuild\). Once configured, these callbacks \(`Window.onBeginFrame`, `Window.onDrawFrame`\) are invoked once per frame by the engine, running transient and persistent processes, respectively. The latter is generally responsible for, `e.g`., ticking animations whereas the former runs the actual rendering pipeline. 

## How is a frame rendered?

* Once a frame is scheduled and callbacks are registered \(via `SchedulerBinding.ensureFrameCallbacksRegistered`\), the engine begins requesting frames automatically. The frame callbacks invoke handlers in response to these requests. In particular, `SchedulerBinding.drawFrame` processes persistent frame callbacks which are used to implement Flutter’s rendering pipeline. `WidgetsBinding.drawFrame` overrides `RendererBinding.drawFrame` to add the build process to this pipeline.
* The rendering pipeline builds widgets, performs layout, updates compositing bits, paints layers, and finally composites everything into a scene which it uploads to the engine \(via `RenderView.compositeFrame`\). Semantics are also updated by this process.
* `RenderView.compositeFrame` retains a reference to the root layer which it recursively composites using `Layer.buildScene`. This iterates through all layers that “`needsAddToScene`,” which if true, composites a new frame. If false, previous invocations of `addToScene` will have stored an `EnglineLayer` in `Layer.engineLayer`, saving work \(“retained rendering”\); this is added using `SceneBuilder.addRetained`. Once a Scene is constructed, it is uploaded to the engine via `Window.render`.

## How does the framework interact with the engine?

* The framework primarily interacts via the Window class, a dart interface with hooks into and out of the engine.
* The majority of the framework’s flows are driven by frame callbacks invoked by the engine. Other flows are driven by gesture handling, platform messaging, and device messaging.
* Each binding serves as the singleton root of a subsystem within the framework; in several cases, bindings are layered to add functionality to more fundamental bindings \(`i.e`., `WidgetsBinding` adds support for building to `RendererBinding`\). All direct framework/engine interaction is managed via the bindings, with the sole exception of the `RenderView`.

## What bindings are implemented?

* `GestureBinding` facilitates gesture handling across the framework, maintaining the gesture arena and pointer routing table.
  * Handles `Window.onPointerDataPacket`.
* `ServicesBinding` facilitates message passing between the framework and platform.
  * Handles `Window.onPlatformMessage`.
* `SchedulerBinding` manages a variety of callbacks \(transient, persistent, post-frame, and non-rendering tasks\), tracking lifecycle states and scheduler phases. It is also responsible for explicitly scheduling frames when visual updates are needed.
  * Handles `Window.onDrawFrame`, `Window.onBeginFrame`.
  * Invokes `Window.scheduleFrame`.
* `PaintingBinding` owns the image cache which efficiently manages memory allocated to graphical assets used by the application. It also performs shader warm up to avoid jank during drawing \(via `ShaderWarmUp.execute` in `PaintingBinding.initInstances`\). This ensures that the corresponding shaders are compiled at a predictable time.
* `SemanticsBinding` which is intended to manage the semantics and accessibility subsystems \(at the moment, this binding mainly tracks accessibility changes emitted by the engine via `Window.onAccessibilityFeaturesChanged`\).
* `RendererBinding` implements the rendering pipeline. Additionally, it retains the root of the render tree \(`i.e`., the `RenderView`\) as well as the `PipelineOwner`, an instance that tracks visual updates due to layout, painting, compositing, etc. The `RendererBinding` also responds to events that may affect the application’s rendering \(including semantic state until these handlers are moved to the `SemanticsBinding`\).
  * Handles `Window.onSemanticsAction`, `Window.onTextScaleFactorChanged`, `Window.onMetricsChanged`, `Window.onSemanticsEnabledChanged`.
  * Invokes `Window.render`.
* `WidgetsBinding` augments the renderer binding with support for widget building \(`i.e`., configuring the render tree based on immutable UI descriptions\). It also retains the `BuildOwner`, an instance that facilitates rebuilding the render tree when configuration changes \(`e.g`., a new widget is substituted\). The `WidgetsBinding` also responds to events that might require rebuilding related to accessibility and locale changes \(though these may be moved to the `SemanticsBinding` in the future\).
  * Handles `Window.onAccessibilityFeaturesChanged`, `Window.onLocaleChanged`.
* `TestWidgetsFlutterBinding` supports the widget testing framework.

## How do global keys work?

* `Element.inflateWidget` checks for a global key and uses it to find the original widget

