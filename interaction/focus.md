# Focus


## What are the focus building blocks?

* FocusManager, stored in the WidgetsBinding, tracks the currently focused node and the most recent node to request focus. It handles updating focus to the new primary \(if any\) and maintaining the consistency of the focus tree by sending appropriate notifications. The manager also bubbles raw keyboard events up from the focused node and tracks the current highlight mode.
* FocusTraversalPolicy dictates how focus moves within a focus scope. Traversal can happen in a TraversalDirection \(TraversalDirection.left, TraversalDirection.up\) or to the next, previous, or first node \(i.e., the node to receive focus when nothing else is focused\). All traversal is limited to the node’s closest enclosing scope. The default traversal policy is the ReadingOrderTraversalPolicy but can be overridden using the DefaultFocusTraversal inherited widget. FocusNode.skipTraversal can be used to allow a node to be focusable without being eligible for traversal.
* FocusAttachment is a helper used to attach, detach, and reparent a focus node as it moves around the focus tree, typically in response to changes to the underlying widget tree. As such, this ensures that the node’s focus parent is associated with a context \(BuildContext\) that is an ancestor of its own context.  The attachment instance delegates to the underlying focus node using inherited widgets to set defaults -- i.e., when reparenting, Focus.of / FocusScope.of are used to find the node’s new parent.
* FocusNode represents a discrete focusable element within the UI. Focus nodes can request focus, discard focus, respond to focus changes, and receive keyboard events when focused. Focus nodes are grouped into collections using scopes which are themselves a type of focus node. Conceptually, focus nodes are entities in the UI that can receive focus whereas focus scopes allow focus to shift amongst a group of descendant nodes. As program state, focus nodes must be associated with a State instance.
* FocusScopeNode is a focus node subclass that organizes focus nodes into a traversable group. Conceptually, a scope corresponds to the subtree \(including nested scopes\) rooted at the focus scope node. Descendent nodes add themselves to the nearest enclosing scope when receiving focus \(FocusNode.\_reparent calls FocusNode.\_removeChild to ensure scopes forget nodes that have moved\). Focus scopes maintain a history stack with the top corresponding to the most recently focused node. If a child is removed, the last focused scope becomes the focused child; this process will not update the primary focus.
* Focus is a stateful widget that manages a focus node that is either provided \(i.e., so that an ancestor can control focus\) or created automatically. Focus.of establishes an inherited relationship causing the dependant widget to be rebuilt when focus changes. Autofocus allows the node to request focus within the enclosing scope if no other node is already focused.
* FocusScope is the same as above, but manages a focus scope node.
* FocusHighlightMode and FocusHighlightStrategy determine how focusable widgets respond to focus with respect to highlighting. By default the highlight mode is updated automatically by tracking whether the last interaction was touch-based. The highlight mode is either FocusHighlightMode.traditional, indicating that all controls are eligible, and FocusHighlightMode.touch, indicating that only those controls that summon the soft keyboard are eligible.

## What is focus?

* Focus determines the UI elements that will receive raw keyboard events.
* The primary focused node is unique within the application. As such, it is the default recipient of keyboard events, as forwarded by the FocusManager. 
* All nodes along the path from the root focus scope to the primary focused node have focus. Keyboard events will bubble along this path until they are handled.
* Moving focus \(via traversal or by requesting focus\) does not call FocusNode.unfocus, which would remove it from the enclosing scope’s stack of recently focused nodes, also ensuring that a pending update doesn’t refocus the node.
* Controls managing focus nodes may choose to render with a highlight in response to FocusNode.highlightMode.

## What is the focus tree?

* The focus tree is a sparse representation of focusable elements in the UI consisting entirely of focus nodes. Focus nodes maintain a list of children, as well as assessors that return the descendants and ancestors of a node in furthest and closest order, respectively.
* Some focus nodes represent scopes which demarcate a subtree rooted at that node. Conceptually, scopes serve as a container for focus traversal operations.
* Both scopes and ordinary nodes can receive focus, but scopes pass focus to the first descendent node that isn’t a scope. Even if a scope is not along the path to the primary focus, it tracks a focused child \(FocusScopeNode.focusedChild\) that would receive focus if it were.
* Scopes maintain a stack of previously focused children with the top of the stack being the first to receive focus if that scope receives focus \(the focused child\). When the focused child is removed, focus is shifted to the previous holder. If a focused child is a scope, it is called the scope’s first focus.
* When a node is focused, all enclosing scopes ensure that their focused child is either the target node or a scope that’s one level closer to the target node.
* As focus changes, affected nodes receive focus notifications that clients can use to update the UI. The focused node also receives raw keyboard events which bubble up the focus path.

