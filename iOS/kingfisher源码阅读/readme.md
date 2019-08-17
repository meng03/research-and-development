---
title: kingfisher源码阅读
date: 2019-07-01 16:05:22
tags:
- iOS
- Swift
- 源码
---

## 模块介绍

### 通用模块

#### ImageSource

Kingfisher中对图片资源的表示有两种，定义在Source枚举中。

````swift
public enum Source {
  case network(Resource)
  case provider(ImageDataProvider)
}
````

`network`类型的需要提供一个url和cacheKey，然后kingfisher会自行下载。provider类型的，是自己提供image数据。下面看下Resource

````swift
public protocol Resource {
    
    /// The key used in cache.
    var cacheKey: String { get }
    
    /// The target image URL.
    var downloadURL: URL { get }
}
````

Resource的协议需要两个属性，下载地址和缓存key。正常情况下我们无需实现这个Resource，因为Kingfisher中的URL通过extension实现了这个协议，我们平常直接用URL就可以了。

````swift
extension URL: Resource {
    public var cacheKey: String { return absoluteString }
    public var downloadURL: URL { return self }
}
````

但是，如果你想定制key的格式，可以使用Resource的另一个实现。

````swift
public struct ImageResource: Resource {
    public init(downloadURL: URL, cacheKey: String? = nil) {
        self.downloadURL = downloadURL
        self.cacheKey = cacheKey ?? downloadURL.absoluteString
    }
    public let cacheKey: String
    public let downloadURL: URL
}
````

ImageDataProvider 提供了三个实现，分别是`LocalFileImageDataProvider`，`Base64ImageDataProvider`，`RawImageDataProvider`。三种方式提供image的data，不过这个没什么使用场景，源码中，也只在测试的地方有用到。



#### KingfisherError

````swift
public enum KingfisherError: Error {
	/// Represents the error reason during networking request phase.
    case requestError(reason: RequestErrorReason)
    /// Represents the error reason during networking response phase.
    case responseError(reason: ResponseErrorReason)
    /// Represents the error reason during Kingfisher caching system.
    case cacheError(reason: CacheErrorReason)
    /// Represents the error reason during image processing phase.
    case processorError(reason: ProcessorErrorReason)
    /// Represents the error reason during image setting in a view related class.
    case imageSettingError(reason: ImageSettingErrorReason) 
}
````

Kingfisher 将 error 按照过程划分，分别是请求，返回，缓存，处理，设置，每个过程一种 error 类型。

###下载

当缓存没有命中的时候，就会启动一个下载。下载主要用到两个类，`ImageDownloader`启动下载，并处理回调，SessionDelegate处理URLSessionDataTask的下载代理，并提供任务管理功能。图片下载过程中，会通过`ImageDownloaderDelegate`通知外部。

此外可以通过ImageDownloadRequestModifier拦截修改图片的url。



### 图片处理

Kingfisher提供丰富的图片处理功能。这些功能通过`ImageProcessor`来提供，如果有需要其他的处理，可以自行实现这个协议来提供，比如webp格式的图片处理。

Kingfisher在demo中列出了目前已经提供了的。

````swift
(DefaultImageProcessor.default, "Default"),
(RoundCornerImageProcessor(cornerRadius: 20), "Round Corner"),
(RoundCornerImageProcessor(cornerRadius: 20, roundingCorners: [.topLeft, .bottomRight]), "Round Corner Partial"),
(BlendImageProcessor(blendMode: .lighten, alpha: 1.0, backgroundColor: .red), "Blend"),
(BlurImageProcessor(blurRadius: 5), "Blur"),
(OverlayImageProcessor(overlay: .red, fraction: 0.5), "Overlay"),
(TintImageProcessor(tint: UIColor.red.withAlphaComponent(0.5)), "Tint"),
(ColorControlsProcessor(brightness: 0.0, contrast: 1.1, saturation: 1.1, inputEV: 1.0), "Vibrancy"),
(BlackWhiteProcessor(), "B&W"),
(CroppingImageProcessor(size: CGSize(width: 100, height: 100)), "Cropping"),
(DownsamplingImageProcessor(size: CGSize(width: 25, height: 25)), "Downsampling"),
(BlurImageProcessor(blurRadius: 5) >> RoundCornerImageProcessor(cornerRadius: 20), "Blur + Round Corner")
````

