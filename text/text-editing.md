# Text Editing


## What data structures support editable text?

* `TextRange` represents a range using \[start, end\) character indices: start is the index of the first character and end is the index after the last character. If both are -1, the range is collapsed \(empty\) and outside of the text \(invalid\). If both are equal, the range is collapsed but potentially within the text \(i.e., an insertion point\). If start &lt;= end, the range is said to be normal.
* `TextSelection` expands on range to represent a selection of text. A range is specified as a \[`baseOffset`, `extentOffset`\) using character indices. The lesser will always become the base offset, with the greater becoming the extent offset \(i.e., the range is normalized\). If both offsets are the same, the selection is collapsed, representing an insertion point; selections have a concept of directionality \(`TextSelection.isDirectional`\) which may be left ambiguous until the selection is uncollapsed. Both offsets are resolved to positions using a provided affinity. This ensures that the selection is unambiguous before and after rendering \(e.g., due to automatic line breaks\).
* `TextSelectionPoint` pairs an offset with the text direction at that offset; this is helpful for determining how to render a selection handle at a given position.
* `TextEditingValue` captures the current editing state. It exposes the full text in the editor \(`TextEditingValue.text`\), a range in that text that is still being composed \(`TextEditingValue.composing`\), and any selection present in the UI \(`TextEditingValue.selection`\). Note that an affinity isn’t necessary for the composing range since it indexes unrendered text.

## How are editing and selection overlays built?

* `TextSelectionDelegate` supports reading and writing the selection \(via `TextSelectionDelegate.textEditingValue`\), configures any associated selection actions \(e.g., `TextSelectionDelegate.canCopy`\), and provides helpers to manage selection UI \(e.g., `TextSelectionDelegate.bringIntoView`, `TextSelectionDelegate.hideToolbar`\). This delegate is utilized primarily by `TextSelectionControls` to implement the toolbar and selection handles.
* `ToolbarOptions` is a helper bundling all options that determine toolbar behavior within an `EditableText` -- that is, how the overridden `TextSelectionDelegate` methods behave.
* `TextSelectionControls` is an abstract class that builds and manages selection-related UI including the toolbar and selection handles. This class also implements toolbar behaviors \(e.g., `TextSelectionControls.handleCopy`\) and eligibility checks \(e.g., `TextSelectionControls.canCopy`\), deferring to the delegate where appropriate \(e.g., `TextSelectionDelegate.bringIntoView` to scroll the selection into view\). These checks are mainly used by `TextSelectionControls`’ build methods \(e.g., `TextSelectionControls.buildHandle`, `TextSelectionControls.buildToolbar`\), which construct the actual UI. Concrete implementations are provided for Cupertino and Material \(`\_CupertinoTextSelectionControls` and `\_MaterialTextSelectionControls`, respectively\), producing idiomatic UI for the corresponding platform. The build process is initiated by `TextSelectionOverlay`.
* `TextSelectionOverlay` is the visual engine underpinning selection UI. It integrates `TextSelectionControls` and `TextSelectionDelegate` to build and configure the text selection handles and toolbar, and `TextEditingValue` to track the current editing state; the editing state may be updated at any point \(via `TextSelectionOverlay.update`\). Updates are made build-safe by scheduling a post-frame callback if in the midst of a persistent frame callback \(building, layout, etc; this avoids infinite recursion in the build method\).
  * The UI is inserted into the enclosing Overlay and hidden and shown as needed \(via `TextSelectionOverlay.hide`, `TextSelectionOverlay.showToolbar`, etc\).
  * The toolbar and selection handles are positioned using leader/follower layers \(via `CompositedTransformLeader` and `CompositedTransformFollower`\). A `LayerLink` instance for each type of UI is anchored to a region within the editable text so that the two layers are identically transformed \(e.g., to efficiently scroll together\). When this happens, `TextSelectionOverlay.updateForScroll` marks the overlay as needing to be rebuilt so that the UI can adjust to its new position.
  * The toolbar is built directly \(via `TextSelectionControls.buildToolbar`\), whereas each selection handle corresponds to a `\_TextSelectionHandleOverlay` widget. These widgets invoke a handler when the selection range changes to update the `TextEditingValue` \(via `TextSelectionOverlay.\_handleSelectionHandleChanged`\).
