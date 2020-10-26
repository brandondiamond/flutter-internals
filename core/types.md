# Types - 类型

## 什么类型用来描述2d坐标 positions?

* `OffsetBase`表示二维（2D）轴对齐向量。子类是不可变的，并且可以使用标准运算符进行比较。\(重载了运算符 +, - 等等\)
* `Offset` 是 `OffsetBase` 子类, 可以将其理解为笛卡尔空间或向量中的点。偏移量可以使用标准运算符进行代数运算。`＆`运算符允许通过将偏移量与大小结合起来来构造`Rect`（偏移量标识矩形的左上角）。偏移~~量~~可以被用于插值计算。\(插值：用来填充图像变换时像素之间的空隙。不理解的自行学习基础知识\)
*  `Point`类，用于表示笛卡尔平面上的2D点。

## 什么类型用来描述大小范围 magnitudes?

* `Size`是一个`OffsetBase`子类，它表示宽度和高度。从几何角度来看,它描述了一个左顶点在坐标原点的矩形. 多个Size可以使用标准运算符进行代数运算。+运算符会根据提供的增量（通过“偏移”）来扩展大小。. 他们可以被用于插值计算
* `Radius` 描述圆形或椭圆形的半径。r与坐标系xy相交. 圆的每个半径值一样.  同上,每个r都能用操作符计算,也能计算插值.
* \`\`

## 什么类型用来描述区域面积 regions?

* `Rect`是一个不变的，二维的，与轴对齐的浮点矩形，其坐标相对于给定的原点。可以通过各种方式（例如，通过其中心，通过边界圆，通过其左侧，顶部，右侧和底部边缘的大小等）来描述矩形，或者通过组合“`Offset`”和“`Size`”来构造矩形。 矩形可以放大，缩小，组合，相交，平移，查询等等。可以比较矩形是否相等并进行插值。 
* `RRect` augments a `Rect` with four independent radii \(via `Radius`\) corresponding to its corners. Rounded rectangles can be described in various ways \(e.g., by offsets to each of its sides and one or more radii, by a bounding box fully enclosing the rounded rectangle with one or more radii, etc.\). Rounded rectangles define a number of sub-rectangles: a bounding rectangle \(`RRect.outerRect`\), an inner rectangle with left and right edges matching the base rectangle and top and bottom edges inset to coincide with the rounded corners' centers \(`RRect.wideMiddleRect`\), a similar rectangle but with the sets reversed \(`RRect.tallMiddleRect`\), and a rectangle that is the intersection of these two \(`RRect.middleRect`\). A rounded rectangle is said to describe a “stadium” if it possesses a side with no straight segment \(e.g., entirely drawn by the two rounded corners\). Rounded rectangles can be interpolated.

## 什么类型用来描述坐标空间 coordinate spaces?

* `Axis` represents the X- or Y-axis \(horizontal or vertical, respectively\) relative to a coordinate space. The coordinate space can be arbitrarily transformed and therefore need not be parallel to the screen’s edges.
* `AxisDirection` applies directionality to an axis. The value represents the direction in which values increase along an associated axis, with the origin rooted at the opposite end \(e.g., `AxisDirection.down` positions the origin at the top with positive values growing downward\).
* `GrowthDirection` is the direction of growth relative to the current axis direction \(e.g., how items are ordered along the axis\). `GrowthDirection.forward` implies an ordering consistent with the axis direction \(the first item is at the origin with subsequent items following\). `GrowthDirection.reverse` is exactly the opposite \(the last item is at the origin with preceding items following\).
  * Growth direction does not flip the meaning of “leading” and “trailing,” it merely determines how children are ordered along a specified axis.
  * For a viewport, the origin is positioned according to an axis direction \(e.g., `AxisDirection.down` positions the origin at the top of the screen, `AxisDirection.up` positions the origin at the bottom of the screen\), with the growth direction determining how children are ordered starting from the origin. As a result, both pieces of information are necessary to determine where a set of slivers should actually appear.
* `ScrollDirection` represents the user’s scroll direction relative to the positive scroll offset direction \(i.e., the direction in which positive scroll offsets increase as determined by axis direction and growth direction\). Includes an idle state \(`ScrollDirection.idle`\).
  * This is typically subject to the growth direction \(e.g., the scroll direction is flipped when growth is reversed\).
  * Confusingly, this refers to the direction the content is moving on screen rather than where the user is scrolling \(e.g., scrolling down a web page causes the page’s contents to move upward; this would be classified as `ScrollDirection.reverse` since this motion is opposite the axis direction\).

## 什么类型用来描述图形 graphics?

* `Color` 是一个32位不变量，描述alpha，红色，绿色和蓝色通道。可以使用从零到一的不透明度值定义Alpha。可以插值颜色并将其转换为亮度值。
* `Shadow` 提供了颜色，投影元素的偏移以及模糊半径的单个投影阴影，该模糊半径表示应用于阴影的高斯模糊。
* `Gradient` 描述一种或多种平滑的颜色过渡。可以对渐变进行插值和缩放； 渐变也可以用于获取对相应着色器的引用。支持线性，径向和扫描渐变（分别通过`LinearGradient`，`RadialGradient`和`SweepGradient`）。`TileMode` 确定渐变如何绘制超出其定义的边界。Gradients may be clamped \(e.g., hold their initial and final values\), repeated \(e.g., restarted at their bounds\), or mirrored \(e.g., restarted but with initial and final values alternating\).

## 树节点如何建模？ ‌

* `AbstractNode` represents a node in a tree without specifying a particular child model \(i.e., the tree's actual structure is left as an implementation detail\). Concrete implementations must call `AbstractNode.adoptChild` and `AbstractNode.dropChild` whenever the child model changes.
  * `AbstractNode.owner` references an arbitrary object shared by all nodes in a subtree.
    * `AbstractNode.attach` assigns an owner to the node. Adopting children will attach them automatically. Used by the owner to attach the tree via its root node.
      * Subclasses should attach all children since the parent can change its attachment state at any time \(i.e., after the child is adopted\) and must keep its children in sync.
    * `AbstractNode.detach` clears a node's owner. Dropping children will detach them automatically. Used by the owner to detach the tree via its root node.
      * Subclasses should detach all children since the parent can change its attachment state at any time \(i.e., after the child is adopted\) and must keep its children in sync.
    * `AbstractNode.attached` indicates whether the node is attached \(i.e., has an owner\).
  * `AbstractNode.parent` references the parent abstract node.
    * `AbstractNode.adoptChild` updates a child's parent and depth. The child is attached if the parent has an owner.
    * `AbstractNode.dropChild` clears the child's parent. The child is detached if the parent has an owner.
  * `AbstractNode.depth` is an integer that increases with depth. All depths below a given node will be greater than that node's depth. Note that values need not match the actual depth of the node.
    * `AbstractNode.redepthChild` updates a child's depth to be greater than its parent.
    * `AbstractNode.redepthChildren` uses the concrete child model to call `AbstractNode.redepthChild` on each child.

