# 「CocoaPods」Podfile文件模板 - 掘金
##### 前言：在iOS项目中，通常会使用到CocoaPods作为一个第三方库的依赖管理工具，可以简化对组件的依赖、更新的过程，本文将介绍在iOS项目中多Target企业级项目的Podfile文件编写格式

### 一、podfile介绍

先简单介绍一下podfile文件，podfile文件主要用来指定工程中依赖了那些组件。主要包含了依赖的组件名、组件版本、组件地址等。添加的一些第三方组件可以通过修改podfile文件，然后进行`pod install`进行引入至项目中；更新已有的第三方库，可以修改podfile文件，然后进行`pod update`进行修改，也可以指定对应的版本号

### 二、podfile文件书写规范

此处先贴代码，在代码大多数地方都进行了标注处理，讲了具体代码的含义和用法

```ruby

source 'https://github.com/CocoaPods/Specs.git'

use_frameworks!
platform :ios, '13.0'
inhibit_all_warnings!

use_modular_headers!


eval IO.read('podext.rb')

@enable_develop = true



def basePod
  
  pod 'SnapKit'
  pod 'JXBanner'

  
  pod 'Alamofire', '5.4.4'

end


def businessPod
  
  pod 'developmentPod1', :git=>'https://github.com/author/developmentPod1.git',:branch=>'feature/0.0.1',:dev=>'..'

  
  pod 'developmentPod2', '0.0.2'
  pod 'developmentPod3', '0.0.3'
  pod 'developmentPod4', '0.0.1'

end


target 'projectName' do
  basePod
  businessPod
end


target 'projectNameDebug' do
  basePod
  businessPod
  pod 'LookinServer'
end


pre_install do |installer|
    
end


post_install do |installer|









end


```

### 三、podfile文件部分代码讲解

##### 1.OC组件库Module化：

现在很多常见常用的第三方组件是用OC进行编写的，但是现在很多新业务、新项目都采用Swift开发，这就会面临一个OC组件库进行Module化，方便Swift代码进行调用使用，所以在podfile文件中添加了`use_modular_headers!`。具体的技术细节讲解可参考网易云音乐编写的[Swift混编Module化实践](https://juejin.cn/post/7207269389474037817?searchId=2023121123483447162A8A3A27F2C813B2 "https://juejin.cn/post/7207269389474037817?searchId=2023121123483447162A8A3A27F2C813B2")

##### 2.组件区分

在podfile文件中分出了两个组件（基础组件和业务组件），基础组件通常为公司已经积累许久的内部基础组件库，业务组件通常为当前业务正在开发和使用的组件库。这样在平时更新组件库的版本时会更为便捷，大多变更的均为业务组件库版本号

```ruby

def basePod
  
  pod 'SnapKit'
  pod 'JXBanner'

  
  pod 'Alamofire', '5.4.4'

end


def businessPod
  
  pod 'developmentPod1', :git=>'https://github.com/author/developmentPod1.git',:branch=>'feature/0.0.1',:dev=>'..'

  
  pod 'developmentPod2', '0.0.2'
  pod 'developmentPod3', '0.0.3'
  pod 'developmentPod4', '0.0.1'

end

```

##### 3.target区分

在podfile文件中分出了正式环境和开发环境两个target，当然在正式开发的时候可以还有更多的环境区分，这边举例仅以两个环境进行举例。在不同的环境拥有着不同的作用和功能，所以有时候需要引入不同的测试库，比如开发环境需要引入LookinServer进行UI查看

##### 4.podext.rb文件

当你运行该podfile文件时，可能会报错找不到podext.rb文件，这个是个ruby文件用于一些组件拉取方法和避免发生拉取报错，如果你需要该文件，可以关注并私信我，或者将备注Debug这两句进行删除即可。

参考资料：[我所理解的CocoaPods](https://juejin.cn/post/6844903618789769230 "https://juejin.cn/post/6844903618789769230")

如果该文章对你有所帮助，可以**点赞、收藏并关注一下**！后续会持续更新更多技术内容