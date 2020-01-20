# Container Slivers


## How do slivers with multiple children manage parent data?

* SliverMultiBoxAdaptorParentData extends SliverLogicalParentData to support multiple box children \(via ContainerParentDataMixin&lt;RenderBox&gt;\) as well as the keep alive protocol \(via KeepAliveParentDataMixin\), allowing certain children to be cached when not visible. In addition to tracking each child’s layout offset and neighbors in the child list, this parent data also contains an integer index assigned by the manager.
* SliverGridParentData is a subclass of SliverMultiBoxAdaptorParentData that also tracks the child’s cross axis offset \(i.e., the distance from the child’s left or top edge from the parent’s left or top edge, depending on whether the axis is vertical or horizontal, respectively\).

## What are the building blocks for slivers with multiple children?

* RenderSliverMultiBoxAdaptor provides a base class for slivers that manage multiple box children to efficiently fill remaining space in the viewport. Generally, subclasses will create children dynamically based on the current scroll offset and remaining paint extent. A manager \(the RenderSliverBoxChildManager\) provides an interface to create, track, and remove children in response to layout; this allows a clean separation between how children are laid out and how they are produced. Subclasses implement layout themselves, delegating to the superclass as needed. Only visible and cached children are associated with the render object at any given time; cached children are managed using the keep alive protocol \(via RenderSliverWithKeepAliveMixin\).
  * Conceptually, this adaptor adds visibility pruning to the sliver protocol. Ordinarily, all slivers are laid out as the viewport is scrolled -- including those that are off screen. Using this adaptor, children are created and destroyed only as they become visible and invisible. Once destroyed, children are removed from the render tree, eliminating unnecessary building and layout.
  * Additional constraints are imposed on the children of this render object:
    * Children cannot be removed once they’ve been laid out in a given pass.
    * Children can only be added by the manager, and then only with a valid index \(i.e., an unused index\).
* RenderSliverBoxChildManager provides a bidirectional interface for managing, creating, and removing box children. This interface serves to decouple the RenderSliverMultiBoxAdaptor from the source of children. 
  * The manager uses parent data to track child indices. These are integers that provide a stable, unique identifier for children as they are created and destroyed.
* SliverMultiBoxAdaptorElement implements the RenderSliverBoxChildManager interface to link element creation and destruction to an associated widget’s SliverChildDelegate. This delegate typically provides \(or builds\) children on demand.
* RenderSliverFixedExtentBoxAdaptor is a subclass of RenderSliverMultiBoxAdaptor that arranges a sequence of box children with equal main axis extent. This is more efficient since the adaptor may reckon each child’s position directly \(i.e., without layout\). Children that are laid out are provided tight constraints.

## What are the building blocks for managing children?

* SliverChildDelegate is a subclass that assists in generating children dynamically and destroying those children when they’re no longer needed \(it also supports destruction mitigation to avoid rebuilding expensive subtrees\). Children are provided on demand \(i.e., lazily\) using a build method that accepts an index and produces the corresponding child.
* SliverChildListDelegate is a subclass of SliverChildDelegate that provides children using an explicit list. This negates the benefit of only building children when they are in view. This delegate may also wrap produced children in RenderBoundary and AutomaticKeepAlive widgets.
* SliverChildBuilderDelegate is a subclass of SliverChildDelegate that provides children by building them on demand. This improves performance by only building those children that are currently visible. This delegate may also wrap produced children in RenderBoundary and AutomaticKeepAlive widgets.

## What are the widget building blocks for dynamic slivers?

* SliverMultiBoxAdaptorWidget
* SliverChildDelegate

## How does keep alive work?

* Keep alive is a mechanism for retaining the render object, element, and state associated with an item when that item might otherwise have been destroyed \(i.e., because it is scrolled out of view\).
* RenderSliverMultiBoxAdaptor incorporates RenderSliverWithKeepAliveMixin to ensure that its associated parent data incldues the KeepAliveParentDataMixin. This mixin introduces “keepAlive” / “keptAlive” flags that form the basis of all caching decisions \(the former indicates whether keep alive behavior is requested; the latter indicates whether the item is currently being kept alive\).
* The “keepAlive” flag signals that the associated child is to be retained even when it would have been destroyed. This flag is altered by KeepAlive, a parent data widget associated with the KeepAliveParentDataMixin.
  * If an item is to be marked as keep alive, no additional work is necessary; the item must have previously been alive, else it would have been destroyed.
  * If an item is to be unmarked as keep alive, its parent must perform layout so that the item may be cleaned up \(i.e., if it would have been destroyed\).
