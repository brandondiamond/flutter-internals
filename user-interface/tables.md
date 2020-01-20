# Tables


## How is table layout described?

* `TableColumnWidth` describes the width of a single column in a `RenderTable`. Implementations can produce a flex factor for the column \(via `TableColumnWidth.flex`, which may iterate over every cell\) as well as a maximum and minimum intrinsic width \(via `TableColumnWidth.maxIntrinsicWidth` and `TableColumnWidth.minIntrinsicWidth`, which also have access to the incoming maximum width constraint\). Intrinsic dimensions are expensive to compute since they typically visit the entire subtree for each cell in the column. Subclasses implement a subset of these methods to provide different mechanisms for sizing columns.
  * `FixedColumnWidth` produces tight intrinsic dimensions, returning the provided constant without additional computation.
  * `FractionColumnWidth` applies a fraction to the incoming maximum width constraint to produce tight intrinsic dimensions. If the incoming constraint is unbounded, the resulting width will be zero.
  * `MaxColumnWidth` and `MinColumnWidth` encapsulate two `TableColumnWidth` instances, returning the greater or lesser value produced by each method, respectively.
  * `FlexColumnWidth` returns the specified flex value which corresponds to the portion of free space to be utilized by the column \(i.e., free space is distributed according to the ratio of the column’s flex factor to the total flex factor\). The intrinsic width is set to zero so that the column does not consume any inflexible space.
  * `IntrinsicColumnWidth` is the most expensive strategy, sizing the column according to the contained cells’ intrinsic widths. A flex factor allows columns to expand even further by incorporating a portion of any unclaimed space. The minimum and maximum intrinsic widths are defined as the maximum value reported by all contained render boxes \(via `RenderBox.getMinIntrinsicWidth` and `RenderBox.getMaxIntrinsicWidth` with unbounded height\).
* `TableCellVerticalAlignment` specifies how a cell is positioned within a row. Top and bottom ensure that the corresponding side of the cell and row are coincident, middle vertically centers the cell, baseline aligns cells such that all baselines are coincident \(cells lacking a baseline are top aligned\), and fill sizes cells to the height of the cell \(if all cells fill, the row will have zero height\).

## How is table appearance described?

* `TableBorder` describes the appearance of borders around and within a table. Similar to Border, `TableBorder` exposes `BorderSide` instances for each of the cardinal directions \(`TableBorder.top`, `TableBorder.bottom`, etc\). In addition, `TableBorder` describes the seams between rows \(`TableBorder.horizontalInside`\) and columns \(`TableBorder.verticalInside`\). Borders are painted via `TableBorder.paint` using row and column offsets determined by layout \(e.g., the number of pixels from the bounding rectangle’s top and left edges for horizontal and vertical borders; there will be one entry for each interior seam\).
* `RenderTable` accepts a list of decorations to be applied to each row in order. These decorations span the full extent of each row, unlike any cell-based decorations \(which would be limited to the dimensions of the cell; cells may be offset within the row due to `TableCellVerticalAlignment` and \).

## How are tables rendered?

* `TableCellParentData` extends `BoxParentData` to include the cell’s vertical alignment \(via `TableCellVerticalAlignment`\) as well as the most recent zero-indexed row and column numbers \(via `TableCellParentData.y` and `TableCellParentData.x`, respectively\). The cell’s coordinates are set during `RenderTable` layout whereas the vertical alignment is set by `TableCell`, a `ParentDataWidget` subclass.
* `RenderTable` is a render box implementing table layout and painting. Columns may be associated with a sizing strategy via a mapping from index to `TableColumnWidth` \(`columnWidths`\); a default strategy \(`defaultColumnWidth`\) and default vertical alignment \(`defaultVerticalAlignment`\) round out layout. The table is painted with a border \(`TableBorder`\) and each row’s full extent may be decorated \(`rowDecorations`, via Decoration\). `RenderBox` children are passed as a list of rows; internally, children are stored in row-major order using a single list. The number of columns and rows can be inferred from the child list, or specifically set. If these values are subsequently altered, children that no longer fit in the table will be dropped.

## How does a table manage its children?