这些功能主要通过 `ImageDrawing`中的方法来实现。

####圆角

````swift
public func image(withRoundRadius radius: CGFloat,
                  fit size: CGSize,
                  roundingCorners corners: RectCorner = .all,
                  backgroundColor: Color? = nil) -> Image
{
    let rect = CGRect(origin: CGPoint(x: 0, y: 0), size: size)
    return draw(to: size) { _ in
        guard let context = UIGraphicsGetCurrentContext() else {
            assertionFailure("[Kingfisher] Failed to create CG context for image.")
            return
        }
        
        if let backgroundColor = backgroundColor {
            let rectPath = UIBezierPath(rect: rect)
            backgroundColor.setFill()
            rectPath.fill()
        }
        
        let path = UIBezierPath(
            roundedRect: rect,
            byRoundingCorners: corners.uiRectCorner,
            cornerRadii: CGSize(width: radius, height: radius)
        )
        context.addPath(path.cgPath)
        context.clip()
        base.draw(in: rect)
    }
}
````

`draw`方法负责开启上下文和关闭上下文，block中是绘制的主要代码。

#### 模糊

```swift
public func blurred(withRadius radius: CGFloat) -> Image {
    // http://www.w3.org/TR/SVG/filters.html#feGaussianBlurElement
    // let d = floor(s * 3*sqrt(2*pi)/4 + 0.5)
    // if d is odd, use three box-blurs of size 'd', centered on the output pixel.
    let s = Float(max(radius, 2.0))
    // We will do blur on a resized image (*0.5), so the blur radius could be half as well.
    
    // Fix the slow compiling time for Swift 3.
    // See https://github.com/onevcat/Kingfisher/issues/611
    let pi2 = 2 * Float.pi
    let sqrtPi2 = sqrt(pi2)
    var targetRadius = floor(s * 3.0 * sqrtPi2 / 4.0 + 0.5)
    
    if targetRadius.isEven { targetRadius += 1 }

    // Determine necessary iteration count by blur radius.
    let iterations: Int
    if radius < 0.5 {
        iterations = 1
    } else if radius < 1.5 {
        iterations = 2
    } else {
        iterations = 3
    }
    
    let w = Int(size.width)
    let h = Int(size.height)
    let rowBytes = Int(CGFloat(cgImage.bytesPerRow))
    
    func createEffectBuffer(_ context: CGContext) -> vImage_Buffer {
        let data = context.data
        let width = vImagePixelCount(context.width)
        let height = vImagePixelCount(context.height)
        let rowBytes = context.bytesPerRow
        
        return vImage_Buffer(data: data, height: height, width: width, rowBytes: rowBytes)
    }
    
    guard let context = beginContext(size: size, scale: scale, inverting: true) else {
        assertionFailure("[Kingfisher] Failed to create CG context for blurring image.")
        return base
    }
    context.draw(cgImage, in: CGRect(x: 0, y: 0, width: w, height: h))
    endContext()
    
    var inBuffer = createEffectBuffer(context)
    
    guard let outContext = beginContext(size: size, scale: scale, inverting: true) else {
        assertionFailure("[Kingfisher] Failed to create CG context for blurring image.")
        return base
    }
    defer { endContext() }
    var outBuffer = createEffectBuffer(outContext)
    
    for _ in 0 ..< iterations {
        let flag = vImage_Flags(kvImageEdgeExtend)
        vImageBoxConvolve_ARGB8888(
            &inBuffer, &outBuffer, nil, 0, 0, UInt32(targetRadius), UInt32(targetRadius), nil, flag)
        // Next inBuffer should be the outButter of current iteration
        (inBuffer, outBuffer) = (outBuffer, inBuffer)
    }
    
    let result = outContext.makeImage().flatMap {
        Image(cgImage: $0, scale: base.scale, orientation: base.imageOrientation)
    }
    
    return blurredImage
}
```