* The KeepAlive widget can be applied out of turn \(i.e., KeepAlive.debugCanApplyOutOfTurn returns true\). This implies that the associated element can alter the child’s parent data without triggering a rebuild \(via ParentDataElement.applyWidgetOutOfTurn\). This allows parent data to be altered in the middle of building, layout, and painting.
  * This is useful when handling lists since their children may request to be kept alive during building \(via AutomaticKeepAlive\). Since enabling this feature will never invalidate the parent’s own layout \(see discussion above\), this is always safe. The benefit is that the bit can be flipped without requesting another frame.
* AutomaticKeepAlive manages an immediate KeepAlive child based on the KeepAliveNotifications emitted by descendent widgets.
  * These notifications indicate that the subtree is to be kept alive \(and must therefore use the KeepAliveParentDataMixin\).
  * A handler associated with the notification is invoked by the client to indicate that it may be released. If the client is deactivated \(i.e., removed from the tree\), it must call the handler; else, the widget will be leaked. The handler must first be called if a new KeepAliveNotification is generated.
  * AutomaticKeepAliveClientMixin provides helpers to help State subclasses appropriately manage KeepAliveNotifications.
* The “keepAlive” flag is honored by RenderSliverMultiBoxAdaptor, which maintains a keep alive bucket and ensures that the associated children remain in the render tree but are not actually rendered.

## What services must the child manager provide?

* RenderSliverBoxChildManager.childCount: the manager must provide an accurate child count if there are finite children.
* RenderSliverBoxChildManager.didAdoptChild: invoked when a child is adopted \(i.e., added to the child list\) or needs a new index \(e.g., moved within the list\). This is the only place where the child’s index is set.
  * Note that any operation that might change the index \(e.g., RenderSliverMultiBoxAdaptor.move, which is called indirectly by RenderObjectElement.updateChild when the child order changes\) will always call RenderSliverBoxChildManager.didAdoptChild to update the index.
* RenderSliverBoxChildManager.didStartLayout, RenderSliverBoxChildManager.didFinishLayout: invoked at the beginning and end of layout. 
  * In SliverMultiBoxAdaptorElement, the latter invokes a corresponding method on the delegate providing the first and last indices that are visible in the list.
* RenderSliverBoxChildManager.setDidUnderflow: indicates that the manager could not fill all remaining space with children. Generally invoked unconditionally at the beginning of layout \(i.e., without underflow\), then again if there is space left over in the viewport \(i.e., with underflow\).
  * Used to determine whether additional children will affect what’s visible in the list.
* RenderSliverBoxChildManager.estimateMaxScrollOffset: estimates the maximum scroll extent that can be consumed by all visible children. Provided information about the children current in view \(first and last indices and leading and trailing scroll offsets\).
  * In SliverMultiBoxAdaptorElement, this defers to the associated widget. If no implementation is provided, it multiples the average extent \(calculated for all visible children\) by the overall child count to obtain an approximate answer. 
  * Leading and trailing scroll offset describe the leading and trailing edges of the children, even if the leading and trailing children aren’t entirely in view.
* RenderSliverBoxChildManager.createChild: returns a child with the given index after incorporating it into the render object \(creates the initial child if a position isn’t specified\). May cache previously built children.
  * In SliverMultiBoxAdaptorElement, this method utilizes Element.updateChild to update or inflate a new child using the widget tree built by a SliverChildDelegate.
    * All children are cached once built. This cache is cleared whenever the list is rebuilt since the associated delegate may have changed \[?\].
  * If a child is updated \(e.g., because it was found in the cache\), it will already have the correct index.
  * If the child is inflated, the index will need to be set \(via RenderSliverBoxChildManager.didAdoptChild\).
    * Element.inflateWidget causes the render object \(RenderSliverMultiBoxAdaptor\) to create and mount the child, which inserts it into a slot associated with the child’s index \(via SliverMultiBoxAdaptorElement.insertChildRenderObject\). This invokes RenderSliverMultiBoxAdaptor.insert to update the render object’s child list and adopt the new child. This calls back into the manager, which sets the index appropriately \(the index is stored in a member variable\).
