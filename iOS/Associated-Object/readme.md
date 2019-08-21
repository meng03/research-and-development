#### 使用

关联对象的使用比较简单，下面借用AlamofireImage库中的使用说明一下。

````swift
private struct AssociatedKey {
	static var imageDownloader = "af_UIButton.ImageDownloader"
    static var sharedImageDownloader = "af_UIButton.SharedImageDownloader"
    static var imageReceipts = "af_UIButton.ImageReceipts"
    static var backgroundImageReceipts = "af_UIButton.BackgroundImageReceipts"
 }

public var af_imageDownloader: ImageDownloader? {
	get {
    	return objc_getAssociatedObject(self, &AssociatedKey.imageDownloader) as? ImageDownloader
        }
   	set {
      	objc_setAssociatedObject(self, &AssociatedKey.imageDownloader, newValue, .OBJC_ASSOCIATION_RETAIN_NONATOMIC)
        }
}
````

以上代码摘自UIButton的分类里面。先声明一些key，然后声明一个计算属性，然后在`set`和`get`里面通过`objc_getAssociatedObject`，和`objc_setAssociatedObject`绑定一个属性。还有一种更简洁的实现，Alamofire库作者提到的，使用`selector`来做，省去了声明key的过程。

````objective-c
@implementation NSObject (AssociatedObject)
@dynamic associatedObject;

- (void)setAssociatedObject:(id)object {
     objc_setAssociatedObject(self, @selector(associatedObject), object, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

- (id)associatedObject {
    return objc_getAssociatedObject(self, @selector(associatedObject));
}
````



#### 实现

 `objc_setAssociatedObject`方法是`object-runtime.mm`提供的方法，方法的实现则是调用了`objc-references.mm`里的`_object_set_associative_reference`方法。`objc_getAssociatedObject`方法类似。下面具体说下`_object_set_associative_reference`方法都做了什么事情。

```` c++
void _object_set_associative_reference(id object, void *key, id value, uintptr_t policy) {
    // 声明一个old变量，用来保存之前的value
    ObjcAssociation old_association(0, nil);
    //如果value不是空，就根据plicy（copy，retain）和value生成一个新的ObjcAssociation对象
    id new_value = value ? acquireValue(value, policy) : nil;
    {
    	//创建一个管理器，管理器中有一个map，用来存放系统中所有的关联对象
    	//map的结构是[object:[key:value]]
    	//关于AssociationsManager，下面还会细说，先接着看
        AssociationsManager manager;
        //获取manager中的map
        AssociationsHashMap &associations(manager.associations());
        //对object进行简单的包装，接下来作为map第一层的key
        disguised_ptr_t disguised_object = DISGUISE(object);
        if (new_value) { //如果传入的value非空
            //根据object，搜索此对象所有的关联属性，结构同样为map，存放的关联的属性的key和value
            AssociationsHashMap::iterator i = associations.find(disguised_object);
            if (i != associations.end()) { //找到了此对象的关联属性map
                //取出map(map中的数据结构是pair，pair中的second就是value)
                ObjectAssociationMap *refs = i->second;
                //查看map中该key下是否有值
                ObjectAssociationMap::iterator j = refs->find(key);
                if (j != refs->end()) { //此key下已经有value
                	//将之前的值先存到old变量中
                    old_association = j->second;
                    //将新值和policy包装成ObjcAssociation，存入此key下
                    j->second = ObjcAssociation(policy, new_value);
                } else { //此key下没有新值
                	//在map中添加一个键值对
                    (*refs)[key] = ObjcAssociation(policy, new_value);
                }
            } else { //map中没有此对象，此对象第一次关联对象
                // 在map中插入此对象，并在该对象对应的value中存入key，value
                ObjectAssociationMap *refs = new ObjectAssociationMap;
                associations[disguised_object] = refs;
                (*refs)[key] = ObjcAssociation(policy, new_value);
                //完成后，在object中标记下
                object->setHasAssociatedObjects();
            }
        } else { //值为空
            //从map中查找object对象的关联属性map。
            AssociationsHashMap::iterator i = associations.find(disguised_object);
            if (i !=  associations.end()) {
                ObjectAssociationMap *refs = i->second;
                ObjectAssociationMap::iterator j = refs->find(key);
                if (j != refs->end()) {
                    old_association = j->second;
                    //找到这个pair，就直接移除
                    refs->erase(j);
                }
            }
        }
    }
    // 释放之前的value
    if (old_association.hasValue()) ReleaseValue()(old_association);
}
````

只要知道数据结构，其实大概流程也就清晰了。简单来说，就是查看map中是否有特定的值，有就替换，没有就添加。了解了set方法，`_object_get_associative_reference`方法就很好理解了。就是从map中把值相应的值取出来。这个流程中最重要的结构是`AssociationsManager`，这个结构确实挺有意思。值得看看

```` c++
class AssociationsManager {
    static AssociationsHashMap *_map;
public:
    AssociationsManager()   { AssociationsManagerLock.lock(); }
    ~AssociationsManager()  { AssociationsManagerLock.unlock(); }
    
    AssociationsHashMap &associations() {
        if (_map == NULL)
            _map = new AssociationsHashMap();
        return *_map;
    }
};
````

这个类非常简单，就是包装了一个`static`的`map`。因为是静态类型，所有的对象共享这个属性，所以算是实现了一个有限访问的全局变量。这样写避免了使用全局变量时，被其他对象任意访问造成的数据不可控问题。另外，在构造函数和析构函数里分别加上了`lock`和`unlock`。保证了map是线程安全的。这是一个很好的最佳实践，值得借鉴到我们自己的代码中。

#### 生命周期

从上述实现中可以看到，当对某个`object`的某个`key`，做`set nil`的操作后，相应的`value`，以及所在的`ObjcAssociation`就被移除并清理了，那如果不`set nil` 操作,这个`value`，以及`object`相应的全部关联对象什么时候销毁呢？

```` objective-c
void *objc_destructInstance(id obj) 
{
    if (obj) {
        // Read all of the flags at once for performance.
        bool cxx = obj->hasCxxDtor();
        bool assoc = obj->hasAssociatedObjects();

        // This order is important.
        if (cxx) object_cxxDestruct(obj);
        //当有关联对象时，移除该
        if (assoc) _object_remove_assocations(obj);
        obj->clearDeallocating();
    }

    return obj;
}
````

上面代码是在`runtime.h`中声明的方法，`objc_destructInstance`方法的调用时机是`NSObject`的`dealloc`方法。也就是说，当对象被释放时，会移除这个对象绑定的所有属性。