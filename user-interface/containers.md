# Containers

## What are the container building blocks?

* `Flex` is the base class for `Row` and `Column`. It implements the flex layout protocol in an axis-agnostic manner.
* `Row` is identical to `Flex` with a default axis of `Axis.horizontal`.
* `Column` is identical to `Flex` with a default axis of `Axis.vertical`.
* `Flexible` is the base class for `Expanded`. It is a parent data widget that alters its child’s flex value. Its default fit is `FlexFit.loose`, which causes its child to be laid out with loose constraints
* `Expanded` is identical to `Flexible` with a default fit of `FlexFit.tight`. Consequently, it passes tight constraints to its children, requiring them to fill all available space.

## How are flex-based containers laid out?

* All flexible containers follow the same protocol.
  * Layout children without flex factors with unbounded main constraints and the incoming cross constraints \(if stretching, cross constraints are tight\).
  * Apportion remaining space among flex children using flex factors.
    * Main axis size = `myFlex` \* \(`freeSpace` / `totalFlex`\)
  * Layout each child as above, with the resulting size as the main axis constraint. Use tight constraints for `FlexFit.tight`; else, use loose.
  * The cross extent is the max of all child cross extents.
  * If using `MainAxisSize.max`, the main extent is the incoming max constraint. Else, the main extent is the sum of all child extents in that dimension \(subject to constraints\).
  * Children are positioned according to `MainAxisAlignment` and `CrossAxisAlignment`.

## How are containers laid out?

* In short, containers size to their child plus any padding; in so doing, they respect any additional constraints provided directly or via a width or height. Decorations may be painted over this entire region. Next, a margin is added around the resulting box and, if specified, a transformation applied to the entire container.
  * If there is no child and no explicit size, the container shrinks in unbounded environments and expands in bounded ones.
* The container widget delegates to a number of sub-widgets based on its configuration. Each behavior is layered atop all previous layers \(thus, child refers to the accumulation of widgets\). If a width or height is provided, these are transformed into extra constraints.
  * If there is no child and no explicit size:
    * Shrink when the incoming constraints are unbounded \(via `LimitedBox`\); else, expand \(via `ConstrainedBox`\).
  * If there is an alignment:
    * Align the child within the parent \(via `Align`\).
  * If there is padding or the decoration has padding...
    * Apply the total padding to the child \(via `Padding`\).
  * If there is a decoration:
    * Wrap the child in the decoration \(via `DecoratedBox`\).
  * If there is a foreground decoration:
    * Wrap the child in the foreground decoration \(via `DecoratedBox`, using `DecorationPosition.foreground`\).
  * If  there are extra constraints:
    * Apply the extra constraints to the incoming constraints \(via `ConstrainedBox`\).
  * If there is a margin...
    * Apply the margin to the child \(via `Padding`\).
  * If there is a transform...
    * Transform the child accordingly \(via `Transform`\).