* RenderSliverBoxChildManager.removeChild: removes a child, generally in response to “garbage collection” when the child is no longer visible; also used to destroy a child that was previously kept alive.
  * In SliverMultiBoxAdaptorElement, this method invokes Element.updateChild with the new widget set to null, causing the child to be deactivated and detached. This invokes SliverMultiBoxAdaptorElement.removeChildRenderObject using the child’s index as the slot identifier, which calls RenderSliverMultiBoxAdaptor.remove to update the render object’s child list. Once updated, the child render object is dropped and removed. If the child had been kept alive, it is removed from the keep alive bucket, instead.

## How are children assigned indices?

* Concrete indices are provided to the child manager when producing children \(e.g., building via the child delegate\). These are computed incrementally, starting from zero. Sequential indices are computed based on items currently in view and where the child is added \(before or after the current children\). Negative indices are valid.
* The only method that updates the index stored in SliverMultiBoxAdaptorParentData is RenderSliverBoxChildManager.didAdoptChild.
* This method is invoked whenever the render object adds a child to its child list. It is not invoked when the keep alive bucket is altered \(i.e., it only considers the effective child list\).
* Changes are generally driven by the widget layer and the SliverChildDelegate in particular. As the list’s manager \(and a convenient integration point between the render tree and the widget\), SliverMultiBoxAdaptorElement facilitates the process.
  * SliverMultiBoxAdaptorElement tracks when children are created and destroyed \(either explicitly or during a rebuild\), setting SliverMultiBoxAdaptorElement.\_currentlyUpdatingChildIndex to the intended index just before the render tree is updated. The index also serves as the slot for overridden RenderObjectElement operations.
  * Updates pass through the element to the RenderSliverMultiBoxAdaptor \(according to Flutter’s usual flow\), which invokes RenderSliverBoxChildManager.didAdoptChild any time the index might change.
  * Finally, the currently updating index is written to the child’s parent data by the manager.

## How does a multi-box adaptor manage its children?

* The multi-box adaptor effectively has two child lists: the effective child list, as implemented by the ContainerRenderObjectMixin, and a keep alive bucket, managed separately.
* The keep alive bucket associates indices to cached render box children. The children in this list are kept alive, and therefore attached to the render tree, but are not otherwise visible or interactive.
  * Keep alive children are not included in the effective child list. Therefore, the helpers provided by ContainerRenderObjectMixin do not interact with these children \(though a few operations have been overridden\).
  * Keep alive children remain attached to the render tree in all other respects required by the render object protocol \(they are adopted, dropped, attached, detached, etc\).
* Both child models \(i.e., the child list and the keep alive bucket\) are considered for non-rendering operations. The keep alive bucket is ignored when iterating for semantics, notifying the manager when a child is adopted, hit testing, painting, and performing layout.
* Child render boxes are obtained via RenderSliverMultiBoxAdaptor.\_createOrObtainChild, which consults the keep alive bucket before requesting the manager to provide a child. Note that the manager incorporates an additional caching layer \(i.e., to avoid rebuilding children unless the entire list has been rebuilt\).
  * If the child is found in the keep alive bucket, it is first dropped \(i.e., because it is being moved to a different child model\) and then re-inserted into the child list. The list is marked dirty because its children have changed.
  * If the manager provides the child, it also inserts it into the render object’s child list.
* Child render boxes are destroyed by RenderSliverMultiBoxAdaptor.\_destroyOrCacheChild, which consults the keep alive flag before requesting the manager to remove the child.
  * If the child has the keep alive flag enabled, it is first removed from the child list and then readopted \(i.e., switched to the keep alive child model\). The manager is not notified of this adoption. Last, the list is marked dirty because its children have changed.
  * If the manager destroys the child, it also removes it from the render object’s child list.
