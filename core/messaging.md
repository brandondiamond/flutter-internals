# Messaging

## How are messages passed between the framework and platform code?

* `ServicesBinding.initInstances` sets the global message handler \(`Window.onPlatformMessage`\) to `ServicesBinding.defaultBinaryMessenger`. This instance processes messages from the platform \(via `BinaryMessenger.handlePlatformMessage`\) and allows other framework code to register message handlers \(via `BinaryMessenger.setMessageHandler`\). Handlers subscribe to a channel \(an identifier used to multiplex the single engine callback\) using an identifier shared by the framework and the platform.

## What are the building blocks of messaging?

* `BinaryMessenger` multiplexes the global message handler via channel names, supporting handler registration and bidirectional binary messaging. Sending a message produces a future that resolves to the raw response.
* `MessageCodec` defines an interface to encode and decode byte data \(`MessageCodec.encodeMessage`, `MessageCodec.decodeMessage`\). A cross-platform binary codec is available \(`StandardMessageCodec`\) as well as a JSON-based codec \(`JSONMessageCodec`\). The platform must implement a corresponding codec natively.
* `MethodCodec` is analogous to `MessageCodec` \(but otherwise independent\) encoding and decoding `MethodCall` instances that wrap a method name and a dynamic list of arguments. Method-based codecs pack and unpack results into envelopes to distinguish success and error outcomes. 
* `BasicMessageChannel` provides a thin wrapper around `BinaryMessager` that uses the provided codec to encode and decode messages to and from raw byte data.
* `MethodChannel` provides a thin wrapper around `BinaryMessager` that uses the provided method codec to encode and decode method invocations. Responses to incoming invocations are packed into envelopes indicating outcome; similarly, results from outgoing invocations are unpacked from their encoded envelope. These are returned as futures.
  * Success envelopes are unpacked and the result returned.
  * Error envelopes throw a `PlatformException`.
  * Unrecognized methods throw a `MissingPluginException` \(except when using an `OptionalMethodChannel`\).
* `EventChannel` is a helper that exposes a remote stream as a local stream. The initial subscription is handled by invoking a remote method called `listen` \(via `MethodChannel.invokeMethod`\) which causes the platform to begin emitting a stream of envelope-encoded items. A top-level handler is installed \(via `ServicesBinding.defaultBinaryMessenger.setMessageHandler`\) to unpack and forward items to an output stream in the framework. If the stream ends \(for any reason\), a remote method called `cancel` is invoked and the global handler cleared.
* `SystemChannels` is a singleton instance that provides references to messaging channels that are essential to the framework \(`SystemChannels.system`, `SystemChannels.keyEvent`, etc.\).