模糊是通过Accelerate.framework中的vImage实现，使用vImage来做比较自由，但是比较麻烦，需要了解算法细节。

其他的诸如Size，blend都比较简单。

### 缓存

#### 缓存策略

Kingfisher有两个关于缓存策略的枚举，`StorageExpiration`用来表示文件创建成功后的缓存策略。

````swift
public enum StorageExpiration {
    /// The item never expires.
    case never
    /// The item expires after a time duration of given seconds from now.
    case seconds(TimeInterval)
    /// The item expires after a time duration of given days from now.
    case days(Int)
    /// The item expires after a given date.
    case date(Date)
    /// Indicates the item is already expired. Use this to skip cache.
    case expired
}
````

`ExpirationExtending`用来表示再次访问后，如何更新该文件的缓存策略。

````swift
public enum ExpirationExtending {
    /// The item expires after the original time, without extending after access.
    case none
    /// The item expiration extends by the original cache time after each access.
    case cacheTime
    /// The item expiration extends by the provided time after each access.
    case expirationTime(_ expiration: StorageExpiration)
}
````

#### 内存缓存

kingfisher内存缓存`MemoryStorage`定义了一个命名空间，里面有三个对象，`Config`，`StorageObject`和实际用来做处理的`Backend`。

`StorageObject`包括数据value，过期策略，和缓存的key值。

```swift
class StorageObject<T> {
  let value: T
  let expiration: StorageExpiration
  let key: String        
}
```

缓存配置对象

```swift
extension MemoryStorage {
    /// Represents the config used in a `MemoryStorage`.
    public struct Config {
        //缓存容量上限
        public var totalCostLimit: Int
        //缓存数量上限
        public var countLimit: Int = .max
        //过期策略
        public var expiration: StorageExpiration = .seconds(300)
        //清理周期
        public let cleanInterval: TimeInterval
    }
}
```

实际做内存缓存处理的类`Backend`，以`NSCache`为存储源，将需要存储的数据构造成`StorageObject`来存储。提供存储，移除，查询三个功能，在查到缓存的时候，会根据延长缓存策略，更新这个对象的缓存策略。

````swift
func value(forKey key: String, extendingExpiration: ExpirationExtending = .cacheTime) -> T? {
    guard let object = storage.object(forKey: key as NSString) else {
        return nil
    }
    if object.expired {
        return nil
    }
    object.extendExpiration(extendingExpiration)
    return object.value
}
````

此外，会创建一个Timer，每隔一段时间（config中配置的）清理一次NSCache中过期的对象。

对NSCache的操作，都需要加锁，这里使用的NSLock。

#### 磁盘缓存

磁盘缓存的结构跟内存缓存类似，使用DiskStorage枚举构造一个命名空间。内部定义了三个结构，配置信息`Config`，代表磁盘文件的`FileMeta`，以及处理逻辑的`Backend`。

````swift
public struct Config {
    //文件大小上限
    public var sizeLimit: UInt
    //过期策略
    public var expiration: StorageExpiration = .days(7)
    //文件后缀
    public var pathExtension: String? = nil
    //是否使用hash值来表示文件名
    public var usesHashedFileName = true
}
````

````swift
struct FileMeta {
    let url: URL    
    let lastAccessDate: Date?
    let estimatedExpirationDate: Date?
    let isDirectory: Bool
    let fileSize: Int
}
````

磁盘缓存的数据源是磁盘，在查询，删除，存储上，比内存缓存要麻烦一点。

`Backend<T: DataTransformable>`这里T是内部使用的数据类型。`DataTransformable`协议定义了T与Data的变换。需要存储时，要从T类型中拿到Data。下面看下store方法。

