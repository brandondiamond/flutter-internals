# Text Rendering

## What are the building blocks of a font?

* Each glyph \(i.e., character\) is laid out on a baseline, with the portion above forming its ascent, and the portion below forming its descent. The glyph’s origin precedes the glyph and is anchored to the baseline. A cap line marks the upper boundary of capital letters; the mean line, or median, serves the same purpose for lowercase letters. Cap height and x-height are measured from the baseline to each of these lines, respectively. Ascenders extend above the cap line or the median, depending on capitalization. Descenders extend below the baseline. Tracking denotes letter spacing throughout a unit of text; kerning is similar, but only measured between adjacent glyphs. Height is given as the sum of ascent and descent. Leading denotes additional spacing split evenly above and below the line. Collectively, height and leading comprise the line height.
* Font metrics are measured in logical units with respect to a box called the “em-square.” `Internally`, the em-square has fixed dimensions \(e.g., 1,000 units per side\). Externally, the em-square is scaled to the font size \(e.g., 12 pixels per side\). This establishes a correspondence between internal/logical units and external/physical units \(e.g., `0.012` pixels per unit\).
* Line height is determined by leading and the font’s height \(ascent plus descent\). If an explicit font height is provided, ascent and descent are constrained such that \(1\) their sum equals the desired height and \(2\) their ratio matches the original ratio. This alters the font’s spacing but not the size of  its glyphs. 
  * Note that explicitly setting the height to the font size is not the same as the default behavior.
    * When height isn’t fixed, the internal ascent and descent are used directly \(after scaling\). These values are font-specific and need not sum to the font size.
    * When height is fixed, ascent and descent have no relation to their internal counterparts \(other than sharing a ratio\). These values are chosen to sum to the target height \(e.g., the font size\).
* Text size is further adjusted according to a text scale factor \(an accessibility feature\). This factor represents the number of logical pixels per font size pixel \(e.g., a value of `1.5` would cause fonts to appear 50% larger\). This effect is achieved my multiply the font size by the scale factor.
* A given paragraph of text may contain multiple runs. Each run \(or text box\) is associated with a specific font and may be broken up due to wrapping. Adjacent runs can be aligned in different ways, but are generally positioned to achieve a common baseline. Boxes with different line height can combine to create a larger line height depending on positioning. Overall height is generally measured from the top of the highest box to the bottom of the lowest.

## What are the building blocks for describing text?

* `FontStyle` and `FontWeight` characterize the glyphs used during rendering \(specifying slant and glyph thickness, respectively\).
* `FontFeature` encodes a feature tag, a four-character tag associated with an integer value that customizes font-specific features \(i.e. these can be anything from enabling slashed zeros to selecting random glyph variants\).
* `TextAlign` describes the horizontal alignment of text. “Left”, “right”, and “center” describe the text’s alignment with respect to its container. “Start” and “end” do the same, but in a directionality-aware manner. “Justify” stretches wrapped text such that it fills the available width.
* `TextBaseline` identifies the horizontal lines used to vertically align glyphs \(alphabetic or ideographic\).
* `TextDecoration` is a linear decoration \(underline, overline, or a strikethrough\) that can be applied to text; overlapping decorations merge intelligently. `TextDecorationStyle` alters how decorations are rendered: using a solid, double, dotted, dashed, or wavy line.
* `TextStyle` describes the size, position, and rendering of text in a way that can be transformed for use by the engine. This description includes colors, spacing, the desired font family, the text decoration, and so on.
  * Specifying a fixed height generally alters the font’s default spacing. Ordinarily, the font’s ascent and descent \(space above and below the baseline\) is calculated without constraint. When a height is specified, both metrics must sum to the desired height while maintaining their original ratio; this is simply different than the default behavior.
* `TextPosition` represents an insertion point in rendered and unrendered text \(i.e., a position between letters\). This is encoded as an offset indicating the index of the letter immediately after the position, even if that index exceeds string bounds. An affinity is used to disambiguate the following cases where an offset becomes ambiguous after rendereding:
  * Positions adjacent to automatic line breaks are ambiguous: the insertion point might be the end of the first line or the start of the second \(with explicit line breaks, the newline is just another character, so there is no ambiguity\).
  * Positions at the interface of right-to-left and left-to-right strings are ambiguous: the renderer will flip half the string, so it’s unclear whether the offset corresponds to the pre-render or post-render version \(`e.x`., offset 3 can be “abc\|`ABC`” or “`abcCBA`\|”\).
