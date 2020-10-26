# Framework - 框架

## Flutter app 如何启动界面?

* `runApp` 通过调用静态方法 `WidgetsFlutterBinding`/`RenderingFlutterBinding.ensureInitialized` 初始化绑定. 这将调用每个绑定的`initInstances`方法，从而允许每个对象依次初始化。\(就是mixin - with实现的多继承,每个bindng都触发了init\)
  * 该流程是使用_**mixin**_链接构建的：每个具体的绑定（例如`WidgetsFlutterBinding`）都扩展了`BaseBinding`，所有 "Binding" _**mixin**_（例如`GestureBinding`）共享的超类约束。因此，可以通过超级调用将常用方法（如`BaseBinding.initInstances`）链接在一起。这些调用是从左到右线性化的，从超类开始，然后依次进行混合。这种严格的顺序允许以后的绑定依赖更早的绑定。
    * 注意:mixin跟with的写法,dart有一种类似多继承的方式,每个mixin实现不同的自身逻辑,依附于同一个父类
* `RendererBinding.initInstances` creates the `RenderView`, passing an initial `ViewConfiguration` \(describing the size and density of the render surface\). It then prepares the first frame \(via `RenderView.prepareInitialFrame`\); this schedules the initial layout and initial paint \(via `RenderView.scheduleInitialLayout` and `RenderView.scheduleInitialPaint`; the latter creates the root layer, a `TransformLayer`\). This marks the `RenderView` as dirty for layout and painting but does not actually schedule a frame.
  * This is important since users may wish to begin interacting with the framework \(by initializing bindings via `BaseBinding.ensureInitialized`\) before starting up the app \(via `runApp`\). For instance, a plugin may need to block on a backend service before it can be used.
* Finally, the `RendererBinding` installs a persistent frame callback to actually draw the frame \(`WidgetsBinding` overrides the method invoked by this callback to add the build phase\). Note that nothing will invoke this callback until the `Window.onDrawFrame` handler is installed. This will only happen once a frame has actually been scheduled.
* Returning to `runApp`, `WidgetsBinding.scheduleAttachRootWidget` asynchronously creates a `RenderObjectToWidgetAdapter`, a `RenderObjectWidget` that inserts its child \(i.e., the app’s root widget\) into the provided container \(i.e., the `RenderView`\). 
  * This asynchronicity is necessary to avoid scheduling two builds back-to-back; while this isn’t strictly invalid, it is inefficient and may trigger asserts in the framework.
  * If the initial build weren’t asynchronous, it would be possible for intervening events to re-dirty the tree before the warm up frame is scheduled. This would result in a second build \(without an intervening layout pass, etc.\) when rendering the warm-up frame. By ensuring that the initial build is scheduled asynchronously, there will be no render tree to dirty until the platform is initialized.
  * For example, the engine may report user settings changes during initialization \(via the `_updateUserSettingsData` hook\). This invokes callbacks on the window \(e.g., `Window.onTextScaleFactorChanged`\), which are forwarded to all `WidgetsBindingObservers` \(e.g., via `RendererBinding.handleTextScaleFactorChanged`\). As an observer, `WidgetsApp` reacts to the settings data by requesting a rebuild.
* It then invokes `RenderObjectToWidgetAdapter.attachToRenderTree` to bootstrap and mount an element to serve as the root of the element hierarchy \(`RenderObjectToWidgetElement`, i.e., the element corresponding to the adapter\). If the element already exists, which will only happen if `runApp` is called again, its associated widget is updated \(`RenderObjectToWidgetElement._newWidget`\) and marked as needing to be built.
  * `RenderObjectToWidgetElement.updateChild` is invoked when this element is mounted or rebuilt, inflating or updating the child widget \(i.e., the app’s root widget\) accordingly. Once a descendant `RenderObjectWidget` is inflated, the corresponding render object \(which must be a `RenderBox`\) will be inserted into the `RenderView` \(via `RenderObjectToWidgetElement.insertChildRenderObject`\). The resulting render tree is managed in the usual way going forward.
  * A reference to this element is stored in `WidgetsBinding.renderViewElement`, serving as the root of the element tree. As a `RootRenderObjectElement`, this element establishes the `BuildOwner` for its descendants.
* Finally, after scheduling the first frame \(via  `SchedulerBinding.instance.ensureVisualUpdate`, which will lazily install the frame callbacks\), `runApp` invokes `SchedulerBinding.scheduleWarmUpFrame`, manually pumping the rendering pipeline. This gives the initial frame extra time to render as it’s likely the most expensive.
* `SchedulerBinding.ensureFrameCallbacksRegistered` lazily installs frame callbacks as part of `SchedulerBinding.scheduleFrame`. Frames are typically scheduled in response to `PipelineOwner.requestVisualUpdate` \(due to UI needing painting, layout, or a rebuild\). Once configured, these callbacks \(`Window.onBeginFrame`, `Window.onDrawFrame`\) are invoked once per frame by the engine, running transient and persistent processes, respectively. The latter is generally responsible for ticking animations whereas the former runs the actual building and rendering pipeline. 

