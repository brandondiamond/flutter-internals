# Introduction

* What is this document?
  * The goal of this document is to provide intuitive descriptions of Flutter’s internals in an easily digestible Q&A format.
  * Descriptions are intended to be comprehensive \(`i.e`., low-level\) without becoming bogged down in implementation details \(`i.e`., unmaintainable\) or sacrificing clarity \(`i.e`., cryptic\).
  * This document strives to provide a “hawk’s eye view” of Flutter’s subsystems to help guide an investigation into the framework.
* Who is the audience of this document?
  * Contributors seeking to ramp up on how a particular subsystem works, developers looking to build intuition about Flutter internals, anyone looking for a guided tour through the documentation.
  * These notes were originally prepared to help “rekindle” intuition about Flutter without re-trawling through the documentation and code.
  * These notes may serve as a reference for other types of learning materials \(such as a deep dive video series\). I also hope to publish these notes as a standalone resource on the Flutter website.
  * Parts of this document may be suitable for “promotion” into the official `API` documentation to help improve clarity.
* Who can contribute to this document?
  * Anyone \(thank you\)! If there’s a corner of the framework that you find confusing, please consider updating this `FAQ` with an intuitive explanation of how things fit together -- with references to relevant code. Please feel free to add yourself to the contributors line, above.
* Won’t this document become stale?
  * Alas, this is certainly possible. While I’ve aimed to provide details that are precise without being too “fragile,” a certain amount of staleness is unavoidable.
  * I’m committed to expanding and maintaining this document as part of my everyday work with Flutter; I see this as a way to remain on top of changes to the framework as things evolve.
  * Once polished, my hope is to release this document as a properly linked `FAQ`, complete with source. In this way, the community can help keep the content up to date.
  * By telling a broader story than the `API` documentation, this `FAQ` should augment -- not compete -- with other documentation efforts.