* `TextAffinity` resolves ambiguity when an offset can correspond to multiple positions after rendering \(e.g., because a renderer might insert line breaks or flip portions due to directionality\). `TextAffinity.upstream` selects the option closer to the start of the string \(e.g., the end of the line before a break, “abc\|`ABC`”\) whereas `TextAffinity.downstream` selects the option closer to the end \(e.g., the start of the line after a break, “`abcCBA`\|”\).
* `TextBox` identifies a rectangular region containing text relative to the parent’s top left corner. Provides direction-aware accessors \(i.e., `TextBox.start`, `TextBox.end`\).
* `TextWidthBasis` enumerates approaches for measuring the width of a paragraph. Longest line selects the minimum space needed to contain the longest line \(e.g., a chat bubble\). Parent selects the width of the container for multi-line text or the actual width for a single line of text \(e.g., a paragraph of text\).
* `RenderComparison` describes the renderable difference between two inline spans. Spans may have differences that will affect their layout \(and painting\), their painting, their metadata, etc.

## What are the building blocks for describing paragraph layout?

* `ParagraphStyle` describes how lines are laid out by `ParagraphBuilder`. Among other things, this class allows a maximum number of lines to be set as well as the text’s directionality and ellipses behavior; crucially, it allows the paragraph’s strut to be configured.
* `ParagraphConstraints` describe the input to `Paragraph` layout. Its only value is “width,” which specifies a maximum width for the paragraph. This maximum is enforced in two ways: \(1\) if possible, a soft line break \(i.e., located between words\) is inserted before the maximum is reached. Otherwise, \(2\) a hard line break \(i.e., located within words\) is inserted, instead.
  * If this would result in an empty line \(i.e., due to inadequate width\), the next glyph is inserted irrespective of the constraint, followed by a hard line break.
  * This width is used when aligning text \(via `TextAlign`\); any ellipses is ignored for the purposes of alignment. Ellipses length is considered when determining line breaks \[?\].
* `StrutStyle` defines a “strut” which dictates a line of text’s minimum height. Glyphs assume the larger of their own dimensions and the strut’s dimensions \(with ascent and descent considered separately\). Conceptually, paragraph’s prepend a zero-width strut character to each line. The strut can be forced, causing line height to be determined solely by the strut.

## What are the building blocks for placeholders in content?

* Placeholders reserve rectangular spaces within paragraph layout. These spaces are subsequently painted using arbitrary content \(e.g., a widget\).
* `PlaceholderAlignment` expresses the vertical alignment of a placeholder relative to the font. Placeholders can have their top, bottom, or baseline aligned to the parent baseline. Placeholders may also be positioned relative to the font’s ascenders, descenders, or median.
* `PlaceholderDimensions` describe the size and alignment of a placeholder. If a baseline-relative alignment is used, the type of baseline must be specified \(e.g., alphabetic or ideographic\). An optional baseline offset indicates the distance from the top of the box to its logical baseline \(e.g., this is used to align inline widgets via `RenderBox.getDistanceToBaseline`\).

## What are the building blocks for inline spans of content?

* `InlineSpan` represents an immutable span of content within a paragraph. A span is associated with a text style which is inherited by child spans. Spans are added to a `ParagraphBuilder` via `InlineSpan.build` \(this is handled automatically by `TextPainter`\); any accessibility text scaling is specified at this point. An ancestor span may be queried to locate the descendent span containing a `TextPosition`.
* `TextSpan` extends `InlineSpan` to represent an immutable span of styled text. The provided `TextStyle` is inherited by all children, which may override all or some styles. Text spans contain text as well as any number of inline span children. Children form a text span tree that is traversed in order. Each node is also associated with a string of text \(styled using the span’s `TextStyle`\) which effectively precedes any children. `TextSpans` are interactive and have an associated gesture recognizer that is managed externally \(e.g., by `RenderParagraph`\).
* `PlaceholderSpan` extends `InlineSpan` to represent a reserved region within a paragraph.
* `WidgetSpan` extends `PlaceholderSpan` to embed a `Widget` within a paragraph. Widgets are constrained to the maximum width of the paragraph. The widget is laid out and painted by `RenderParagraph`; `TextPainter` simply leaves space within the paragraph.
  * `ParagraphBuilder` tracks the number of placeholders added so far; this number is used when building to identify the `PlaceholderDimensions` corresponding to this span. These dimensions are typically computed by `RenderParagraph`, which lays out all widget spans before building the paragraph so that the necessary amount of space is reserved.

## What are the building blocks for building paragraphs?