````swift
func store(
    value: T,
    forKey key: String,
    expiration: StorageExpiration? = nil) throws
{
    //缓存失效策略
    let expiration = expiration ?? config.expiration
    guard !expiration.isExpired else { return }
    
    //拿到数据Data
    let data: Data
    do {
        data = try value.toData()
    } catch {
        throw KingfisherError.cacheError(reason: .cannotConvertToData(object: value, error: error))
    }
    //根据key构建存储的url
    let fileURL = cacheFileURL(forKey: key)

    //构造文件属性：创建时间，更新时间（其实是过期时间）
    let now = Date()
    let attributes: [FileAttributeKey : Any] = [
        // The last access date.
        .creationDate: now.fileAttributeDate,
        // The estimated expiration date.
        .modificationDate: expiration.estimatedExpirationSinceNow.fileAttributeDate
    ]
    //创建文件
    config.fileManager.createFile(atPath: fileURL.path, contents: data, attributes: attributes)
}
````

存储文件带的属性，将创建时间和过期时间放进去。方便后续的处理。

````swift
func value(forKey key: String, referenceDate: Date, actuallyLoad: Bool) throws -> T? {
    let fileManager = config.fileManager
    let fileURL = cacheFileURL(forKey: key)
    let filePath = fileURL.path
    guard fileManager.fileExists(atPath: filePath) else {
        return nil
    }

    //获取文件的meta信息，主要是创建时间，和过期时间
    let meta: FileMeta
    do {
        let resourceKeys: Set<URLResourceKey> = [.contentModificationDateKey, .creationDateKey]
        meta = try FileMeta(fileURL: fileURL, resourceKeys: resourceKeys)
    } catch {
        throw KingfisherError.cacheError(
            reason: .invalidURLResource(error: error, key: key, url: fileURL))
    }
    //过期则返回nil
    if meta.expired(referenceDate: referenceDate) {
        return nil
    }
    //actuallyLoad为false，表明是为了查询，并不需要加载这个数据
    if !actuallyLoad { return T.empty }

    do {
        //读取数据，修改过期时间
        let data = try Data(contentsOf: fileURL)
        let obj = try T.fromData(data)
        metaChangingQueue.async { meta.extendExpiration(with: fileManager) }
        return obj
    } catch {
        throw KingfisherError.cacheError(reason: .cannotLoadDataFromDisk(url: fileURL, error: error))
    }
}
````

磁盘缓存没有定时清理。App将要进入后台，被杀死和被挂起的时候，会做一次清理。

##### 文件操作

**文件写入**

````swift
let now = Date()
let attributes: [FileAttributeKey : Any] = [
    // The last access date.
    .creationDate: now.fileAttributeDate,
    // The estimated expiration date.
    .modificationDate: expiration.estimatedExpirationSinceNow.fileAttributeDate
]
//创建文件
config.fileManager.createFile(atPath: fileURL.path, contents: data, attributes: attributes)
````

缓存的文件写入是通过`FileMananger`的`open func createFile(atPath path: String, contents data: Data?, attributes attr: [FileAttributeKey : Any]? = nil) -> Bool`方法写入，在创建时间和修改时间属性中填入当前时间和过期时间。后续可以根据过期时间来清理磁盘。

**文件属性的操作**

读取文件属性可以通过`url`的`resourceValues`方法，如下

````swift
let resourceKeys: Set<URLResourceKey> = [.contentModificationDateKey, .creationDateKey]
let meta = try fileURL.resourceValues(forKeys: resourceKeys)
````

文件的属性很多，需要了解更多的，可以查看`URLResourceKey`这个结构体。库中主要用到了

```swift
//修改时间，实际存储的是过期时间
contentModificationDateKey
//创建时间
creationDateKey
//文件大小
fileSizeKey
//是否是目录
isDirectoryKey
```

文件属性的修改则是通过`FileManager`来处理

````swift
let attributes: [FileAttributeKey : Any] = [
                .creationDate: Date().fileAttributeDate,
                .modificationDate: .estimatedExpirationSinceNow.fileAttributeDate
            ]
try? fileManager.setAttributes(attributes, ofItemAtPath: url.path)
````

文件数据的读入，比较简单

````swift
let data = try Data(contentsOf: fileURL)
````

#### ImageCache

`ImageCache`包装了磁盘缓存和内存缓存，对外提供简单的查询，存储，清除等功能。