* The child list may be manipulated using the usual render object container methods. Insert, remove, remove all, and move have been overridden to support both child models \(i.e., keep alive and container\).
  * RenderSliverMultiBoxAdaptor.insert cannot be used to add children to the keep alive bucket.
* The initial child is bootstrapped via RenderSliverMultiBoxAdaptor.addInitialChild to allow the initial layout offset and starting index to be specified. The child is sourced from the keep alive bucket or, if not found, the manager. If no child can be provided, underflow is indicated. Importantly, this child is not laid out.
* Subsequent children are inserted and laid out via RenderSliverMultiBoxAdaptor.insertAndLayoutLeadingChild and RenderSliverMultiBoxAdaptor.insertAndLayoutChild. The former method positions children at the head of the child list whereas the latter positions them at its tail.
  * Newly inserted leading children become RenderSliverMultiBoxAdaptor.firstChild; non-leading children may or may not become RenderSliverMultiBoxAdaptor.lastChild depending on where they’re inserted.
  * If the child was successfully inserted, it is then laid out with the provided constraints. Otherwise, underflow is indicated.
* Children that are no longer visible are removed \(or shifted to the keep alive bucket\) via RenderSliverMultiBoxAdaptor.collectGarbage. This method destroys a number of children at the start and end of the list. It also cleans up keep alive overhead, and so must always be called during layout.
  * Iterates the keep alive bucket to find children that no longer have the keep alive flag set \(recall that toggling the flag doesn’t force a rebuild\). Such children must be destroyed by the manager; this is what will actually remove them from the keep alive bucket \(via RenderSliverMultiBoxAdaptor.remove\).

## How is the multi-box adaptor rebuilt?

* Any time the adaptor is rebuilt \(e.g., because the delegate has changed\), all children must also be rebuilt. In some cases, additional children may be built, too.
  * Rebuilding may update or remove existing children, causing the list to relayout.
  * Relayout will cause additional children to be built if there is space in the viewport and the new delegate can provide them.
* Recall that updating or creating a render object \(e.g., in response to Element.updateChild\) will modify fields in that render object. Most render objects will schedule layout, painting, etc., in response to such changes.
* The element associated with the multi-box adaptor \(SliverMultiBoxAdaptorElement\) maintains an ordered cache of all child elements and widgets that have been built since the list itself was rebuilt. This list includes keep alive entries.
  * This cache must be recreated whenever the list is rebuilt since the delegate may itself produce different children.
* Rebuilding attempts to preserve children \(i.e., by matching keys\) while gracefully handling index changes. Children are updated with the rebuilt widget so that unchanged subtrees can be reused.
  * A candidate set of child elements is computed by assigning all the old elements their original indices.
  * If a child is being moved \(as determined by SliverChildDelegate.findIndexByKey\), that child is unconditionally assigned its new index.
    * If an earlier item was assigned that index, it will be overwritten and deactivated.
  * All candidate children are built and updated \(via SliverMultiBoxAdaptorElement.updateChild\). This will attempt to reuse elements where possible. If a candidate cannot be built \(i.e., because the delegate does not build anything at that index\), it is deactivated and removed.
    * The element and widget cache is repopulated with those children that are successfully built.
    * If at any point the adaptor’s child list is mutated \(i.e., because a child is added or removed\), it will be dirtied for layout, painting, and compositing.
    * If a child is being kept alive \(i.e., not visible but otherwise retained\), care is taken to avoid using when manipulating the child list \(e.g., it never serves as the “after” argument when inserting children\).
  * Last, if the list underflowed during the most recent layout attempt, the element will attempt to build one additional child \(using the next available index\). If this produces a child, it will be inserted into its parent’s child list, scheduling a relayout. This, in turn, will provide the delegate an opportunity to build additional children to fill the remaining space in the viewport.
  * The child associated with the greatest index is tracked as SliverMultiBoxAdaptorElement.\_currentBeforeChild. This serves as the “after” argument whenever the child list is manipulated, preserving the index order.

## How does the multi-box adaptor paint its children?

* The child’s position relative to the visible portion of the viewport is calculated using the incoming scroll offset and the child’s layout position in the list. These quantities are mapped to a concrete offset from the top-left corner of the canvas, taking into account the viewport’s axis direction. If the child intersects with the viewport, it is painted at the resulting offset.

