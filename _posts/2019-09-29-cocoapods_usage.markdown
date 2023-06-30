---
title: 利用Pod管理静态库
---

#### 背景
常规的SDK开发流程是用Xcode新建工程，写代码，然后执行Build，即可生成Framework。这样的开发流程有下面几个弊端：
1. 一次打包无法生成同时支持arm架构和x86的Framework。需要先选择`Generic iOS Device`生成arm架构的Framework，在选择模拟器，生成x86架构的Framework。然后利用lipo命令将2个Framework合并成一个。
2. 使用不方便。如果Framework还依赖了其他的静态库，那么在提供Framework的时候，还需要同时提供依赖的静态库。因为在构建Framework的时候是不会把依赖的静态库打包进去的。
利用cocoapods上面的这些问题可以得到很好的解决，SDK的开发和使用会变得如丝般顺滑。

#### 开发
首先，来看下如何利用cocoapods建立一个Framework工程。

执行`pod lib create MyFramework`，并根据提示进行选择，完成工程创建。
![pod lib create](/assets/cocoapods_usage/pod_lib_create@2x.png)
创建完成之后，会自动打开工程。下图是工程目录文件结构：
![create_file](/assets/cocoapods_usage/create_file@2x.png)
再来看下在Finder中的文件结构：
![init_dir](/assets/cocoapods_usage/init_dir@2x.png)
Example下是Demo工程，MyFramework下是源码文件，还有个比较重要的文件是MyFramework.podspec。Demo工程执行`pod install`的时候，会根据这个文件里面的配置查找源码、静态库、及其他的配置信息。
特别需要注意的是`s.version`字段，后面打包的时候，**需要保证在远程分支有对应的tag**。
详细说明可以查看官方文档[Podspec Syntax Reference](http://guides.cocoapods.org/syntax/podspec.html)。

介绍完工程目录后，就进入实际的开发了。首先将自动生成的ReplaceMe.m文件删除，并创建自己的类文件MyClass，将MyClass.m改为MyClass.mm（后面引入的静态库需要）。注意创建的时候，类要放在Classes目录下。下面我们使用银联的SDK作为测试依赖的静态库。首先，将银联静态库文件放在如下目录中：
![union_pay](/assets/cocoapods_usage/union_pay@2x.png)

并在MyFramework.podspec文件中添加如下命令：
```ruby
  # 银联云闪付
  s.subspec 'UnionPay' do |sss|
      sss.source_files = 'MyFramework/Classes/ThirdParty/Libraries/UnionPay/**/*.{h}'
      sss.vendored_libraries = 'MyFramework/Classes/ThirdParty/Libraries/UnionPay/**/*.a'
      sss.private_header_files = 'MyFramework/Classes/ThirdParty/Libraries/UnionPay/**/*.h'
  end
```
完整配置如下：
```ruby
Pod::Spec.new do |s|
  s.name             = 'MyFramework'
  s.version          = '0.1.0'
  s.summary          = 'A short description of MyFramework.'

  s.description      = <<-DESC 
TODO: Add long description of the pod here.
                       DESC

  s.homepage         = 'https://github.com/jeelun/MyFramework'
  s.license          = { :type => 'MIT', :file => 'LICENSE' }
  s.author           = { 'xxx' => 'xxx@xxx.com' }
  s.source           = { :git => 'https://github.com/xxx/MyFramework.git', :tag => s.version.to_s }

  s.ios.deployment_target = '8.0'

  
  s.libraries     = 'z', 'c++'

  s.subspec 'Core' do |sss|
      sss.source_files = 'MyFramework/Classes/Sources/**/*'
      sss.public_header_files = 'MyFramework/Classes/Sources/**/*.h'
      
      sss.dependency 'MyFramework/UnionPay'
  end
  
  # 银联云闪付
  s.subspec 'UnionPay' do |sss|
      sss.source_files = 'MyFramework/Classes/ThirdParty/Libraries/UnionPay/**/*.{h}'
      sss.vendored_libraries = 'MyFramework/Classes/ThirdParty/Libraries/UnionPay/**/*.a'
      sss.private_header_files = 'MyFramework/Classes/ThirdParty/Libraries/UnionPay/**/*.h'
  end
  
end
```
进入Example目录，执行`pod update`，更新工程配置。此时，依赖的静态库即被引入了工程中。在MyClass中增加`sayHello`的方法，并调用银联静态库的相关方法：
```objc
#import "MyClass.h"
#import "UPPaymentControl.h"

@implementation MyClass

+(void) sayHello {
    id obj = [UPPaymentControl defaultControl];
    NSLog(@"hello, myClass, obj:%@", obj);
}

@end
```
编译一下，没问题的话，就表示SDK开发完成。下面来看下如何打包SDK。

#### 打包
在打包之前，我们先将代码push到远程分支，并创建tag，tag需要与MyFramework.podspec中的`s.version`保持一致。
完成之后，调用`pod package`打包Framework。
进入Example的上级目录，执行下面的命令：

`pod package MyFramework.podspec --force --verbose --no-mangle --exclude-deps`

如果没有报错的话，会在当前目录下生成产物文件夹:
![pod_package](/assets/cocoapods_usage/pod_package@2x.png)

Framework在ios目录下。打出来的Framework默认是支持arm架构和x86架构的。新建一个文件夹output，将Framework和依赖的银联SDK放在改目录下，并将工程中的LICENSE也拷贝到里面。
新建一个文件 MyFramework-SDK.podspec。将如下内容拷贝到文件中：
```ruby
Pod::Spec.new do |s|
  s.name = "MyFramework"
  s.version = "0.0.1"
  s.summary = "MyFramework"
  s.license = {"type"=>"MIT", "file"=>"LICENSE"}
  s.authors = {"xxx"=>"xxx@xx.com"}
  s.homepage = "https://www.google.com/"
  s.description = "MyFramework"
  s.ios.deployment_target = '8.0'
  s.platform = :ios, '8.0'
  s.source = { :http => 'file:' + __dir__ + '/MyFramework.zip' }
 
  s.libraries     = 'z', 'c++'
 
  s.subspec 'Core' do |ss|
    ss.vendored_frameworks = "MyFramework.framework"
  end
 
   s.subspec 'UnionPay' do |sss|
          sss.source_files = 'UnionPay/**/*.{h}'
          sss.vendored_libraries = 'UnionPay/**/*.a'
          sss.public_header_files = 'UnionPay/**/*.h'
   end
end
```
将LICENSE、Framework和UnionPay压缩为`MyFramework.zip`文件。压缩完成之后，删除这3个文件。到这一步，SDK的制作就大功告成了。
![pod_package](/assets/cocoapods_usage/output@2x.png)
使用的时候，只需要提供 MyFramework-SDK.podspec和zip文件。

打包这个环节可以用脚本实现自动化，将podspec文件和zip包的生成通过一行命令完成。


#### 使用
新建测试工程MyApp。在Podfile中增加如下依赖:
```ruby
pod 'MyFramework', :podspec=>'output/MyFramework-SDK.podspec'
```
执行`pod install`，即完成对SDK的引用。


#### 发布到仓库
利用`pod repo push`命令可以将`MyFramework-SDK.podspec`文件push到远程仓库，相应的，也需要将zip文件放置在远程，并修改`MyFramework-SDK.podspec`中的`s.source`字段。
此时，再使用的时候，就不需要将output目录放置在工程中了，只需要在Podfile中添加依赖，即可使用。可参考支付宝SDK的[podspec](https://github.com/CocoaPods/Specs/blob/master/Specs/7/c/2/AlipaySDK-iOS/15.5.9/AlipaySDK-iOS.podspec.json)。





