````swift
store(_ image: Image,
                original: Data? = nil,
                forKey key: String,
                options: KingfisherParsedOptionsInfo,
                toDisk: Bool = true,
                completionHandler: ((CacheStoreResult) -> Void)? = nil)
{
    let identifier = options.processor.identifier
    let callbackQueue = options.callbackQueue
    //创建一个key，用于缓存的键值
    let computedKey = key.computedKey(with: identifier)
    //使用内存缓存存储
    memoryStorage.storeNoThrow(value: image, forKey: computedKey, expiration: options.memoryCacheExpiration)
    
    guard toDisk else {
        //如果不需要存磁盘，这里就算存储成功里，构建callback
        if let completionHandler = completionHandler {
            let result = CacheStoreResult(memoryCacheResult: .success(()), diskCacheResult: .success(()))
            callbackQueue.execute { completionHandler(result) }
        }
        return
    }
    //io专用的队列，异步将data存入磁盘
    ioQueue.async {
        let serializer = options.cacheSerializer
        //将数据序列化
        if let data = serializer.data(with: image, original: original) {
            //调用同步存磁盘的方法，方法的实现是调用磁盘缓存的store方法
            self.syncStoreToDisk(
                data,
                forKey: key,
                processorIdentifier: identifier,
                callbackQueue: callbackQueue,
                expiration: options.diskCacheExpiration,
                completionHandler: completionHandler)
        } else {
            guard let completionHandler = completionHandler else { return }
            
            let diskError = KingfisherError.cacheError(
                reason: .cannotSerializeImage(image: image, original: original, serializer: serializer))
            let result = CacheStoreResult(
                memoryCacheResult: .success(()),
                diskCacheResult: .failure(diskError))
            callbackQueue.execute { completionHandler(result) }
        }
    }
}
````



## 流程介绍

Kingfisher最常用的方式

````swift
let url = URL(string: "https://example.com/image.png")
imageView.kf.setImage(with: url)
````

入口即`setImage`，从这个方法进去看看

````swift
@discardableResult
public func setImage(
    with source: Source?,
    placeholder: Placeholder? = nil,
    options: KingfisherOptionsInfo? = nil,
    progressBlock: DownloadProgressBlock? = nil,
    completionHandler: ((Result<RetrieveImageResult, KingfisherError>) -> Void)? = nil) -> DownloadTask?
{
    var mutatingSelf = self
    //没有source直接返回
    guard let source = source else {
        mutatingSelf.placeholder = placeholder
        mutatingSelf.taskIdentifier = nil
        completionHandler?(.failure(KingfisherError.imageSettingError(reason: .emptySource)))
        return nil
    }
    //处理options
    var options = KingfisherParsedOptionsInfo(KingfisherManager.shared.defaultOptions + (options ?? .empty))
    let noImageOrPlaceholderSet = base.image == nil && self.placeholder == nil
    if !options.keepCurrentImageWhileLoading || noImageOrPlaceholderSet {
        // Always set placeholder while there is no image/placeholder yet.
        mutatingSelf.placeholder = placeholder
    }
    //如果有动画，启动动画
    let maybeIndicator = indicator
    maybeIndicator?.startAnimatingView()
    
    //取自增值作为任务的Identifier
    let issuedIdentifier = Source.Identifier.next()
    mutatingSelf.taskIdentifier = issuedIdentifier

    if base.shouldPreloadAllAnimation() {
        options.preloadAllAnimationData = true
    }

    //进度block
    if let block = progressBlock {
        options.onDataReceived = (options.onDataReceived ?? []) + [ImageLoadingProgressSideEffect(block)]
    }

    if let provider = ImageProgressiveProvider(options, refresh: { image in
        self.base.image = image
    }) {
        options.onDataReceived = (options.onDataReceived ?? []) + [provider]
    }
    
    options.onDataReceived?.forEach {
        $0.onShouldApply = { issuedIdentifier == self.taskIdentifier }
    }
    //通过KingfisherManager来获取图片
    let task = KingfisherManager.shared.retrieveImage(
        with: source,
        options: options,
        completionHandler: { result in
            CallbackQueue.mainCurrentOrAsync.execute {
                maybeIndicator?.stopAnimatingView()
                //如果拿到图片后，任务已经不是当前任务了，就不处理
                guard issuedIdentifier == self.taskIdentifier else {
                    let reason: KingfisherError.ImageSettingErrorReason
                    do {
                        let value = try result.get()
                        reason = .notCurrentSourceTask(result: value, error: nil, source: source)
                    } catch {
                        reason = .notCurrentSourceTask(result: nil, error: error, source: source)
                    }
                    let error = KingfisherError.imageSettingError(reason: reason)
                    completionHandler?(.failure(error))
                    return
                }
                
                mutatingSelf.imageTask = nil
                mutatingSelf.taskIdentifier = nil
                switch result {
                //数据拿到设置image
                case .success(let value):
                    guard self.needsTransition(options: options, cacheType: value.cacheType) else {
                        mutatingSelf.placeholder = nil
                        self.base.image = value.image
                        completionHandler?(result)
                        return
                    }
                    
                    self.makeTransition(image: value.image, transition: options.transition) {
                        completionHandler?(result)
                    }
                //任务失败
                case .failure:
                    if let image = options.onFailureImage {
                        self.base.image = image
                    }
                    completionHandler?(result)
                }
            }
        }
    )
    mutatingSelf.imageTask = task
    return task
}
````

