## generic & protocol associatedType

对一个有关联类型的protocol进行范型化子类

```swift
//用于模型转换
protocol DataTransformer {
    associatedtype Result
    func transform(data: Data) -> Result
}
//为block定制的实现
class BlockTransformer<T>: DataTransformer {
    typealias Result = T
    var block: (Data) -> T
    func transform(data: Data) -> T {
        return block(data)
    }
    init(block: @escaping (Data) -> T) {
        self.block = block
    }
}

class Processor<T: DataTransformer,M> where T.Result == M {
    var dataTransformer: T!
    var block: ((Data) -> M)? {
        didSet {
            if let block = self.block {
                dataTransformer = (BlockTransformer(block: block) as! T)
            }
        }
    }
}

struct TestModel {
    var a = 1
}
//这里感觉有点啰嗦，写了两遍TestModel
let processor = Processor<BlockTransformer<TestModel>,TestModel>()
//
processor.block = {(data) in
    var model = TestModel()
    model.a = 2
    return model
}
processor.dataTransformer.transform(data: Data()).a

```