* Paragraph is a piece of text wherein each glyph is sized and positioned appropriately; `Paragraph.layout` must be invoked to compute these metrics. Paragraph supports efficient resizing and painting \(via `Canvas.addParagraph`\) and must be built by `ParagraphBuilder`. Each glyph is assigned an integer offset computed before rendering \(thus there is no affinity issue\). Note that `Paragraph` is a thin wrapper around engine code.
  * After layout, paragraphs can report height, width, longest line width, and intrinsic width. Maximum intrinsic width maximally reduces the paragraph’s height. Minimum intrinsic width is the smallest width allowing correct rendering.
  * Paragraphs can report all placeholder locations and dimensions \(reported as `TextBox` instances\), the `TextPosition` associated with a 2D offset, and word boundaries given an integer offset.
  * Once laid out, `Paragraphs` can provide bounding boxes for a range within the pre-rendered text. Boxes are derived from runs of text, each of which may have a distinct style and therefore height. Thus, boxes can be sized in a variety of ways \(via `BoxHeightStyle`, `BoxWidthStyle`\). Boxes can tightly enclose only the rendered glyphs \(the default\), expand to include different portions of the line height, be sized to the maximum height in a given line, etc. 
* `ParagraphBuilder` assembles a single `Paragraph` from a sequence of text and placeholders using the provided `ParagraphStyle`. `TextStyles` are pushed and popped, allowing styles to be inherited and overridden, and placeholders boxes \(e.g., for embedded widgets\) are reserved and tracked. The `Paragraph` itself is built by the engine.

## What are the text rendering building blocks?

* `TextOverflow` describes how visual overflow is handled when rendering text via `RenderParagraph`. Options include clipping, fading, adding ellipses, or tolerating overflow.
* `TextPainter` performs the actual work of building a `Paragraph` from an `InlineSpan` tree and painting it to the canvas; the caller must explicitly request layout and painting. The painter incorporates the various text rendering `APIs` to provide a convenient interface for rendering text. If the container’s width changes, layout and painting must be repeated.
  * Text layout selects a width that is within the provided minimum and maximum values, but that is as close to the maximum intrinsic width as possible \(i.e., consumes the least height\). Redundant calls are ignored.
  * `TextPainter` supports a variety of queries; it can provide bounding boxes for a `TextSelection`, the height and offset of the glyph at a given `TextPosition`, neighboring editable offsets, and can convert a 2D offset into a `TextPosition`.
  * A preferred line height can be computed by rendering a test glyph and measuring the resulting height \(via `TextPainter.preferredLineHeight`\).
* Text is a wrapper widget for `RichText` which obtains the ambient text style via `DefaultTextStyle`.
* `RichText` is a `MultiChildRenderObjectWidget` that configures a single `RenderParagraph` with a provided `InlineSpan`. Its children are obtained by traversing this span to locate `WidgetSpan` instances. Any render objects produced by the associated widgets will be attached to the `RichText`’s `RenderParagraph`; this is how the `RenderParagraph` is able to layout and paint widgets into the paragraphs’ placeholders.
* `TextParentData` is the parent data associated with the inline widgets contained in a paragraph of text. It extends `ContainerBoxParentData` to track the text scale applied to the child.
* `RenderParagraph` displays a paragraph of text, optionally containing inline widgets. It delegates to \(and augments\) an underlying `TextPainter` which builds, lays out, and paints the actual paragraph. Any operations that depend on text layout generally recompute layout blindly; this is acceptable since `TextPainter` ignores redundant calls.

## How is text laid out by the render tree?

* `RenderParagraph` adapts a `TextPainter` to the render tree, adding the ability to render widgets \(i.e., `WidgetSpan` instances\) within the text. Any such widgets are parented to the `RenderParagraph`; a list of all descendent placeholders \(`RenderParagraph._placeholderSpans`\) is also cached. The `RenderParagraph` proxies the various inputs to the text rendering system, invalidating painting and layout as values change \(`RenderComparison` allows this to be done efficiently\).
* Layout is performed in three stages: inline children are laid out to determine placeholder dimensions, text is laid out with placeholder dimensions defined, and children are positioned using the final placeholder positions computed by the engine. Finally, the desired `TextOverflow` effect is applied \(by clipping, configuring an overflow shader, etc\).
* Children \(i.e., inline widgets\) are laid out in sequence with infinite height and a maximum width matching that of the paragraph. A list of `PlaceholderDimensions` is assembled using the resulting layout. If the associated `InlineSpan` is aligned to the baseline, the offset to the child’s baseline is computed \(via `RenderBox.getDistanceToBaseline`\); this ensures that the child’s baseline is coincident with the text’s baseline. Finally, the dimensions are bound to the `TextPainter` \(via `TextPainter.setPlaceholderDimensions`\).
* Next, the box constraints are converted to `ParagraphConstraints` \(i.e., height is ignored\). If wrapping or truncation is enabled, the maximum width is retained; otherwise, the width is considered infinite. These constraints are provided to `TextPainter.layout`, which builds and lays out the underlying `Paragraph` \(via the engine\).
* Finally, children are positioned based on the final text layout. Positions are read from `TextPainter.inlinePlaceholderBoxes`, which exposes an ordered list of `TextBox` instances.