流程大概是：

* UIImageView调用setImage设置图片
* 通过KingfisherManager.shared.retrieveImage获取图片
* KingfisherManager.shared.retrieveImage去查内存缓存，有就返回，没有继续
* KingfisherManager.shared.retrieveImage去查磁盘缓存，有就返回，没有继续
* KingfisherManager.shared.loadAndCacheImage下载并缓存图片
* 查看图片中当前的任务和刚执行完的任务是不是同一个，同一个就设置图片，结束
* 不是同一个，就不处理，结束

UIButton的setImage是类似的逻辑，就不多说了

这里需要特别说明的是在tableView中的处理，因为tableView中，快速上下滑动，cell重用。会导致一个UIImageView多次setImage。上面流程中的回设图片中会对比当前task和完成的task，如果不一致，说明这个是之前的某次setImage开启的下载完成了。这是一个对重用的处理细节。

另外当重新滑到之前的位置，setImage的url是开启的，并且没有下载完成，缓存中找不到。这个时候也不会立刻开始下载，而是去看下载任务队列中是否有该url的任务，找到之后，将这个任务绑定到这个UIImageView上。这也是重用的一个场景。



## 多线程处理

库中会频繁用到网络请求和磁盘读写，为了不阻塞主线程，很显然需要用到多线程处理。使用URLSession，数据的返回默认会从异步执行。而磁盘的读写则需要我们自己处理，库中创建了一个处理`IO`的串行队列来处理。

因为这些异步处理，需要一些手段保证数据的一致性。下面列举一下临界区。

`KingfisherManager`是一个单例，所有`UIImageView`，`UIButton`的`setImage`操作都是通过它来完成。`KingfisherManager`中有一个`ImageCache`和一个`downloader`。`cache`中的内存缓存是一个`NSCache`对象，全局的存取到会用到。

`downloader`中下载会返回一个task，这些task会在sessionDelegate中管理。`private var tasks: [URL: SessionDataTask] = [:]`

`SessionDataTask`中保存了该任务的回调信息





多线程会导致一些同步问题，从网络下载完图片，会在未知线程缓存，接下来缓存时，有可能遇到其他任务正在读取缓存的情况。所以内存缓存的写入时加锁处理的。

库中多用队列来处理任务。流程中可能出现的多线程问题：

* 同个文件的读写

请求的起点一般是从`UIImageView.setImage`开始的，这个方法的调用一般是在主线程中发起。这个方法拿到数据后，`completionHandler`是通过`CallbackQueue.mainCurrentOrAsync`在主线程中执行。