## 每一帧如何渲染的?

* 当第一帧已经就绪,并且回调已经被注册 \(via `SchedulerBinding.ensureFrameCallbacksRegistered`\), flutter引擎便开始自动处理每一帧. 帧回调响应这些请求调用处理程序。特别是Flutter实现在渲染管道\(rendering pipeline\) 的 `SchedulerBinding.drawFrame`持久帧回调方法. `WidgetsBinding.drawFrame`重写`RendererBinding.drawFrame`以将构建过程添加到此管道。
* 渲染管道会构建小部件，渲染管道可构建小部件，执行布局，更新合成位，绘制图层，最后将所有内容合成到一个场景中，并通过`RenderView.compositeFrame`上载到引擎。语义也会通过此过程进行更新。 
* `RenderView.compositeFrame` retains a reference to the root layer \(a `TransformLayer`\) which it recursively composites using `Layer.buildScene`. This iterates through all layers that`needsAddToScene`. If true, the layer is freshly composited into the scene. If false, previous invocations of `addToScene` will have stored an `EngineLayer` in `Layer.engineLayer`, which refers to a retained rendering of the layer subtree. A reference to this retained layer is added to the scene via `SceneBuilder.addRetained`. Once the `Scene` is built, it is uploaded to the engine via `Window.render`.

## 框架如何与引擎交互？ 

* 框架主要通过Window类进行交互，该Window类是带有进出 dart engine 的hook接口。 
* 框架的主要流程是由engine调用框架回调驱动的\(就是说主要是dart本身驱动框架\)。框架的其他入口点包括手势处理\(gesture handling\)，平台消息传递\(platform messaging\)和设备消息传递\(device messaging\)。
* Each binding serves as the singleton root of a subsystem within the framework; in several cases, bindings are layered to augment more fundamental bindings \(i.e., `WidgetsBinding` adds support for building to `RendererBinding`\). All direct framework/engine interaction is managed via the bindings, with the sole exception of the `RenderView`, which uploads frames to the engine. 

## 实现了那些功能绑定\(bindings\)?

* `GestureBinding` facilitates gesture handling across the framework, maintaining the gesture arena and pointer routing table.
  * Handles `Window.onPointerDataPacket`.
* `ServicesBinding` \(服务消息绑定\) 它辅助消息在框架与平台\(原生系统\)之间的传递.
  * Handles `Window.onPlatformMessage`.
* `SchedulerBinding` 管理各种回调（临时，持久，首帧回调和非渲染任务），跟踪生命周期状态和调度程序阶段。 也负责当需要更新界面时显式调度帧 .
  * Handles `Window.onDrawFrame`, `Window.onBeginFrame`.
  * Invokes `Window.scheduleFrame`.
* `PaintingBinding` owns the image cache which manages memory allocated to graphical assets used by the application. It also performs shader warm up to avoid stuttering during drawing \(via `ShaderWarmUp.execute` in `PaintingBinding.initInstances`\). This ensures that the corresponding shaders are compiled at a predictable time.
* `SemanticsBinding` which is intended to manage the semantics and accessibility subsystems \(at the moment, this binding mainly tracks accessibility changes emitted by the engine via `Window.onAccessibilityFeaturesChanged`\).
* `RendererBinding` implements the rendering pipeline. Additionally, it retains the root of the render tree \(i.e., the `RenderView`\) as well as the `PipelineOwner`, an instance that tracks when layout, painting, and compositing need to be re-processed \(i.e., have become dirty\). The `RendererBinding` also responds to events that may affect the application’s rendering \(including semantic state, though this will eventually be moved to the `SemanticsBinding`\).
  * Handles `Window.onSemanticsAction`, `Window.onTextScaleFactorChanged`, `Window.onMetricsChanged`, `Window.onSemanticsEnabledChanged`.
  * Invokes `Window.render` via `RenderView`. 
* `WidgetsBinding` augments the renderer binding with support for widget building \(i.e., configuring the render tree based on immutable UI descriptions\). It also retains the `BuildOwner`, an instance that facilitates rebuilding the render tree when configuration changes \(e.g., a new widget is substituted\). The `WidgetsBinding` also responds to events that might require rebuilding related to accessibility and locale changes \(though these may be moved to the `SemanticsBinding` in the future\).
  * Handles `Window.onAccessibilityFeaturesChanged`, `Window.onLocaleChanged`.
* `TestWidgetsFlutterBinding` supports the widget testing framework.

## global keys 如何运作?

* `Element.inflateWidget` 在填充小部件之前检查 **global key**。如果找到了 **global key**，则返回相应的元素（保留相应的元素并渲染子树）。
* Global keys 在对应的元素被删除\(unmounted\)时被删除 \(via `Element.unmount`\).