* Children are stored in row-major order using a single list. Tables accept a flat list of children \(via `RenderTable.setFlatChildren`\), using a column count to divide cells into rows. New children are adopted \(via `RenderBox.adoptChild`\) and missing children are dropped \(via `RenderBox.dropChild`\); children that are moved are neither adopted nor dropped. Children may also be added using a list of rows \(via `RenderTable.setChildren`\); this clears all children before adding each row incrementally \(via `RenderTable.addRow`\). Note that this may unnecessarily drop children, unlike `RenderTable.setFlatChildren`.
* Children are visited in row-major order. That is, the first row is iterated in order, then the second row, and so on. This is the order used when painting; hit testing uses the opposite order \(i.e., starting from the last item in the last row\).

## How are columns widths calculated?

* A collection of `TableColumnWidth` instances describe how each column consumes space in the table. During layout, these instances are used to produce concrete widths given the incoming constraints \(via `RenderTable._computeColumnWidths`\).
  * Intrinsic widths and flex factors are computed for each column by locating the appropriate `TableColumnWidth` and passing the maximum width constraint as well as all contained cells.
    * The column’s width is initially defined as its maximum intrinsic width \(flex factor only increases this width\). Later, column widths may be reduced to satisfy incoming constraints.
    * Table width is therefore computed by summing the maximum intrinsic width of all columns.
    * Flex factors are summed for all flexible columns; maximum intrinsic widths are summed for all inflexible columns. These values are used to identify and distribute free space.
  * If there are flexible columns and room for expansion given the incoming constraints, free space is divided between all such columns. That is, if the table width \(i.e., total maximum intrinsic width\) is less than the incoming maximum width \(or, if unbounded, the minimum width\), there is room for flexible columns to expand.
    * Remaining space is defined as the relevant width constraint minus the maximum intrinsic widths of all inflexible columns.
    * This space is distributed in proportion to the ratio of the column’s flex factor to the sum of all flex factors.
    * If this would expand the column, the delta is computed and applied both to the column’s width and the table’s width.
  * If there were no flexible columns, ensure that the table is at least as wide as the minimum width constraint.
    * The difference between the table width \(i.e., total maximum intrinsic width\) and the minimum width is evenly distributed between all columns.
  * Ensure that the table does not exceed the maximum width constraint.
    * Columns may be sized using an arbitrary combination of intrinsic widths and flexible space. Some columns also specify a minimum intrinsic width. As a result, it’s not possible to resize a table using flex alone. An iterative approach is necessary to resize columns to respect the maximum width constraint without violating their other layout characteristics. The amount by which the table exceeds the maximum width constraint is the deficit.
    * Flexible columns are repeatedly shrunk until they’ve all reached their minimum intrinsic widths \(i.e., no flexible columns remain\) or the deficit has been eliminated.
      * The deficit is divided according to each column’s flex factor \(in relation to the total flex factor, which may change as noted below\).
      * If this amount would shrink the column below its minimum width, the column is clamped to this width and the deficit reduced by the corresponding delta. The column is no longer considered flexible \(reducing the total flex factor for subsequent calculations\). Otherwise, the deficit is reduced by the full amount.
      * This process is iterative because some columns cannot be shrunk by the full amount.
    * Any remaining deficit must be addressed using inflexible columns \(all flexible space has been consumed\). Columns are considered “available” if they haven’t reached their minimum width. Available columns are repeatedly shrunk until the deficit is eliminated or there are no more available columns.
      * The deficit is divided evenly between available columns.
      * If this amount would shrink the column below its minimum width, the column is clamped to this width and the deficit reduced by the corresponding delta \(this reduces the number of available columns\). Otherwise, the deficit is reduced by the full amount.
      * This process is iterative because some columns cannot be shrunk by the full amount.

## What are a table’s intrinsic dimensions?

