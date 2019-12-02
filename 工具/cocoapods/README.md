### 创建私有库的步骤（官方文档的步骤）
1、创建一个私有的用来存放Spec的git仓库。用到这些库的人需要有这个仓库的访问权限。
2、添加私有仓库到CocoaPods的安装目录
````
pod repo add REPO_NAME SOURCE_URL
````
可以通过下面的命令来校验是否成功
````
$ cd ~/.cocoapods/repos/REPO_NAME
$ pod repo lint .
````
3、创建一个私有库模版
````
$ pod lib create yourLibRepo
````
这里会有很多引导，帮你完成创建，比如选择平台（iOS，MACOS），选择语言（Objective-C,Swift）等。
4、打开`Podspec Metadata`目录里的podspec文件，填写版本，介绍，配置`homepage`，`source`等字段。`version`字段是你的初始版本，添加代码，git仓库与配置的source一致。上传代码，打上tag，注意，这里的tag与你的version一致。
5、校验一下你的库`pod lib lint`，不通过就按照报错信息调整。
6、把你的私有库podspec上传到你的podspec仓库
````
$ pod repo push REPO_NAME SPEC_NAME.podspec
````
至此，私有库就算是完成了。

### 补充
1、一个库可以包含多个子库，可以通过podspec文件的subspec描述。个子库可以有独立的依赖。
````
s.subspec 'DataBase' do |d|
    d.source_files = 'Classes/DataBase/*'
    d.dependency 'RealmSwift', '~> 2.10.1'
  end
````
2、依赖私有库的情况
依赖私有库在subspec文件中不需要额外标注
````
s.subspec 'DataBase' do |d|
      d.source_files = 'Classes/DataBase/*''
      d.dependency 'MyLib' //私有库
  end
````
但是`pod lib lint`需要额外的参数用来校验
```` shell
pod lib lint --sources=git@yourPodSpec.com/PodSpecs.git,https://github.com/CocoaPods/Specs.git
````
注意，要确保你的电脑可以访问这个私有库。
### pod机制
从pod配置过程感觉到很多联系，比如podspec文件中有库源码的完整信息，版本号对应tag。pod引用时，source要添加你的podspec库地址。通过这些蛛丝马迹可以看到，pod没有什么黑魔法，大部分是git的使用。帮助管理第三方库的版本与依赖。
那么`pod install`实际做了什么？
pod有个git仓库，存储了所有第三方库的spec文件，pod去检索Podfile中的库名，从git仓库中检索相关的spec文件。从spec文件中取到库的源码地址，根据version即tag，下载源码。

有时候我们`pod install`时，pod告诉我们库找不到，那是因为我们本地的spec文件不全，需要从pod的git仓库更新。这就是`pod repo updata`做的事情。