在获取数据，会先查内存缓存，在查磁盘缓存。内存缓存的查询比较快，会在当前线程知节执行。磁盘缓存的查询，会使用一个单独的串行队列完成。

## 补充

#### Delegate<Input, Output>

Kingfisher 中使用Delegate类来包装block，添加`[weak self]`来避免 retain cycle。

````swift
class Delegate<Input, Output> {
    init() {}    
    private var block: ((Input) -> Output?)?    
    func delegate<T: AnyObject>(on target: T, block: ((T, Input) -> Output)?) {
        self.block = { [weak target] input in
            guard let target = target else { return nil }
            return block?(target, input)
        }
    }    
    func call(_ input: Input) -> Output? {
        return block?(input)
    }
}
````

delegate方法，将传入的block包装成自己的block，中间加入weak处理。call方法则是调用这个block。



#### defer的使用

````swift
func addCallback(_ callback: TaskCallback) -> CancelToken {
    lock.lock()
    defer { lock.unlock() }
    callbacksStore[currentToken] = callback
    defer { currentToken += 1 }
    return currentToken
}
````

第二个`defer`调用之后，返回的currentToken，值是+1之前的。**所以deffer的调用时机，其实是返回之后。**

> 从语言设计上来说，`defer` 的目的就是进行资源清理和避免重复的返回前需要执行的代码，而不是用来以取巧地实现某些功能。这样做只会让代码可读性降低。

上边这段话是作者在[关于 Swift defer 的正确使用](<https://onevcat.com/2018/11/defer/>)中说的话，然后还是在kingfisher中用了。一个小趣点。

另外这篇文章中还提到了作用域的问题，**defer这个词，并不是在方法返回后执行的。而是所在的作用域结束后返回的**。比如你在if语句中写defer，那这个defer就是在if之后执行。不是在方法返回之后。文中有具体例子，可以细看。

这里使用了多个defer，之前没见过，查了下，**多个defer的调用会以反序执行**，类似入栈，出栈，网上有个很不错的例子，这里看一下。

````swift
guard let database = openDatabase(...) else { return }
defer { closeDatabase(database) }
guard let connection = openConnection(database) else { return } 
defer { closeConnection(connection) }
guard let result = runQuery(connection, ...) else { return }
````

打开一个数据库，打开一个连接，结束之后，defer反序执行，先关掉连接，在关数据库。很恰当。



#### GCD的使用

库中对GCD的使用，是包装了一个`CallbackQueue`。

````swift
public enum CallbackQueue {
    case mainAsync
    case mainCurrentOrAsync
    case untouch
    case dispatch(DispatchQueue)
    
    public func execute(_ block: @escaping () -> Void) {
        switch self {
        case .mainAsync:
            DispatchQueue.main.async { block() }
        case .mainCurrentOrAsync:
            DispatchQueue.main.safeAsync { block() }
        case .untouch:
            block()
        case .dispatch(let queue):
            queue.async { block() }
        }
    }

    var queue: DispatchQueue {
        switch self {
        case .mainAsync: return .main
        case .mainCurrentOrAsync: return .main
        case .untouch: return OperationQueue.current?.underlyingQueue ?? .main
        case .dispatch(let queue): return queue
        }
    }
}
````

从execute的实现可以了解这几个枚举的含义。有一个是没出现过的，就是safeAsync。

````swift
extension DispatchQueue {
    // This method will dispatch the `block` to self.
    // If `self` is the main queue, and current thread is main thread, the block
    // will be invoked immediately instead of being dispatched.
    func safeAsync(_ block: @escaping ()->()) {
        if self === DispatchQueue.main && Thread.isMainThread {
            block()
        } else {
            async { block() }
        }
    }
}
````

这个safeAsync里会先判断当前是不是主队列和主线程。如果是主线程，主队列，就直接执行block，不然就async执行该block。这个方法只在上面的`mainCurrentOrAsync`由主队列调用，感觉这样写不是特别好，或者名字不是特别恰当。因为这个方法是加载DispatchQueue上的，所以其他队列也是可以调用的，但是非主队列调用，都是有问题的。这个方法的声明的范围太广了。