---
title: MMKV
date: 2019-07-26 10:36:25
tags: 
- iOS
- mmap
- 源码

---



## 介绍

MMKV是基于mmap的键值存储库。提供了类似NSUserDefaults的功能。

## MMKV的基础 - MMAP

mmap主要有2种用法，一个是建立匿名映射，可以起到父子进程之间共享内存的作用。另一个是磁盘文件映射进程的虚拟地址空间。MMKV就是用的磁盘文件映射。

mmap的主要的好处在于，减少一次内存拷贝。在我们平时的read/write系统调用中，文件内容的拷贝要多经历内核缓冲区这个阶段，所以比mmap多了一次内存拷贝，mmap只有用户空间的内存拷贝(这个阶段read/write也有）。正是因为减少了从Linux的页缓存到用户空间的缓冲区的这一次拷贝，所以mmap大大提高了性能，mmap也被称为zero-copy技术。

### 使用步骤

* 创建文件或者指定文件
* 打开文件
* 调整文件大小（非必须步骤）
* mmap内存映射
* 拷贝内容到映射区
* 扩容 （看需要）
* munmap结束映射
* 关闭文件

### 创建或者打开文件

没什么可说的，指定路径，创建文件。

### 打开文件

使用open函数，返回文件句柄

````Objective-C
_fd = open([url UTF8String], O_RDWR,S_IRWXU);
if (_fd < 0) {
    NSLog(@"fail to open file:%@",url);
    return;
}
````
### 获取文件大小
````Objective-C
size_t fileSize = 0;
struct stat st = {};
if (fstat(_fd, &st) != -1) {
     fileSize = (size_t) st.st_size;
}
````
### 调整文件大小

如果设置的比文件小，则会截取文件。

````Objective-C
//代表将文件中多大的部分对应到内存。以字节为单位，不足一内存页按一内存页处理
//向上取整，找到pagesize的整倍数
size_t pageSize = getpagesize();
if (fileSize == 0 || fileSize/pageSize != 0) {
    _mmapSize = (fileSize/pageSize + 1) * pageSize;
    if (ftruncate(_fd, _mmapSize) != 0) {
        return;
    }
}else {
    _mmapSize = pageSize;
}
````
### 文件内存映射

````Objective-C
void *start = NULL; //由系统选定地址
off_t offset = 0;//offset为文件映射的偏移量，通常设置为0，代表从文件最前方开始对应，offset必须是分页大小的整数倍。可以简单理解为被映射对象内容的起点。
_ptr = (char *) mmap(start, _mmapSize, PROT_READ | PROT_WRITE, MAP_SHARED, _fd, offset);
if (_ptr == MAP_FAILED) {
    NSLog(@"mmap失败,%s",strerror(errno));
    //EBADF 参数fd 不是有效的文件描述词
    //EACCES 存取权限有误。如果是MAP_PRIVATE 情况下文件必须可读，使用MAP_SHARED则要有PROT_WRITE以及该文件要能写入。
    //EINVAL 参数start、length 或offset有一个不合法。
    //EAGAIN 文件被锁住，或是有太多内存被锁住。
    //ENOMEM 内存不足。
    return;
}
````
函数原型为`void *mmap(void *start,size_t length,int prot,int flags,int fd,off_t offsize);`
参数介绍：
* **start** 传入一个期望的映射起始地址。同常传入null，由系统寻找合适的内存区域，并将地址返回。
* **length** 传入映射的长度
* **port** 映射区域的操作属性，有如下四种类型，这里我们使用读写属性。
````Objective-C
#define	PROT_NONE	0x00	/* [MC2] no permissions */
#define	PROT_READ	0x01	/* [MC2] pages can be read */
#define	PROT_WRITE	0x02	/* [MC2] pages can be written */
#define	PROT_EXEC	0x04	/* [MC2] pages can be executed */
````
* **flag** 会影响映射区域的各种特性，可以看下定义，类型比较多
* **fd** 打开的文件句柄
* **offset** 为文件映射的偏移量，通常设置为0，代表从文件最前方开始对应

### 扩容

需要三个步骤，使用ftruncate扩容文件，munmap结束映射，使用新的大小，重新映射。比如如下方法，是一个添加数据的方法，内存不够会扩容后继续添加

````Objective-C
- (void)appendData: (NSData *)data {
    if ((_offset + data.length) > _mmapSize) {
        off_t newSize = _mmapSize + getpagesize();
        if (ftruncate(_fd, newSize) != 0) {
            NSLog(@"fail to truncate [%zu] to size %lld, %s", _mmapSize, newSize, strerror(errno));
            return;
        }
        if (munmap(_ptr, _mmapSize) != 0) {
            NSLog(@"fail to munmap, %s", strerror(errno));
            return;
        }
        _mmapSize = newSize;
        _ptr = (char *) mmap(NULL, _mmapSize, PROT_READ | PROT_WRITE, MAP_SHARED, _fd, 0);
        if (_ptr == MAP_FAILED) {
            NSLog(@"mmap失败,%s",strerror(errno));
            return;
        }
    }
    memcpy(_ptr + _offset, data.bytes, data.length);
    _offset = _offset + data.length;
}
````

## MMKV 数据处理概要

一个MMKV实例会产生两个文件，内容文件，CRC校验文件。校验文件就是存储了内容文件中内容部分的CRC校验值。内容文件的前四个字节存储内容长度，后面这个长度的数据是实际内容。因为文件扩容是以页的整数倍。所以可能还会有一些空白内容在最后。

MMKV启动会读取文件，根据文件长度，把实际数据读取出来转成字典。其中还会进行CRC校验。

后续的操作，不管是读还是写，都是对这个字典进行操作。写操作之后会设置`m_hasFullWriteBack = NO;`。表示有内容没有写回。之后会在合适的时机调用`[self fullWriteBack]`进行数据写入。

## MMKV set的实现

### set的基本逻辑

字典中存的值都是NSData类型，所以数据在存入字典前，需要进行一下转化，如下是基本逻辑。

````objective-c
- (BOOL)setFloat:(float)value forKey:(NSString *)key {
	if (key.length <= 0) {
		return NO;
	}
    //获取Float类型的长度，4个字节
	size_t size = pbFloatSize(value);
    //创建一个data，用来保存这个float
	NSMutableData *data = [NSMutableData dataWithLength:size];
    //构建MiniCodedOutputData，来做float转data
	MiniCodedOutputData output(data);
    //将float写入data
	output.writeFloat(value);
    //将数据写入词典
	return [self setRawData:data forKey:key];
}
````

MMKV 实现了`MiniCodedOutputData`类，来处理字节信息。下面的类型到Data的转换就是通过此类完成。首先`writeRawByte`方法，提供安字节写入的功能，直接填充一个字节，然后位置后移一位

```c++
void MiniCodedOutputData::writeRawByte(uint8_t value) {
	if (m_position == m_size) {
		NSString *reason = [NSString stringWithFormat:@"position: %d, bufferLength: %u", m_position, (unsigned int) m_size];
		@throw [NSException exceptionWithName:@"OutOfSpace" reason:reason userInfo:nil];
	}
	m_ptr[m_position++] = value;
}
```

不同的数据转成NSData的逻辑不一样，整体逻辑是分割成字节，然后使用`writeRawByte`写入

#### Bool类型

Bool类型只占一个字节，直接写入即可

```c++
void MiniCodedOutputData::writeBool(BOOL value) {
	this->writeRawByte(value ? 1 : 0);
}
```



#### Float类型

32位float类型按照小顶端顺序分成4个字节存入

````objective-c
void MiniCodedOutputData::writeRawLittleEndian32(int32_t value) {
	this->writeRawByte((value) &0xff);
	this->writeRawByte((value >> 8) & 0xff);
	this->writeRawByte((value >> 16) & 0xff);
	this->writeRawByte((value >> 24) & 0xff);
}
````

64位的类似

````objective-c
void MiniCodedOutputData::writeRawLittleEndian64(int64_t value) {
	this->writeRawByte((int32_t)(value) &0xff);
	this->writeRawByte((int32_t)(value >> 8) & 0xff);
	this->writeRawByte((int32_t)(value >> 16) & 0xff);
	this->writeRawByte((int32_t)(value >> 24) & 0xff);
	this->writeRawByte((int32_t)(value >> 32) & 0xff);
	this->writeRawByte((int32_t)(value >> 40) & 0xff);
	this->writeRawByte((int32_t)(value >> 48) & 0xff);
	this->writeRawByte((int32_t)(value >> 56) & 0xff);
}
````

#### Int类型

Int类型的存储逻辑稍有不同

````c++
void MiniCodedOutputData::writeInt64(int64_t value) {
	this->writeRawVarint64(value);
}

void MiniCodedOutputData::writeRawVarint64(int64_t value) {
	while (YES) {
    //对0x7FL的取反，在& 结果为0，意味着只有后七位有值
		if ((value & ~0x7FL) == 0) {
			this->writeRawByte((int32_t) value);
			return;
		} else {
      //(value & 0x7f) | 0x80
      // 先 & 0111 1111 ：取后七位，在 | 10000000 ： 首位填1
			this->writeRawByte(((int32_t) value & 0x7f) | 0x80);
      //写入后，右移7位继续
			value = logicalRightShift64(value, 7);
		}
	}
}
````

64位Int类型是每次存7位，首位填1的方式存储。共需要10个字节。7*9 = 63，最后一个字节只存一位。这里每次只处理7位的原因，会在后续进行说明。

32位的Int类型类似

````c++
void MiniCodedOutputData::writeRawVarint32(int32_t value) {
	while (YES) {
		if ((value & ~0x7f) == 0) {
			this->writeRawByte(value);
			return;
		} else {
			this->writeRawByte((value & 0x7f) | 0x80);
			value = logicalRightShift32(value, 7);
		}
	}
}
````



#### NSData类型

data类型的写入，需要记录data的长度，以及值

````c++
void MiniCodedOutputData::writeData(NSData *value) {
  //写入data的长度
	this->writeRawVarint32((int32_t) value.length);
  //写入data本身
	this->writeRawData(value);
}
````

长度的写入已经介绍过了，下面看下Data类型的写入。

```c++
void MiniCodedOutputData::writeRawData(NSData *data) {
	this->writeRawData(data, 0, (int32_t) data.length);
}

void MiniCodedOutputData::writeRawData(NSData *value, int32_t offset, int32_t length) {
	if (length <= 0) {
		return;
	}
	if (m_size - m_position >= length) {
		memcpy(m_ptr + m_position, ((uint8_t *) value.bytes) + offset, length);
		m_position += length;
	} else {
		[NSException exceptionWithName:@"Space" reason:@"too much data than calc" userInfo:nil];
	}
}
```

data类型比较容易处理，直接内存拷贝即可。

#### NSString类型

````c++
void MiniCodedOutputData::writeString(NSString *value) {
	NSUInteger numberOfBytes = [value lengthOfBytesUsingEncoding:NSUTF8StringEncoding];
	this->writeRawVarint32((int32_t) numberOfBytes);
	[value getBytes:m_ptr + m_position
	         maxLength:numberOfBytes
	        usedLength:0
	          encoding:NSUTF8StringEncoding
	           options:0
	             range:NSMakeRange(0, value.length)
	    remainingRange:nullptr];
	m_position += numberOfBytes;
}
````

NSString 类型和Data类似，先写入长度，在转成Data存入。NSString有个方便的方法，直接将数据放到指定的内存位置。这里正好合用。



### 取数据的基本逻辑

取数据的逻辑是对set的反向操作，先从字典中取出数据，然后转成具体类型。

````c++
- (int64_t)getInt64ForKey:(NSString *)key {
	return [self getInt64ForKey:key defaultValue:0];
}
- (int64_t)getInt64ForKey:(NSString *)key defaultValue:(int64_t)defaultValue {
	if (key.length <= 0) {
		return defaultValue;
	}
    //从通过key从字典中取出数据，类型是NSData
	NSData *data = [self getRawDataForKey:key];
	if (data.length > 0) {
		@try {
            //将Data类型转成Int64
			MiniCodedInputData input(data);
			return input.readInt64();
		} @catch (NSException *exception) {
			MMKVError(@"%@", exception);
		}
	}
	return defaultValue;
}
````

MMKV实现了MiniCodedInputData来将NSData转成需要的类型。这个类的操作是MiniCodedOutputData的反向操作。其他都比较类似，这里看一下`readRawVarint64`方法，解释下之前的留下的小问题，为什么要每7位存一个字节。

````c++
int64_t MiniCodedInputData::readRawVarint64() {
	int32_t shift = 0;
	int64_t result = 0;
	while (shift < 64) {
		int8_t b = this->readRawByte();
		result |= (int64_t)(b & 0x7f) << shift;
		if ((b & 0x80) == 0) {
			return result;
		}
		shift += 7;
	}
	@throw [NSException exceptionWithName:@"InvalidProtocolBuffer" reason:@"malformedVarint" userInfo:nil];
	return -1;
}
````

读取的字节是存到一个8位Int中 `int8_t b = this->readRawByte();`。前面的逻辑中每次存入七位，首位填1，因为首位是填充的，没有意义，这里通过 & 0x7f直接移除。如果是8位存储，首位的1会使这个数据变成负数，这里的处理就会变得比较麻烦。