* The table’s intrinsic widths are calculated as the sum of each column’s largest intrinsic width \(using maximum or minimum dimensions with no height constraint, via `TableColumnWidth`\).
* The table’s minimum and maximum intrinsic heights are equivalent, representing the sum of the largest intrinsic height found in each row \(using the calculated column width as input\). That is, each row is as tall as its tallest child, with the table’s total height corresponding to the sum of all such heights.
  * Concrete column widths are computed \(via `RenderTable._computeColumnWidths`\) using the width argument as a tight constraint.
  * Next, the largest maximum intrinsic height for each row is calculated \(via `RenderBox.getMaxIntrinsicHeight`\) using the calculated column width. The maximum row heights are summed to produce the table’s intrinsic height.

## How does a table layout its children?

* If the table has zero columns or rows, it’s as small as possible given the incoming constraints.
* First, concrete columns widths are calculated. These widths are incrementally summed to produce a list of x-coordinates describing the left edge of each column \(`RenderTable._columnLefts`\). The copy of this list used by layout is flipped for right-to-left locales. The overall table width is defined as the sum of all columns widths \(e.g., the last column x-coordinate plus the last column’s width\).
* Next, a list of a y-coordinates describing the top edge of each row is calculated incrementally \(`RenderTable._rowTops`\). Child layout proceeds as this list is calculated \(i.e., row-by-row\).
  * The list of row tops \(`RenderTable._rowTops`\) is cleared and seeded with an initial y-coordinate of zero \(i.e., layout starts from the origin along the y-axis\). The current row height is zeroed as are before- and after-baseline distances. These values track the maximum dimensions produced as cells within the row are laid out. The before-baseline distance is the maximum distance from a child’s top to its baseline; the after-baseline distance is the maximum distance from a child’s baseline to its bottom.
  * Layout pass: iterate over all non-null children within the row, updating parent data \(i.e., x- and y-coordinates within the table\) and performing layout based on the child’s vertical alignment \(read from parent data and set by `TableCell`, a `ParentDataWidget` subclass\).
    * Children with top, middle, or bottom alignment are laid out with unbounded height and a tight width constraint corresponding to the column’s width.
    * Children with baseline alignment are also laid out with unbounded height and a tight width constraint.
      * Children with a baseline \(via `RenderBox.getDistanceToBaseline`\) update the baseline distances to be at least as large as the child’s values.
      * Children without a baseline update the row’s height to be at least as large as the child’s height. These children are positioned at the column’s left edge and the row’s top edge \(this is the only position set during the first pass\).
    * Children with fill alignment are an exception; these are laid out during the second pass, once row height is known.
  * If a baseline is produced during the first pass, row height is updated to be at least as large as the total baseline distance \(i.e., the sum of before- and after-baseline distances\).
    * The table’s baseline distance is defined as the first row’s before-baseline distance.
  * Positioning pass: iterate over all non-null children within the row, positioning them based on vertical alignment.
    * Children with top, middle, and bottom alignment are positioned at the column’s left edge and the row’s top, middle, or bottom edges, respectively.
    * Children with baseline alignment and an actual baseline are positioned such that all baselines align \(i.e., each child’s baseline is coincident with the maximum before baseline distance\). Those without baselines have already been positioned.
    * Children with fill alignment are now laid out with tight constraints matching the row’s height and the column’s width; children are positioned at the column’s left edge and the row’s top edge.
  * Proceed to the next row by calculating the next row’s top using the row height \(and adding it to `RenderTable._rowTops`\).
* The table’s width and height \(i.e., size\) is defined as the sum of columns widths and row heights, respectively.

## How does a table paint its children?

* If the table has zero columns or rows, its border \(if defined\) is painted into a zero-height rectangle matching the table’s width.
* Each non-null decoration \(`RenderTable._rowDecorations`\) is painted via `Decoration.createBoxPainter`. Decorations are positioned using the incoming offset and the list of row tops \(`RenderTable._rowTops`\).
* Each non-null child is painted at the position calculated during layout, adjusted by the incoming offset.
* Finally, the table’s border is painted using the list of row and column edges \(these lists are filtered such that only interior edges are passed to `TableBorder.paint`\).
  * The border will be sized to match the total width consumed by columns and total height consumed by rows.
  * The painted height may fall short of the render object’s actual height \(i.e., if the total row height is less than the minimum height constraint\). In this case, there will be empty space below the table.
  * Table layout always satisfies the minimum width constraint, so there will never be empty horizontal space.

