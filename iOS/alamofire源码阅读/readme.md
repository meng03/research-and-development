---
title: alamofire源码阅读
date: 2019-07-09 10:44:42
tags:
- iOS
- 第三方库源码
---

## 错误处理

alamofire的错误处理跟kingfisher是一样的。非常易读和维护，而且很容易借鉴。

`public enum AFError: Error`定义一个enum实现Error协议。按类型，在case中列出错误。

````swift
/// `Session` which issued the `Request` was deinitialized, most likely because its reference went out of scope.
case sessionDeinitialized
/// `Session` was explicitly invalidated, possibly with the `Error` produced by the underlying `URLSession`.
case sessionInvalidated(error: Error?)
/// `Request` was explcitly cancelled.
case explicitlyCancelled
/// `URLConvertible` type failed to create a valid `URL`.
case invalidURL(url: URLConvertible)
/// `ParameterEncoding` threw an error during the encoding process.
case parameterEncodingFailed(reason: ParameterEncodingFailureReason)
/// `ParameterEncoder` threw an error while running the encoder.
case parameterEncoderFailed(reason: ParameterEncoderFailureReason)
/// Multipart form encoding failed.
case multipartEncodingFailed(reason: MultipartEncodingFailureReason)
/// `RequestAdapter` failed threw an error during adaptation.
case requestAdaptationFailed(error: Error)
/// Response validation failed.
case responseValidationFailed(reason: ResponseValidationFailureReason)
/// Response serialization failued.
case responseSerializationFailed(reason: ResponseSerializationFailureReason)
/// `ServerTrustEvaluating` instance threw an error during trust evaluation.
case serverTrustEvaluationFailed(reason: ServerTrustFailureReason)
/// `RequestRetrier` threw an error during the request retry process.
case requestRetryFailed(retryError: Error, originalError: Error)
````

错误原因有很多种的时候，不是简单使用字符串表示，而是用一个reason的enum来表达。如下。

````swift
public enum ParameterEncodingFailureReason {
    case missingURL
    case jsonEncodingFailed(error: Error)
}
````

每个reason会有一个`localizedDescription`

````swift
extension AFError.ParameterEncodingFailureReason {
    var localizedDescription: String {
        switch self {
        case .missingURL:
            return "URL request to encode was missing a URL"
        case .jsonEncodingFailed(let error):
            return "JSON could not be encoded because of error:\n\(error.localizedDescription)"
        }
    }
}
````

有了reason的`localizedDescription`，就很好提供AFError的errorDescription了，如下所示。这个去掉了一些代码，避免太长。

````swift
extension AFError: LocalizedError {
    public var errorDescription: String? {
        switch self {
        case .parameterEncodingFailed(let reason):
            return reason.localizedDescription
        case .parameterEncoderFailed(let reason):
            return reason.localizedDescription
        case .multipartEncodingFailed(let reason):
            return reason.localizedDescription
        }
    }
}
````

这样就大概完成了这个结构，当需要一个错误的时候，创建起来很简单。

````swift
AFError.parameterEncodingFailed(reason: .missingURL)
````

当你拿到这个错误的时候，也可以方便的通过`switch`来处理。需要错误描述的时候，也可以通过Error的`errorDescription`拿到，很方便。



## 属性的原子操作

Swift 中没有类似OC的atomic来支持属性的原子操作。需要的话，都是自己去做。Alamofire通过实现一个Protector容器来实现一种属性的原子操作。

OSSPinLock已经废弃了，苹果提供了一个替代方案，支持iOS10以上，叫做`os_unfair_lock_t`。这里首先用Swift将这个lock包装了一个Swift版本`UnfairLock`。对外提供`lock`和`unlock`，以及`around`操作。

````swift
func around(_ closure: () -> Void) {
    lock(); defer { unlock() }
    return closure()
}
````

around操作比较简单，传入一个block，在block前后加锁。



## 一次request的流程

对外接口统一在Alamofire文件中通过全局方法的方式提供。

````swift
public func request(
    _ url: URLConvertible,
    method: HTTPMethod = .get,
    parameters: Parameters? = nil,
    encoding: ParameterEncoding = URLEncoding.default,
    headers: HTTPHeaders? = nil)
    -> DataRequest
{
    return SessionManager.default.request(
        url,
        method: method,
        parameters: parameters,
        encoding: encoding,
        headers: headers
    )
}
````

这个方法没做任何事，直接交给了SessionManager来处理。

sessionManager拿到这一些参数，进行一些包装，处理成一个request，然后继续传递。

```swift
@discardableResult
open func request(
    _ url: URLConvertible,
    method: HTTPMethod = .get,
    parameters: Parameters? = nil,
    encoding: ParameterEncoding = URLEncoding.default,
    headers: HTTPHeaders? = nil)
    -> DataRequest
{
    var originalRequest: URLRequest?

    do {
        originalRequest = try URLRequest(url: url, method: method, headers: headers)
        let encodedURLRequest = try encoding.encode(originalRequest!, with: parameters)
        return request(encodedURLRequest)
    } catch {
        return request(originalRequest, failedWith: error)
    }
}
```



接下来将request进一步包装成task，然后将原始信息与task信息存入delegate中，然后启动task

````swift
@discardableResult
open func request(_ urlRequest: URLRequestConvertible) -> DataRequest {
    var originalRequest: URLRequest?

    do {
        originalRequest = try urlRequest.asURLRequest()
        let originalTask = DataRequest.Requestable(urlRequest: originalRequest!)

        let task = try originalTask.task(session: session, adapter: adapter, queue: queue)
        let request = DataRequest(session: session, requestTask: .data(originalTask, task))

        delegate[task] = request

        if startRequestsImmediately { request.resume() }

        return request
    } catch {
        return request(originalRequest, failedWith: error)
    }
}
````