* `TextSelectionGestureDetector` is a stateful widget that recognizes a sequence of selection-related gestures \(e.g., a tap followed by a double tap\), unlike a typical detector which recognizes just one. The text field \(e.g., `TextField`\) incorporates the gesture detector when building the corresponding UI.
  * `\_TextSelectionGestureDetectorState` coordinates the text editing gesture detectors, multiplexing them as described above. A map of recognizer factories is assembled and assigned callbacks \(via `GestureRecognizerFactoryWithHandlers`\) given the widget’s configuration. These are passed to a `RawGestureDetector` widget which constructs the recognizers as needed.
  * `\_TransparentTapGestureRecognizer` is a `TapGestureRecognizer` capable of recognizing while ceding to other recognizers in the arena. Thus, the same tap may be handled by multiple recognizers. This is particularly useful since selection handles tend to overlap editable text; a single tap in the overlap region is generally processed by the selection handle, whereas a double tap is processed by the editable text.
  * `TextSelectionGestureDetectorBuilderDelegate` provides a hook for customizing the interaction model \(typically implemented by the text field, e.g., `\_CupertinoTextFieldState`, `\_TextFieldState\`). The delegate also exposes the `GlobalKey` associated with the underlying `EditableTextState`.
  * `TextSelectionGestureDetectorBuilder` configures a `TextSelectionGestureDetector` with sensible defaults for text editing. The delegate is used to obtain a reference to the editable text and to customize portions of the interaction model.
  * Platform-specific text fields extend `TextSelectionGestureDetectorBuilder` to provide idiomatic interaction models \(e.g., `\_TextFieldSelectionGestureDetectorBuilder\`).

## How can editable behavior be customized?

* `TextInputFormatter` provides a hook to transform text just before `EditableText.onChange` is invoked \(i.e., when a change is committed -- not as the user types\). Blocklisting, allowlisting, and length-limiting formatters are available \(`BlacklistingTextInputFormatter`, `WhitelistingTextInputFormatter`, and `LengthLimitingTextInputFormatter`, respectively\).
* `TextEditingController` provides a bidirectional interface for interacting with an `EditableText` or subclass thereof; as a `ValueNotifier`, the controller will notify whenever state changes, including as the user types. The text \(`TextEditingController.text`\), selection \(`TextEditingController.selection`\), and underlying `TextEditingValue` \(`TextEditingController.value`\) can be read and written, even in response to notifications. The controller may also be used to produce a `TextSpan`, an immutable span of styled text that can be painted to a layer.

## How is editable text implemented?

* `EditableText` is the fundamental text input widget, integrating the other editable building blocks \(e.g., `TextSelectionControls`, `TextSelectionOverlay`, etc.\) with keyboard interaction \(via `TextInput`\), scrolling \(via Scrollable\), and text rendering to implement a basic input field. `EditableText` also supports basic gestures \(tapping, long pressing, force pressing\) for cursor and selection management and `IME` interaction. A variety of properties allow editing behavior and text appearance to be customized, though the actual work is performed by `EditableTextState`. When `EditableText` receives focus but is not fully visible, it will be scrolled into view \(via `RenderObject.showOnScreen`\).
  * The resulting text is styled and structured \(via `TextStyle` and `StrutStyle`\), aligned \(via `TextAlign`\), and localized \(via `TextDirection` and Locale\). `EditableText` also supports a text scale factor.
  * `EditableText` layout behavior is dependant on the maximum and minimum number of lines \(`EditableText.maxLines`, `EditableText.minLines`\) and whether expansion is enabled \(`EditableText.expands`\).
    * If maximum lines is one \(the default\), the field will scroll horizontally on one line.
    * If maximum lines is null, the field will be laid out for the minimum number of lines, and grow vertically.
    * If maximum lines is greater than one, the field will be laid out for the minimum number of lines, and grow vertically until the maximum number of lines is reached.
    * If a multiline field reaches its maximum height, it will scroll vertically.
    * If a field is expanding, it is sized to the incoming constraints.
  * `EditableText` follows a simple editing flow to allow the application to react to text changes and handle keyboard actions \(via `EditableTextState.\_finalizeEditing`\).
    * `EditableText.onChanged` is invoked as the field’s contents are changed \(i.e., as characters are explicitly typed\).
    * `EditableText.onEditingComplete` \(by default\) submits changes, clearing the controller’s composing bit, and relinquishes focus. If a non-completion action was selected \(e.g., “next”\), focus is retained to allow the submit handler to manage focus itself. A custom handler can be provided to alter the latter behavior.
    * `EditableText.onSubmitted` is invoked last, when the user has indicated that editing is complete \(e.g., by hitting “done”\).
* `EditableTextState` applies the configuration described by `EditableText` to implement a text field; it also manages the flow of information with the platform `TextInput` service. Additionally, the state object exposes a simplified, top-level interface for interacting with editable text. The editing value can be updated \(via `EditableTextState.updateEditingValue`\), the toolbar toggled \(via `EditableTextState.toggleToolbar`\), the `IME` displayed \(via `EditableTextState.requestKeyboard`\), editing actions performed \(via `EditableTextState.performAction`\), text scrolled into view \(via `EditableText.bringIntoView`\) and prepared for rendering \(via `EditableText.buildTextSpan`\). In this respect, `EditableTextState` is the glue binding many of the editing components together.
  * Is a: `TextInputClient`, `TextSelectionDelegate`
    * Breakdown how it is updated by the client / notifies the client of changes to keep things in sync
    * `updateEditingValue` is invoked by the client when user types on keyboard \(same for `performAction` / `floatingCursor`\).
  * `EditableTextState` participates in the keep alive protocol \(via `AutomaticKeepAliveClientMixin`\) to ensure that it isn’t prematurely destroyed, losing editing state \(e.g., when scrolled out of view\).
  * * When text is specified programmatically \(via `EditableTextState.textEditingValue`, `EditableTextState.updateEditingValue`\), the underlying `TextInput` service must be notified so that platform state remains in sync \(applying any text formatters beforehand\). `EditableTextState.\_didChangeTextEditingValue` 
* `RenderEditable`

## How are platform-specific text fields implemented?

* `TextField`
* `TextFieldState`
* `CupertinoTextField`

## How is the toolbar rendered?

* The toolbar UI is built by `TextSelectionControls.buildToolbar` using the line height, a bounding rectangle for the input \(in global, logical coordinates\), an anchor position and, if necessary, a tuple of `TextSelectionPoints`.
* `EditableText` triggers on gesture, overlay does the work

## How are selection handles rendered?

* Selection handles are visual handles rendered just before and just after a selection. Handles need not be symmetric; `TextSelectionHandleType` characterizes which variation of the handle is to be rendered \(left, right, or collapsed\). 
* Each handle is built by `TextSelectionControls.buildHandle` which requires a type and a line height.
* The handle’s size is computed by `TextSelectionControls.getHandleSize`, typically using the associated render editable’s line height \(`RenderEditable.preferredLineHeight`\), which is derived from the text painter’s line height \(`TextPainter.preferredLineHeight`\). The painter calculates this height through direct measurement.
* The handle’s anchor point is computed by `TextSelectionControls.getHandleAnchor` using the type of handle being rendered and the associated render editable’s line height, as above.
* `EditableText` triggers on select / cursor position, overlay does the work
* `EditableText` uses the handle’s size and anchor to ensure that selection handles are fully visible on screen \(via `RenderObject.showOnScreen`\).

## How does the editable retain state in response to platform lifecycle events?


## What is the best way to manage input via forms?


## `IME` \(input method editor\)?


