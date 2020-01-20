# Text Input


## How are key events sent from the keyboard?

* `SystemChannels.keyEvent` exposes a messaging channel that receives raw key data whenever the platform produces keyboard events.
* `RawKeyboard` subscribes to this channel and forwards incoming messages as `RawKeyEvent` instances \(which encapsulate `RawKeyEventData`\). Physical and logical interpretations of the event are exposed via `RawKeyEvent.physicalKey` and `RawKeyEvent.logicalKey`, respectively. The character produced is available as `RawKeyEvent.character` but only for `RawKeyDownEvent` events. This field accounts for modifier keys / past keystrokes producing null for invalid combinations or a dart string, otherwise.
* The physical key identifies the actual position of the key that was struck, expressed as the equivalent key on a standard `QWERTY` keyboard. The logical key ignores position, taking into account any mappings or layout changes to produce the actual key the user intended.
* Subclasses of `RawKeyEventData` interpret platform-specific data to categorize the keystroke in a portable way \(`RawKeyEventDataAndroid`, `RawKeyEventDataMacOs`\)

## What is an `IME`?

* `IME` stands for “input method editor,” which corresponds to any sort of on-screen text editing interface, such as the software keyboard. There can only be one active `IME` at a time.

## How does Flutter interact with `IMEs`?

* `SystemChannels.textInput` exposes a method channel that implements a transactional interface for interacting with an `IME`. Operations are scoped to a given transaction \(client\), which is implicit once created. Outbound methods support configuring the `IME`, showing/hiding UI, and update editing state \(including selections\); inbound methods handle `IME` actions and editing changes. Convenient wrappers for this protocol make much of this seamless.

## What are the building blocks for interacting with an `IME`?

* `TextInput.attach` federates access to the `IME`, setting the current client \(transaction\) that can interact with the keyboard.
* `TextInputClient` is an interface to receive information from the `IME`. Once attached, clients are notified via method invocation when actions are invoked, the editing value is updated, or the cursor is moved.
* `TextInputConnection` is returned by `TextInput.attach` and allows the `IME` to be altered. In particular, the editing state can be changed, the `IME` shown, and the connection closed. Once closed, if no other client attaches within the current animation frame, the `IME` will also be hidden. 
* `TextInputConfiguration` encapsulates configuration data sent to the `IME` when a client attaches. This includes the desired input type \(`e.g`., “datetime”, “`emailAddress`”, “phone”\) for which to optimize the `IME`, whether to enable autocorrect, whether to obscure input, the default action, capitalization mode \(`TextCapitalization`\), and more. 
* `TextInputAction` enumerates the set of special actions supported on all platforms \(`e.g`., “`emergencyCall`”, “done”, “next”\). Actions may only be used on platforms that support them. Actions have no intrinsic meaning; developers determine how to respond to actions themselves.
* `TextEditingValue` represents the current text, selection, and composing state \(range being edited\) for a run of text.
* `RawFloatingCursorPoint` represents the position of the “floating cursor” on `iOS`, a special cursor that appears when the user force presses the keyboard. Its position is reported via the client, including state changes \(`RawFloatingCursorDragState`\).