## How is text painted by the render tree?

* Since `TextPainter` is stateful, and certain operations \(e.g., computing intrinsic dimensions\) destroy this state, the text must be relaid out before painting.
* Any applicable clip is applied to the canvas; if the `TextOverflow` setting requires a shader, the canvas is configured accordingly.
* The `TextPainter` paints the text \(via `Canvas.drawParagraph`\). Empty spaces will appear in the text wherever placeholders were included.
* Finally, each child is rendered at the offset corresponding to the associated placeholder in the text. If applicable, the text scale factor is applied to the child during painting \(e.g., causing it to appear larger\).

## How is text made interactive?

* All `InlineSpan` subclasses can be associated with a gesture recognizer \(`InlineSpan.recognizer`\). This instance’s lifecycle, however, must be maintained by client code \(i.e., the `RenderParagraph`\). Additionally, events must be propagated to each `InlineSpan` since spans do not support bubbling themselves.
  * The `RenderParagraph` overrides `RenderObject.handleEvent` to attach new pointers \(i.e., `PointerDownEvent`\) directly to the span that was tapped.
  * The affected span is identified by first laying out the text, mapping the event’s offset to a `TextPosition`, then finally locating the span that contains this position \(via `TextPainter.getPositionForOffset` and `InlineSpan.getSpanForPosition`, respectively\).
* The `RenderParagraph` is also responsible for hit testing any render objects included via inline widgets. These are hit tested using a similar approach as `RenderBox` that additionally takes into account any text scaling.
* The `RenderParagraph` is always included in the hit test result. 

## How are text’s intrinsic dimensions calculated?

* Intrinsic dimensions cannot be calculated if placeholders are to be aligned relative to the text’s baseline; this would require a full layout pass according to the `RenderBox` layout protocol.
* Intrinsics are dependant on two things: inline widgets \(children\) and text layout. Children must be measured to establish the size of any placeholders in the text. Next, the text is laid out using the resulting placeholder dimensions so its that intrinsic dimensions can be retrieved \(`Paragraph.layout` is required to obtain the intrinsic dimensions from the engine\).
* The minimum and maximum intrinsic width of text is computed without constraint on width. The minimum value corresponds to the width of the first word and the maximum value corresponds to the full string laid out on a single line.
* The minimum and maximum intrinsic height of text is computed using the provided width \(text layout is width-in-height-out\). Moreover, the maximum and minimum values are equivalent: the text’s height is solely determined by its width. Thus, both values match the height of the text when rendered within the given width.
* Intrinsic dimensions for all children are queried via the `RenderBox` protocol. The input dimension \(e.g., height\) is provided to obtain the opposite intrinsic dimension \(e.g., minimum intrinsic width\); this value is used to obtain the remaining intrinsic dimension \(e.g., minimum intrinsic height\). These values are then wrapped in a `PlaceholderDimension` instance which is aligned appropriately \(via `RenderParagraph._placeholderSpans`\). 
  * Variants obtaining minimum and maximum intrinsic dimensions are equivalent other than the render box methods they invoke.
  * Both the intrinsic width and height children must be computed since the placeholder boxes affect both dimensions of text layout. However, when measuring maximum intrinsic width, height can be ignored since text is laid out in a single line.
* The intrinsic sizes of all children are provided to the text painter prior to layout \(via `TextPainter.setPlaceHolderDimensions`\). This ensures that the resulting intrinsics accurately reflect any inline widgets. After layout, intrinsics can be read from the `Paragraph` directly. Note that this is destructive in that it wipes out earlier dimensions associated with the `TextPainter`.

## How is text directionality handled by the framework?

* Unlike other frameworks, `Flutter` does not have a default text direction \(`TextDirection`\). Throughout the lower levels of the framework, directionality must be specified. At the widget level, an ambient directionality is introduced by the `Directionality` widget. Other widgets use the ambient directionality when interacting with lower level aspects of the framework.
* Often, a visual and a directional variant of a widget is provided \(`EdgeInsets` vs. `EdgeInsetsDirectional`\). The former exposes top, left, right, and bottom properties; the latter exposes top, start, end, and bottom properties.
* Painting is a notable exception; the canvas \(Canvas\) is always visually oriented, with the X and Y axes running from left-to-right and top-to-bottom, respectively.

## How does the engine layout text?

* Minikin measures and lays out the text.
  * Minikin uses `ICU` to split text into lines.
  * Minikin uses `HarfBuzz` to retrieve glyphs from a font.
* Skia paints the text and text decorations on a canvas.

