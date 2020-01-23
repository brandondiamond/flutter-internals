# Asset Management

## How are assets managed?

* `AssetBundle` is a container that provides asynchronous access to application resources \(e.g., images, strings, fonts\). Resources are associated with a string-based key and can be retrieved as bytes \(via `AssetBundle.load`\), a string \(via `AssetBundle.loadString`\), or structured data \(via `AssetBundle.loadStructuredData`\). A variety of subclasses support different methods for obtaining assets \(e.g., `PlatformAssetBundle`, `NetworkAssetBundle`\). Some bundles also support caching; if so, keys can be evicted from the bundle’s cache \(via `AssetBundle.evict`\).
* `CachingAssetBundle` caches strings and structured data throughout the application’s lifetime \(unless explicitly evicted\). Binary data is not cached since the higher level methods are built atop `AssetBundle.load`, and the final representation is more efficient to store.
* Every application is associated with a `rootBundle`. This `AssetBundle` contains the resources that were packaged when the application was built \(i.e., as specified by `pubspec.yaml`\). Though this bundle can be queried directly, `DefaultAssetBundle` provides a layer of indirection so that different bundles can be substituted \(e.g., for testing or localization\).

## How are assets fetched?

* `NetworkAssetBundle` loads resources over the network. It does not implement caching; presumably, this is provided by the network layer. It provides a thin wrapper around dart’s `HttpClient`.
* `PlatformAssetBundle` is a `CachingAssetBundle` subclass that fetches resources from a platform-specific application directory via platform messaging \(specifically, `Engine`::`HandleAssetPlatformMessage`\).