## How are children laid out when their extent is fixed?

* RenderSliverFixedExtentList is a trivial RenderSliverFixedExtentBoxAdaptor subclass that lays out a sequence of children using a fixed main axis extent in order and without gaps \(cross axis extent is determined from the incoming constraints\).
* Calculations are reckoned directly \(using index and item extent\). Indices start at zero and increase along the sliver’s axis direction. Items are laid out at the position corresponding to the item’s index multiplied by item extent. Max scroll extent is trivially the number of children multiplied by item extent. Two calculations, however, are a bit confusing:
  * RenderSliverFixedExtentBoxAdaptor.getMaxChildIndexForScrollOffset: returns the maximum index for the child at the current scroll offset or earlier.
    * Conceptually, this computes the maximum index of all children preceding or intersecting the list’s leading edge \(i.e., including off-screen children\).
  * RenderSliverFixedExtentBoxAdaptor.getMinChildIndexForScrollOffset:  returns the minimum index for the child at the current scroll offset or later.
    * Conceptually, this computes the minimum index for all children exceeding or intersecting the list’s leading edge \(i.e., including on-screen children\).
  * These are identical except when children do not intersect the leading edge \(i.e., the leading edge is coincident with the seam between children\).
* Layout is performed sequentially, with new children retrieved from the manager as space allows or no more children are available. Depending on the SliverChildDelegate in use, this might lead to the interleaving of build and layout phases \(i.e., if children are built on demand rather than provided up front\). Indices are assigned sequentially as layout progresses and relevant lifecycle methods within the manager are invoked \(e.g., RenderSliverBoxChildManager.didStartLayout\).
  * Target first and last indices are computed based on the children that would be visible given the scroll offset and the remaining paint extent.
    * Shrink wrapping viewports have infinite extent. In this case, there is no last index.
  * Any children that were visible but are now outside of the target index range are garbage collected \(via RenderSliverMultiBoxAdaptor.collectGarbage\). This also cleans up any expired keep alive children.
  * If an initial child hasn’t been attached to the list, it is created but not laid out \(via RenderSliverMultiBoxAdaptor.addInitialChild\). This establishes the initial index and layout offset as all other children are positioned relative to this one.
    * If this fails, layout concludes with scroll extent and maximum paint extent set to the estimated maximum extent. All other geometry is zero.
  * If indices have become visible that precede the first child’s index, the corresponding children are built and laid out \(via RenderSliverMultiBoxAdaptor.insertAndLayoutLeadingChild\).
    * These children will need to be built since they could not have been attached to the list by assumption.
    * If one of these children cannot be built, layout halts with a scroll offset correction. Though this correction is currently computed incorrectly, the intent is to scroll the viewport such that the leading edge is coincident with the earliest, successfully built child.
  * Identify the child with the largest index that has been built and laid out so far. This is the trailing child.
    * If there were leading children, this will be the leading child adjacent to the initial child. If not, this is the initial child itself \(which is now laid out if necessary\).
  * Lay out every remaining child until there is no more room \(i.e., the target index is reached\) or no more children \(i.e., a child cannot be built\). Update the trailing child as layout progresses.
    * The trailing child serves as the “after” argument when inserting children \(via RenderSliverMultiBoxAdaptor.insertAndLayoutChild\).
    * The children may have already been attached to the list. If so, the child is laid out without being rebuilt.
    * Layout offsets are calculated directly using the child’s index and the fixed extent.
  * Compute the estimated maximum extent using the first and last index that were actually reified as well as the enclosing leading and trailing scroll offsets.
  * Return the resulting geometry:
    * SliverGeometry.scrollExtent: estimated maximum extent \(this is correct for fixed extent lists\).
    * SliverGeometry.maxPaintExtent: estimated maximum extent \(this is the most that can be painted\).
    * SliverGeometry.paintExtent: the visible portion of the range defined by the leading and trailing scroll offsets.
    * SliverGeometry.hasVisualOverflow: true if the target index wasn’t reached or the list precedes the viewport’s leading edge \(i.e., incoming scroll offset is greater than zero\).
  * If the list was fully scrolled, it will not have had an opportunity to lay out children. However, it is still necessary to report underflow to the manager.