## How is focus attached to the widget tree?

* A FocusAttachment ensures a FocusNode is anchored to the correct parent when the focus tree changes \(e.g., because the widget tree rebuilds\). This ensures that the associated build contexts are consistent with the actual widget tree.
* As a form of program state, every focus node must be hosted by a StatefulWidget’s State instance. When state is initialized \(State.initState\), the focus node is attached to the current BuildContext \(via FocusNode.attach\), returning a FocusAttachment handle.
* When the widget tree rebuilds \(State.build, State.didChangeDependencies\), the attachment must be updated via FocusAttachment.reparent. If a widget is configured to use a new focus node \(State.didUpdateWidget\), the previous attachment must be detached \(FocusAttachment.detach, which calls FocusNode.unfocus as needed\) before attaching the new node. Finally, the focus node must be disposed when the host state is itself disposed \(FocusNode.dispose, which calls FocusAttachment.detach as needed\).
* Reparenting is delegated to FocusNode.\_reparent, using Focus.of / FocusScope.of to locate the nearest parent node. If the node being reparented previous had focus, focus is restored through the new path via FocusNode.\_setAsFocusedChild.
* A focus node can have multiple attachments, but only one attachment may be active at a time.

## How is focus managed?

* FocusManager tracks the root focus scope as well as the current \(primary\) and requesting \(next\) focus nodes. It also maintains a list of dirty nodes that require update notifications.
* As nodes request focus \(FocusNode.requestFocus\), the manager is notified that the node is dirty \(FocusNode.\_markAsDirty\) and that a focus update is needed \(FocusManager.\_markNeedsUpdate\), passing the requesting node. This sets the next focus node and schedules a microtask to actually update focus. This can delay focus updates by up to one frame.
* This microtask promotes the requesting node to primary \(FocusManager.\_applyFocusChange\), marking all nodes from the root to the incoming and outgoing nodes, inclusive, as being dirty. Then, all dirty nodes are notified that focus has changed \(FocusNode.\_notify\) and marked clean.
* The new primary node updates all enclosing scopes such that they describe a path toward it via each scope’s focused child \(FocusNode.\_setAsFocusedChild\).

## How is focus requested?

* When a node requests focus \(FocusNode.requestFocus\), every enclosing scope is updated to focus toward the requesting node directly or via an intermediate scope \(FocusNode.\_setAsFocusedChild\). The node is then marked dirty \(FocusNode.\_markAsDirty\) which updates the manager’s next node and requests an update.
* When a scope requests focus \(FocusScopeNode.requestFocus\), the scope tree is traversed to find the first descendent node that isn’t a scope. Once found, that node is focused. Otherwise, the deepest scope is focused.
* The focus manager resolves focus via FocusManager.\_applyFocusChange, promoting the next node to the primary and sending notifications to all dirty nodes.

## How is focus relinquished?

* When a node is unfocused, the enclosing scope “forgets” the node and the manager is notified \(via FrameManager.\_willUnfocusNode\).
  * If the node had been tracked as primary or next, the corresponding property is cleared and an update scheduled. If a next node is available, that node becomes primary. If there is no primary and no next node, the root scope becomes primary.
  * The deepest non-scope node will not be automatically focused. However, future traversal will attempt to identify the current first focus \(FocusTraversalPolicy.findFirstFocus\) when navigating.
* When a node is detached, if it had been primary, it is unfocused as above. It is then removed from the focus tree.
* When a node is disposed, the manager is notified \(via FrameManager.\_willDisposeFocusNode\) which calls FrameManager.\_willUnfocusNode, as above. Finally, it is detached.
* Focus updates are scheduled once per frame. As a result, state will not stack but resolve to the most recent request.

## What is a keyboard token?

* Some controls display the soft keyboard in response to focus. The keyboard token is a boolean tracking whether focus was requested explicitly \(via FocusNode.requestFocus\) or assigned automatically due to another node losing focus.
* Controls that display the keyboard consume the token \(FocusNode.consumeKeyboardToken\), which returns its value and sets it to false. This ensures that the keyboard is shown exactly once in response to explicit user interaction.

