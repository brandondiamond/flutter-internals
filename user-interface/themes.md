---
description: TODO
---

# Themes


## How do themes work?


## How are colors managed?

* The common Color type is generally used throughout the framework. These may be organized into swatches with a single primary `aRGB` value and a mapping from arbitrary keys to Color instances. Material provides a specialization called `MaterialColor` which uses an index value as key and limits the map to ten entries \(50, 100, 200, ... 900\), with larger indices being associated with darker shades. These are further organized into a standard set of colors and swatches within the Colors class.

