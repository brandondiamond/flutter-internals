---
description: TODO
---

# Conventions - 约定

* `performAction`是内部的，`action`是外部的。
* `computeValue`是内部的，`getValue`是外部的。
  * 递归调用应使用外部变量。Recursive calls should use the external variant.
* `adoptChild` 和`dropChild` 从`AbstractNode` 调用时，必须再child实例再框架作出反应之后。
* 许多小部件由较低级的小部件组成，通常以`Raw`开头，（例如`GestureDetector`和`RawGestureDetector`，`Chip`和`RawChip`，`MaterialButton`和`RawMaterialButton`）。 `Text`和`RichText`是一个例外。



