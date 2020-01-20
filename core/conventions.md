---
description: TODO
---

# Conventions


## `performAction`\(\) is internal, action\(\) is external.


## `computeValue`\(\) is internal, `getValue`\(\) is external.

* Recursive calls should use the external variant.

## `adoptChild`\(\) and `dropChild`\(\) from `AbstractNode` must be called after updating the actual child model to allow the framework to react accordingly.


## Many widgets are comprised of lower-level widgets, generally prefixed with raw \(`e.g`., `GestureDetector` and `RawGestureDetector`, Chip and `RawChip`, `MaterialButton` and `RawMaterialButton`\). Text and `RichText` is a naming exception.


