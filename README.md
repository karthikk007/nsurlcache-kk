# nsurlcache-kk

# To `NSCache` or not to `NSCache`, what is the `URLCache`

Almost every mobile app now-a-days rely on networking, either for JSON, image or advertisement. It's rare to find an app in the AppStore that doesn't manage data. To make the app fast and responsive sometimes app developer uses caching mechanism. Though caching is not just about improving UX by making the interface fast and responsive, it's about efficiently managing data that will be used frequently. Not to say, it will effectively strip down repetitive tasks.

There are a lot of great third-party caching libraries available. [Haneke](https://github.com/Haneke/HanekeSwift), [PINCache](https://github.com/pinterest/PINCache), [Cache](https://github.com/hyperoslo/Cache), [Awesome Cache](https://github.com/aschuch/AwesomeCache) are some of the popular third-parties out there. Even image libraries, like [SDWebImage](https://github.com/SDWebImage/SDWebImage), [Kingfisher](https://github.com/onevcat/Kingfisher) or [Nuke](https://github.com/kean/Nuke) provides great caching feature.

But sometimes using a library might be an overkill. Sometimes you might need a customized caching system or maybe you don't like or trust third parties at all.

Apple's `Foundation` framework provides two caching API out of the box. `NSCache` and `URLCache`. `NSCache` is an in-memory mutable collection to temporarily store key-value pairs. It's just an `NSMutableDictionary` with auto-eviction policies in order to free up space in memory as needed, as well as more thread-safety and explicit copying, `NSCopying` protocol, confirmation. 

Let's implement a simple custom UIImageView class that loads an image asynchronously and cache the image using `NSCache` so that it won't perform the next network call for the same image.

```Swift
class UIAsyncImageView: UIImageView {
    static let imageCache = NSCache<AnyObject, AnyObject>()
    
    // Associated Object property; implementation omitted.
    final private var imageUrl: URL?

    func loadImage(from urlString: String, placeholder: UIImage? = nil) {
        setImage(placeholder)

        guard let url = URL(string: urlString) else {
            return
        }

        imageUrl = url

        if let cachedImage = UIAsyncImageView.imageCache.object(forKey: url as AnyObject) as? UIImage {
            self.setImage(cachedImage)
            return
        }

        APIService().get(url: url) { [weak self] (data, error) in
            guard let _imageUrl = self?.imageUrl, _imageUrl == url else {
                return
            }

            guard let data = data, let image = UIImage(data: data) else {
                return
            }

            self?.setImage(image)
            UIAsyncImageView.imageCache.setObject(image, forKey: url as AnyObject)
        }
    }

    private func setImage(_ image: UIImage?) {
        DispatchQueue.main.async { [weak self] in
            self?.image = image
        }
    }
}

// service class
class APIService: ServiceProtocol {
    func get( url: URL, callback: @escaping (_ data: Data?, _ error: Error?) -> Void ) {
        URLSession.shared.dataTask(with: url) { (data, _, error) in
            callback(data, error)
            }.resume()
    }
}

```

As `NSCache` is an in-memory cache, it will use system's memory and will allocate a proportional size to the size of images, or data in generic term. Until other applications need memory and system forces this app to minimize its memory footprint by removing some of its cached objects. Though, `NSCache`  doesn't guarantee that the eviction process will be in orderly manner. Moreover, the cached objects won't be there in next run. The main advantages of `NSCache` are performance and auto-purging feature for objects with transient data that are expensive to create.

For any network data management we should use `URLCache` rather than `NSCache` for caching any data. Because, `URLCache` is both in-memory and on-disk cache, and it doesn't allocate a chunk of memory for it's data. You can define it's in-memory and on-disk size, which is more flexible. `URLCache` will persist the cached data until the system runs low on disk space.

So, the revised implementation of previous `UIAsyncImageView` class would be as,

```Swift
class UIAsyncImageView: UIImageView {
    // Associated Object property; implementation omitted.
    final private var imageUrl: URL?

    func loadImage(from urlString: String, placeholder: UIImage? = nil) {
        setImage(placeholder)

        guard let url = URL(string: urlString) else {
            return
        }

        imageUrl = url

        APIService().get(url: url) { [weak self] (data, error) in
            guard let _imageUrl = self?.imageUrl, _imageUrl == url else {
                return
            }

            guard let data = data, let image = UIImage(data: data) else {
                return
            }

            self?.setImage(image)
        }
    }

    private func setImage(_ image: UIImage?) {
        DispatchQueue.main.async { [weak self] in
            self?.image = image
        }
    }
}

// service class
class APIService: ServiceProtocol {
    func get( url: URL, callback: @escaping (_ data: Data?, _ error: Error?) -> Void ) {
        let request = URLRequest(url: url, cachePolicy: .returnCacheDataElseLoad, timeoutInterval: 30)
        URLSession.shared.dataTask(with: request) { (data, _, error) in
            callback(data, error)
            }.resume()
    }
}

```

And you can change the cache size and disk cache path in shared `URLCache`.

```Swift
URLCache.shared = {
    let cacheDirectory = (NSSearchPathForDirectoriesInDomains(.cachesDirectory, .userDomainMask, true)[0] as String).appendingFormat("/\(Bundle.main.bundleIdentifier ?? "cache")/" )

    return URLCache(memoryCapacity: /*your_desired_memory_cache_size*/,
                    diskCapacity: /*your_desired_disk_cache_size*/,
                    diskPath: cacheDirectory)
}()
```

I usually put this block of code in an extension file and call it from `AppDelegate`'s `didFinishLaunchingWithOptions launchOptions:` method.

```Swift
extension URLCache {
    static func configSharedCache(directory: String? = Bundle.main.bundleIdentifier, memory: Int = 0, disk: Int = 0) {
        URLCache.shared = {
            let cacheDirectory = (NSSearchPathForDirectoriesInDomains(.cachesDirectory, .userDomainMask, true)[0] as String).appendingFormat("/\(directory ?? "cache")/" )
            return URLCache(memoryCapacity: memory, diskCapacity: disk, diskPath: cacheDirectory)
        }()
    }
}

// AppDelegate.swift
func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {

        let diskCacheSize = 500*1024*1024 // 500MB
        URLCache.configSharedCache(disk: diskCacheSize)

        return true
}
```

For more details, check [`NSCache`](https://developer.apple.com/documentation/foundation/nscache) and [`URLCache`](https://developer.apple.com/documentation/foundation/urlcache) pages from `Foundation` documentation.

That's all for this tutorial. Let me know if you have any corrections or questions for me. Happy Coding!
