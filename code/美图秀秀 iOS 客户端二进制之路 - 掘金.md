# 美图秀秀 iOS 客户端二进制之路 - 掘金
[美图秀秀 iOS 客户端二进制之路 - 掘金](https://juejin.cn/post/7175023366783385659#heading-34) 

 一、前言
----

美图秀秀 iOS App 自2011年2月上线，业务一直在高速发展迭代，项目代码也在一直的增长，工程环境复杂，且一直处于多种语言混编状态，包含 OC、Swift、C++、C 等，目前代码量除了底层库已超 250W 行，依赖 pod 库达到了300个，代码编译的速度持续劣化，一次全量编译需要花费很长时间（ M1：10+min，MacBook Pro(16-inch, 2019)：20+min ），每当需要重新编译时，比如：拉取新代码、`pod install/update` 、合并分支（祈祷别有冲突吧）等，再编译、验证，期间有可能会多次遇到编译问题，这种等待的煎熬简直不能忍（多么痛的领悟）！

2021年底随着各业务组件抽离的逐步落地，为了提高项目编译速度，提升多业务线研发效率，二进制化成为了我们项目编译优化的必经之路。

历时7个月，从前期调研、确认方案、开发插件、处理编译问题、反复发现及修改完善插件、项目组件集成、脚手架开发、制定新的开发流程及规范、CI集成、主项目集成、到最后全团队推动使用二进制。终于全链路打通了秀秀 iOS 工程的二进制化。

截止11月底，我们的整体工程的编译速度已经提升了85%左右，目前还有很多遗留在壳工程的代码在持续下沉，预期全部下沉后整体编译时间能够提升90%以上。

二、二进制方案
-------

### 二进制制作方案

在开始开发之前，我们对业内主流的二进制制作方案进行了调研，目前主流的方案有2种：

*   基于`podspec`制作二进制
*   基于`壳工程`制作二进制

#### 基于`podspec`制作二进制

基于`podspec`制作二进制是指，根据`podspec`生成`Podfile`，然后执行`pod install`生成`Xcode`工程，最后用`xcodebuild`构建出最终的二进制产物，此过程跟`pod spec/lib lint`类似

*   优点：
    *   支持单个`Pod`库独立制作
    *   源码和二进制版本号一一对应
    *   不依赖壳工程，只要`podspec`能够`lint`通过就可以编译制作
    *   制作时机比较明确（跟随组件发版节奏）
*   缺点：
    *   需要保证`podspec`能够`lint`通过
    *   当依赖多个`Pod`时，需要在`podspec`中明确版本，当壳工程依赖版本比较复杂时，较难维护

#### 基于`壳工程`制作二进制

`CocoaPods`在安装依赖库的时候，会自动生成对应的`project`和`target`，我们可以在壳工程内，利用`xcodebuild`对各个`Pod`库对应的`target`进行编译构建出最终的二进制产物，只要壳工程能编译通过即可制作成功

*   优点：
    *   不需要保证`podspec`能够`lint`通过
    *   可以一次完成所有库的二进制制作
    *   各`Pod`库版本依赖清晰明确
*   缺点：
    *   制作时机不明确，需要根据实际情况进行调整（目前我们使用的是定时任务触发）
    *   全量制作，耗时较长
    *   会存在部分库制作失败的问题，需要单独排查

#### 制作方案选择

虽然基于`podspec`的方案更加“标准化”，更加符合`CocoaPods`的设计理念，但由于秀秀工程依赖库较多，依赖关系复杂，所以各`Pod`库内的`podspec`基本都没有指定依赖库版本号，导致无法通过`lint`，制作二进制的成功率很低，而`壳工程`是开发者平时开发使用的工程，都能保证编译通过，制作二进制的成功率高很多，所以最终选择了基于`壳工程`制作二进制的方案

### 二进制产物形式

#### `动态库` VS `静态库`

众所周知，iOS平台库的形式有2种：`动态库`和`静态库`，动态库在App启动的时候需要通过`dyld`动态加载，系统动态库由于缓存的原因，加载速度很快，而自定义动态库没有缓存，如果数量过多，会导致启动速度变慢，而静态库是合并到最终的可执行文件中，对启动速度影响较小，所以这里我们选择`静态库`

#### `.a` VS `framework`

静态库的组织形式有2种，一种是`library`也就是`.a`，还有一种是`framework`。秀秀工程是`OC`和`Swift`混编项目，`Swift`调用`OC`需要`Clang Module`的支持，而`framework`天然支持`Clang Module`，而且`framework`文件组织方式更加规范，再加上苹果不止一次的推荐使用`framework`，所以最终我们选择了`static framework`这种二进制产物形式

### 二进制产物存储方式

通常二进制产物存放方式有2种，目前我们使用的是第二种：

*   组件所在`git`仓库
*   静态文件服务器

相较于`git`仓库，静态文件服务器有如下优势：

*   接口访问，易于扩展和自动化处理
*   源码和二进制分离，依赖二进制时只需要下载二进制包，比`git clone`快
*   不会增加`git`仓库大小，这点也涉及到源码下载速度

### 二进制`podspec`管理方式

关于二进制`podspec`的管理，目前业内主要有3种方式：

*   单私有源单版本：在不更改私有源和组件版本的前提下，通过动态变更源码`podspec`，达到管理二进制`podspec`的目的
*   单私有源双版本：在不更改私有源的前提下，通过变更组件版本（如：版本号加`-binary`），达到管理二进制`podspec`的目的
*   双私有源单版本：在不更改组件版本的前提下，通过变更组件的私有源，达到管理二进制`podspec`的目的

以上这3种方案主要是针对基于`podspec`制作二进制提供的管理方式，通过上面的分析，秀秀工程选择的是**基于壳工程制作二进制**的方案，由于`subspec`的原因，同一个版本的`Pod`库集成进壳工程时的代码可能会不同，这就导致最终制作出的二进制包不同，但是版本号却是一样的，所以基于`壳工程`制作二进制的方案会出现源码版本号和二进制版本号不能一一对应的问题，一个源码版本可能对应多个二进制版本，所以上面3种方案都不太能解决我们的问题，最终我们使用了了`双私有源多版本`的管理方式

*   双私有源：这里的“双”并不是2个，而是表达源码和二进制私有源要分开存放，秀秀项目有好几个团队，每个团队都有自己的源码私有源，所以这里的意思是，多个源码私有源，单个二进制私有源
*   多版本：一个源码版本对应多个二进制版本，如下图所示，`AFNetworking`的`4.0.1`版本就对应多个二进制版本

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a8c020a56f634dc1a3b5ceccd97ac98c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

> 关于版本号的生成规则，下面会有介绍

三、二进制插件开发
---------

根据秀秀团队日常开发情况，提出以下二进制功能需求点：

*   无侵入：对现有业务无影响
*   `pod install/update`无感知
*   基于`壳工程`制作二进制
*   组件级别的`源码 / 二进制`切换能力
*   支持`Clang Module`, `OC/Swift`混编
*   环境配置：可灵活配置`configuration`、上传二进制包地址、下载地址、二进制源等等
*   源码白名单
*   无二进制时自动切换到源码
*   支持断点调试时切换源码
*   接近原生`CocoaPods`的使用体验，完美利用`CocoaPods`缓存能力

为了满足以上需求，参考`cocoapods-bin`和`cocoapods-imy-bin`，我们开发了`cocoapods-meitu-bin`插件，主要功能如下图：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/be29219996b04d66ae68b9eb51fdfbeb~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

### 二进制制作

#### 制作

`cocoapods-meitu-bin`制作二进制比较简单，进入`Podfile`所在目录，执行`pod bin build-all`即可，根据需要添加相应的`option`选项，支持的`option`选项如下：

| 选项 | 含义 |
| --- | --- |
| `--clean` | 全部二进制包制作完成后删除编译临时目录 |
| `--clean-single` | 每制作完一个二进制包就删除该编译临时目录 |
| `--repo-update` | 更新`Podfile`中指定的`repo`仓库 |
| `--full-build` | 是否全量打包 |
| `--skip-simulator` | 跳过模拟器编译 |
| `--configuration=configName` | 在构建每个目标时使用`configName`指定构建配置，如：'Debug'、'Release'等 |

#### 原理

基于壳工程制作二进制主要流程如下：

*   判断配置文件`BinConfig.yaml`是否存在
    *   存在，读取配置信息
    *   不存在，跳过
*   判断是否需要更新`Podfile`中的`spec`仓库
    *   需要，更新
    *   不需要，跳过
*   判断配置文件中是否存在`pre_build`
    *   存在，执行
    *   不存在，跳过
*   执行依赖分析，获取各`Pod`库对应的`target`
*   遍历所有的`pod targets`制作二进制包
*   判断配置文件中是否存在`post_build`
    *   存在，执行
    *   不存在，跳过
*   清理制作二进制包时的临时文件

其中，最关键的步骤是`依赖分析（analyse）`和`编译所有pod targets（build_pod_targets）` ，下面会重点介绍这两步

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2d77b7871c444a498b72ac4014b7924b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

```ruby

read_config

repo_update

pre_build

@analyze_result = analyse

clean_build_pods

results = build_pod_targets

post_build(results)

clean_build_pods if @clean

```

##### 依赖分析（analyse）

因为我们是基于`壳工程`来制作的二进制包，所以需要获取所有`Pod`库对应的`project`和`target`，另外我们还需要根据源码`podspec`去生成二进制`podspec`，所以我们需要依赖分析来帮我们完成这些工作

依赖分析不需要我们自己做，`CocoaPods`已经提供了`Pod::Installer::Analyzer`类来帮我们完成

##### 编译所有pod targets（build\_pod\_targets）

从依赖分析中我们可以获取所有的`pod targets`，然后遍历该数组，对每一个`pod target`执行如下操作：

*   判断是否是`:path`引入，如果是，跳过，进入下一个`target`
*   判断是否是`external source`引入，如果是，跳过，进入下一个`target`
*   判断是否已经是二进制了，如果是，跳过，进入下一个`target`
*   构建二进制产物
*   压缩并上传二进制产物
*   生成二进制`podspec`并上传

流程图如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b961483e073649f882035409e5c1d6d4~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

###### 构建二进制产物

构建二进制产物分为2部分：

*   分别构建模拟器和真机产物
*   按照`framework`目录结构构造最终的二进制产物

第一步比较简单，核心就是通过`xcodebuild`编译各个`pod target`，主要代码如下：

```ruby

def build
    UI.info "编译`#{@pod_target}`".yellow
    dir = result_product_dir
    FileUtils.rm_rf(dir) if File.exist?(dir)
    
    unless @skip_simulator
      result = build_pod_target
      return false unless result
    end
    
    build_pod_target(false)
end


def build_pod_target(simulator = true)
    ... ...
    command = <<-BUILD
xcodebuild GCC_PREPROCESSOR_DEFINITIONS='$(inherited)' \
GCC_WARN_INHIBIT_ALL_WARNINGS=YES \
-sdk #{sdk} \
ARCHS=#{archs} \
CONFIGURATION_TEMP_DIR=#{temp_dir} \
BUILD_ROOT=#{product_dir} \
BUILD_DIR=#{product_dir} \
clean build \
-configuration #{@configuration} \
-target #{@pod_target} \
-project #{project}
    BUILD
    `#{command}`
    ... ...
end

```

第二步需要按照2种情况进行处理：

*   `framework(.framework)`
*   `static library(.a)`

首先解释一下为什么要按照2种情况进行处理？我们在执行`pod install`的时候，`CocoaPods`会按照`Podfile`和`podspec`中的设置，来决定每一个`Pod`库类型，不同的`Pod`库类型决定了通过`xcodebuild`构建出来的产物形式，以秀秀工程为例，既有`static library`，也有`static framework`，还有`dynamic framework`，因为我们是基于`壳工程`制作二进制，所以需要分别处理这2种情况

> `static framework`和`dynamic framework`文件结构相同，可以当成一种情况进行处理

```ruby
target "MTXX" do
    
    use_frameworks! :linkage => :static
    ... ...
end


def pod_names_for_dynamic_framework
    return [
    'OptimizedSQLCipher',
    '...'
    ]
end


def pod_names_for_static_library
    return [
    'CocoaLumberjack',
    '...',
    ]
end

pre_install do |installer|
  installer.pod_targets.each do |pod|
    if pod_names_for_dynamic_framework.include?(pod.name)
      build_type = Pod::BuildType.dynamic_framework
      pod.instance_variable_set(:@build_type, build_type)
    end
    if pod_names_for_static_library.include?(pod.name)
      build_type = Pod::BuildType.static_library
      pod.instance_variable_set(:@build_type, build_type)
    end
  end
end

```

**`framework(.framework)`的处理**，因为我们选择的二进制产物形式是`framework`，所以如果`Pod`库类型是`framework`的话处理起来相对比较简单，主要流程如下：

*   拷贝真机`framework`到目标产物目录
*   判断是否有`Swift`代码
    *   有，拷贝模拟器`swiftmodules`到目标产物`framework`目录
    *   无，跳过
*   合并模拟器真机二进制文件（`lipo`）
*   拷贝资源文件到`resources(自定义)`文件夹
*   拷贝`vendored_frameworks`到`fwks(自定义)`文件夹
*   拷贝`vendored_libraries`到`libs(自定义)`文件夹

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/546451f0f8c948b4be4e5581c44bc4bc~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

最终产物目录结构如下图所示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d5d4225b24e841d199f173e2c21111c8~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

这里有个问题，如何找到`资源文件`、`vendored_frameworks`和`vendored_libraries`呢？如果我们自己去处理也可以实现，但是会有点麻烦，幸运的是，`CocoaPods`在进行依赖分析的时候，已经帮我们分析出了这些文件，核心代码如下：

```ruby

module Pod
    
    class PodTarget < Target
        
        
        
        attr_reader :file_accessors
    end
end


module Pod
  class Sandbox
    class FileAccessor
        
        
        def resources
            paths_for_attribute(:resources, true)
        end
        
        
        
        
        def vendored_frameworks
            paths_for_attribute(:vendored_frameworks, true)
        end
        
        
        
        
        def vendored_libraries
            paths_for_attribute(:vendored_libraries)
        end
    end
  end
end

```

**`.a`的处理**，相较于`framework`形式要稍微复杂一些，主要流程如下：

*   创建`framework`文件夹，以`AFNetworking`为例，我们需要创建`AFNetworking.framework`的文件夹
*   拷贝头文件（`public`和`private`）
*   生成`umbrella header`和`modulemap`，这一步为了支持`Clang Module`
*   编译特殊的资源文件（下面会介绍）
*   判断是否有`Swift`代码
    *   有，拷贝真机和模拟器`swiftmodules`到目标产物`framework`目录
    *   无，跳过
*   合并真机模拟器二进制文件（`lipo`）
*   拷贝资源文件到`resources(自定义)`文件夹
*   拷贝`vendored_frameworks`到`fwks(自定义)`文件夹
*   拷贝`vendored_libraries`到`libs(自定义)`文件夹

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/afc4e89a89a8484b9198934dc5380d79~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

获取`头文件`、`资源文件`、`vendored_frameworks`和`vendored_libraries`的方法跟上面相同，这里不多赘述，主要讲一下`生成 umbrella header 和 modulemap`以及`编译特殊的资源文件`这2步

1.  生成`umbrella header`和`modulemap`

秀秀工程是`Swift`和`OC`混编的项目，`Swift`调用`OC`需要`Clang Module`的支持，所以需要生成`umbrella header`和`modulemap`，如果`Pod`库类型是`framework`，只要设置`DEFINES_MODULE`为`YES`，使用`xcodebuild`构建该`target`时，会自动生成`umbrella header`和`modulemap`，而`.a`这种形式的类型则不会自动生成，需要我们自己手动去生成

*   `umbrella header`文件是该`Pod`库对外暴露的头文件的合集，以`AFNetworking`为例，内容如下：

```c
#ifdef __OBJC__
#import <UIKit/UIKit.h>
#else
#ifndef FOUNDATION_EXPORT
#if defined(__cplusplus)
#define FOUNDATION_EXPORT extern "C"
#else
#define FOUNDATION_EXPORT extern
#endif
#endif
#endif

#import "AFNetworking.h"
#import "AFHTTPSessionManager.h"
#import "AFURLSessionManager.h"
#import "AFCompatibilityMacros.h"
#import "AFNetworkReachabilityManager.h"
#import "AFSecurityPolicy.h"
#import "AFURLRequestSerialization.h"
#import "AFURLResponseSerialization.h"
#import "AFAutoPurgingImageCache.h"
#import "AFImageDownloader.h"
#import "AFNetworkActivityIndicatorManager.h"
#import "UIActivityIndicatorView+AFNetworking.h"
#import "UIButton+AFNetworking.h"
#import "UIImage+AFNetworking.h"
#import "UIImageView+AFNetworking.h"
#import "UIKit+AFNetworking.h"
#import "UIProgressView+AFNetworking.h"
#import "UIRefreshControl+AFNetworking.h"
#import "WKWebView+AFNetworking.h"

FOUNDATION_EXPORT double AFNetworkingVersionNumber;
FOUNDATION_EXPORT const unsigned char AFNetworkingVersionString[];

```

我们可以通过上面`file_accessors`提供的`public_headers`方法获取所有公开头文件的路径及头文件名，按照`umbrella header`的文件格式写入即可，`umbrella header`的命名规则是`pod_target_name-umbrella.h`，以`AFNetworking`为例，`umbrella header`为`AFNetworking-umbrella.h`，关键代码如下：

```ruby
umbrella_header = Pod::Generator::UmbrellaHeader.new(@pod_target)

umbrella_header.imports = @file_accessors.flat_map(&:public_headers).compact.uniq.map { |header| header.basename }
result = umbrella_header.generate
File.open(umbrella_header_path, "w+") do |f|
  f.write(result)
end

```

> `Clang Module`对`C++`支持的不好，所以`umbrella header`中包含的头文件不要有`C++`代码，否则会有各种奇怪的编译报错

*   `modulemap`文件格式也是相对固定的，下面以`AFNetworking(OC)`和`SwiftyJSON(Swift)`为例，内容如下：

`AFNetworking`:

```c
framework module AFNetworking {
  umbrella header "AFNetworking-umbrella.h"

  export *
  module * { export * }
}

```

`SwiftyJSON`:

```ruby
framework module SwiftyJSON {
  umbrella header "SwiftyJSON-umbrella.h"

  export *
  module * { export * }
}

module SwiftyJSON.Swift {
  header "SwiftyJSON-Swift.h"
  requires objc
}

```

可以看到，含有`Swift`代码的`modulemap`比没有`Swift`代码的多了下面一部分，这一部分主要是用做`OC`调用`Swift`的，不过格式都是相对固定的，我们只要按照这个格式写入`modulemap`文件即可

2.  编译特殊的资源文件

特殊的资源文件包括如下几种：

*   `storyboard / xib`
*   `xcdatamodeld / xcdatamodel`
*   `xcmappingmodel`

以上几种文件有3点需要注意的：

*   需要编译，编译后的产物后缀分别为：
    *   `storyboard` -\> `storyboardc`
    *   `xib` -\> `nib`
    *   `xcdatamodeld` -\> `momd`
    *   `xcdatamodel` -\> `mom`
    *   `xcmappingmodel` -\> `cdm`
*   编译后的产物必须存放在`framework`根目录下，否则无法找到，这是`framework`自己的查找规则
*   `Pod`编译类型为`framework`的无需手动编译，用`xcodebuild`编译时会自动生成，如果编译类型是`.a`则需要手动编译

那如何编译这几种文件呢？我们可以参考`CocoaPods`的实现方式：当我们执行完`pod install`后，在`Pods/Target Support Files/Pods-xxx`目录下有一个`Pods-xxx-resources.sh`的脚本文件，这个文件是`CocoaPods`自动生成的，专门用来处理资源文件的，里面有关于所有资源文件的处理方式，主要代码如下：

```shell
install_resource()
{
    case $RESOURCE_PATH in
    *.storyboard)
        ibtool --reference-external-strings-file --errors --warnings --notices --minimum-deployment-target ${!DEPLOYMENT_TARGET_SETTING_NAME} --output-format human-readable-text --compile "${TARGET_BUILD_DIR}/${UNLOCALIZED_RESOURCES_FOLDER_PATH}/`basename \"$RESOURCE_PATH\" .storyboard`.storyboardc" "$RESOURCE_PATH" --sdk "${SDKROOT}" ${TARGET_DEVICE_ARGS};;
    *.xib)
        ibtool --reference-external-strings-file --errors --warnings --notices --minimum-deployment-target ${!DEPLOYMENT_TARGET_SETTING_NAME} --output-format human-readable-text --compile "${TARGET_BUILD_DIR}/${UNLOCALIZED_RESOURCES_FOLDER_PATH}/`basename \"$RESOURCE_PATH\" .xib`.nib" "$RESOURCE_PATH" --sdk "${SDKROOT}" ${TARGET_DEVICE_ARGS};;
    *.xcdatamodel)
        xcrun momc "$RESOURCE_PATH" "${TARGET_BUILD_DIR}/${UNLOCALIZED_RESOURCES_FOLDER_PATH}/`basename "$RESOURCE_PATH" .xcdatamodel`.mom";;
    *.xcdatamodeld)
        xcrun momc "$RESOURCE_PATH" "${TARGET_BUILD_DIR}/${UNLOCALIZED_RESOURCES_FOLDER_PATH}/`basename "$RESOURCE_PATH" .xcdatamodeld`.momd";;
    *.xcmappingmodel)
        xcrun mapc "$RESOURCE_PATH" "${TARGET_BUILD_DIR}/${UNLOCALIZED_RESOURCES_FOLDER_PATH}/`basename "$RESOURCE_PATH" .xcmappingmodel`.cdm";;
}

```

通过上面的代码，我们就可以编译这些特殊的资源文件，然后将编译产物拷贝到`framework`根目录。

至此，我们的二进制产物就已经制作完成了

###### 压缩并上传二进制产物

1.  压缩二进制产物

```shell
zip --symlinks -r #{output_library} #{input_library}

```

2.  上传二进制产物

```shell
curl -F \"name=#{@pod_target.product_module_name}\" -F \"version=#{@version || @pod_target.root_spec.version}\" -F \"xcode_version=#{xcode_version}\" -F \"file=@#{zip_file}\" #{upload_url}

```

###### 生成二进制`podspec`并上传

1.  生成二进制`podspec`

二进制`podspec`的生成，主要是对源码`podspec`字段的增删和修改，主要流程如下：

*   获取源码`podspec`并转成`hash`
*   对源码`podspec`字段进行增删和修改
*   处理`subspec`
*   生成二进制`podspec`并转成`json`格式
*   将`json`格式的二进制`podspec`写入文件

主要代码如下：

```ruby

def create_binary_podspec
    UI.info "创建二进制podspec：`#{@pod_target}`".yellow
    spec = @pod_target.root_spec.to_hash
    root_dir = @pod_target.framework_name
    
    spec['version'] = version
    
    spec['source'] = source
    
    spec['source_files'] = "#{root_dir}/Headers/*.h"
    spec['public_header_files'] = "#{root_dir}/Headers/*.h"
    spec['private_header_files'] = "#{root_dir}/PrivateHeaders/*.h"
    
    spec['vendored_libraries'] = "#{root_dir}/libs/*.a"
    spec['vendored_frameworks'] = %W[#{root_dir} #{root_dir}/fwks/*.framework]
    
    resources = %W[#{root_dir}/*.{#{special_resource_exts.join(',')}} #{root_dir}/resources/*]
    spec['resources'] = resources
    
    delete_unused(spec)
    
    handle_subspecs(spec)
    
    bin_spec = Pod::Specification.from_hash(spec)
    bin_spec.description = <<-EOF
     「converted automatically by plugin cocoapods-meitu-bin @美图 - zys」
      #{bin_spec.description}
    EOF
    bin_spec
end


def write_binary_podspec(spec)
    UI.info "写入podspec：`#{@pod_target}`".yellow
    podspec_dir = "#{Pathname.pwd}/build_pods/#{@pod_target}/Products/podspec"
    FileUtils.mkdir(podspec_dir) unless File.exist?(podspec_dir)
    file = "#{podspec_dir}/#{@pod_target.pod_name}.podspec.json"
    FileUtils.rm_rf(file) if File.exist?(file)
    
    File.open(file, "w+") do |f|
      f.write(spec.to_pretty_json)
    end
    file
end

```

二进制`podspec`主要字段如下：

*   `version`：二进制版本号，与源码版本号不一样，下面会有介绍
*   `source`：二进制文件下载地址，格式为`{http: 二进制文件下载URL, type: 'zip'}`
*   `source_files`：公开头文件目录，`Headers/*.h`
*   `public_header_files`：公开头文件目录，`Headers/*.h`
*   `private_header_files`：私有头文件目录，`PrivateHeaders/*.h`
*   `vendored_libraries`：依赖的`library`，`libs/*.a`
*   `vendored_frameworks`：依赖的`framework`，`%W[#{root_dir} #{root_dir}/fwks/*.framework]`
*   `resources`：资源文件，`%W[#{root_dir}/*.{#{special_resource_exts.join(',')}} #{root_dir}/resources/*]`

`subspec`采取递归方式处理，每个`subspec`依赖整个二进制文件

```ruby

def handle_subspecs(spec)
    spec['subspecs'].map do |subspec|
      
      handle_single_subspec(subspec, spec)
      
      recursive_handle_subspecs(subspec['subspecs'], spec)
    end if spec && spec['subspecs']
end


def recursive_handle_subspecs(subspecs, spec)
    subspecs.map do |s|
      
      handle_single_subspec(s, spec)
      
      recursive_handle_subspecs(s['subspecs'], spec)
    end if subspecs
end

```

2.  上传二进制`podspec`

上传二进制`podspec`比较简单，使用`CocoaPods`定义好的`Pod::Command::Repo::Push`类进行上传

```ruby
repo_name = Pod::Config.instance.sources_manager.binary_source.name
argvs = %W[#{repo_name} #{binary_podsepc_json} --skip-import-validation --use-libraries --allow-warnings --verbose]

begin
  push = Pod::Command::Repo::Push.new(CLAide::ARGV.new(argvs))
  push.validate!
  push.run
rescue Pod::StandardError => e
  UI.info "推送podspec：#{@pod_target} 失败，#{e.to_s}".red
end

```

需要注意的是，使用`Pod::Command::Repo::Push`类进行上传，默认会进行`xcodebuild`编译检验，此过程耗时且容易失败，所以`cocoapods-meitu-bin`插件进行了hook，去掉了相应的耗时校验，代码如下：

```ruby
module Pod
  class Validator
    def perform_extensive_analysis(spec)
      return true
    end
  end
end

```

#### 版本号

**基于壳工程打包二进制这种方式，无法实现二进制和源码版本号一一对应，一个源码库可能对应多个二进制版本，这取决于`subspec`的数量以及壳工程如何引用该`Pod`库**，所以二进制版本号不能直接使用源码版本号，需要有所区分

如何进行版本号的区分呢？这里主要考虑如下几点：

*   `subspec`组合方式：不同`subspec`可以组合出不同的二进制
*   `Xcode`版本：不同`Xcode`版本的`Swift`编译器无法兼容
*   `configuration`：需要区分`Debug`、`Release`
*   `dependency`不同

基于上面这些考虑因素，我们决定将这几部分拼接成字符串，然后做`MD5`，取该`MD5`的前6位，再在前面加上`bin`组成版本号后缀，最终的二进制版本号形式为`x.y.z.bin[md5前6位]`

```ruby

def version(pod_name, original_version, specifications, configuration = 'Debug')
    
    if @specs_str_md5_hash[pod_name].nil?
      specs = specifications.map(&:name).select { |spec|
        spec.include?(pod_name) && !spec.include?('/Binary')
      }.sort!
      specs << xcode_version
      specs << (configuration.nil? ? 'Debug' : configuration)
      specs_str = specs.join('')
      specs_str_md5 = Digest::MD5.hexdigest(specs_str)[0,6]
      @specs_str_md5_hash[pod_name] = specs_str_md5
    else
      specs_str_md5 = @specs_str_md5_hash[pod_name]
    end
    "#{original_version}.bin#{specs_str_md5}"
end

```

### `二进制 / 源码`切换

#### 使用

在`Podfile`中添加如下代码：

```ruby

plugin 'cocoapods-meitu-bin'

use_binaries!

set_use_source_pods ['AFNetworking']

```

#### 原理

##### 自定义`DSL`

`Podfile`实际上是一个`ruby`文件，所以自定义`DSL`就是自定义方法，代码如下：

```ruby
module Pod
  class Podfile
    module DSL

      def use_binaries!(flag = true)
        set_internal_hash_value(USE_BINARIES, flag)
      end

      def set_use_source_pods(pods)
        hash_pods_use_source = get_internal_hash_value(USE_SOURCE_PODS) || []
        hash_pods_use_source += Array(pods)
        set_internal_hash_value(USE_SOURCE_PODS, hash_pods_use_source)
      end

    end
  end
end

```

从上面的代码我们可以看到，`use_binaries!`和`set_use_source_pods`是把传递的参数存储到了一个全局的`hash`表中，这个全局的`hash`表叫做`internal_hash`，在`pod install`过程中会从该`hash`表中取出相应的值然后进行使用

##### 无感知切换源码二进制

使用`cocoapods-meitu-bin`插件后，我们在执行`pod install`时就可以做到有二进制时用二进制，没有二进制时用源码，那我们是如何实现这个功能的呢？要解释这个问题，我们先来看一下`pod install`的执行流程，核心代码如下：

```ruby



prepare

resolve_dependencies

download_dependencies

validate_targets
if installation_options.skip_pods_project_generation?
  show_skip_pods_project_generation_message
else
  
  integrate
end

write_lockfiles

perform_post_install_actions

```

其中，`resolve_dependencies`是利用`Molinillo`算法来解析依赖，获取所有依赖库的`podspec`信息，我们不想干预该算法的流程，于是我们就在依赖分析完成后，获取所有解析到的`podspec`信息，根据版本号规则获取二进制版本号，然后在二进制`podspec`源中进行查找，看是否有对应的二进制版本，如果有，替换掉源码版本，如果没有，返回原来的源码版本

### 断点调试时`二进制 / 源码`映射

二进制可以加快编译速度，提高研发同学的开发效率。但如果想查看源码需要将组件添加到白名单，然后重新`pod install`、重新编译运行app，当研发同学只想在调试时查看一下源码，会很不方便，因此我们需要提供一种在调试过程中能够查看源码的能力

为了解决这个问题，我们可以利用`LLDB`提供的源码映射功能，由于`LLDB`是调试器，用于调试代码，所以这种方式只适用于调试过程中，不需要重新`pod install`和编译，在只想调试看一下该库内部逻辑的时候还是很方便的

#### 源码映射原理

1.  下载二进制对应版本的源码
2.  利用`LLDB`提供的命令进行`二进制 / 源码`映射

##### 下载源码

源码下载功能已经集成进了`cocoapods-meitu-bin`插件中，使用方式如下：

> 下面的命令需要在项目Podfile同级目录执行，如果使用MBox，需要在命令前加上mbox

```shell
# 下载源码（不需要指定版本号，会进行依赖分析获取版本号）
[mbox] pod bin source add AFNetworking
# 查看已经下载的源码
[mbox] pod bin source list [Pod_Name]
# 删除已经下载的源码
[mbox] pod bin source delete [Pod_Name] [--all]
# 查看帮助
[mbox] pod bin source --help

```

最终的源码下载到`~/LLDB_Sources`目录下

##### `二进制 / 源码`映射

`二进制 / 源码`映射功能需要使用`LLDB`提供的`settings append target.source-map`命令，为了方便大家使用，我们通过`Python`脚本将该命令进行了封装，并提供了更加简洁的`LLDB`命令：`mtmap`

脚本安装步骤及使用方法可以参见插件文档。

除了提供最基本的`二进制 / 源码`映射功能外，还提供了以下3个`LLDB`命令：

*   `mtlist`：列出已经下载的源码
*   `mtshow`：显示当前所有的映射
*   `mtreset`：将之前所有的映射清空

### 配置文件

为了让`cocoapods-meitu-bin`插件更加灵活，我们提供了2个配置文件：

*   `.bin_dev.yml`
*   `BinConfig.yaml`

#### `.bin_dev.yml`

配置文件`.bin_dev.yml`主要用来配置二进制文件上传/下载地址、二进制文件格式及`podspec`存储地址等信息，我们模仿`Podfile`的生成方式，提供了`pod bin init`命令来初始化该配置文件，该命令采用问答的方式，初始化成功后，该文件被存放在`Podfile`同级目录

该配置文件在二进制制作和下载时需要使用，所以需要加到`git`追踪中

之所以用隐藏文件，是因为该配置文件一旦生成，基本不会修改，即使修改也是一个人修改提交，其他人同步

配置文件`.bin_dev.yml`内容如下：

```yaml
---
configuration_env: dev

binary_repo_url: git@techgit.meitu.com:mtbinaryspecs.git

binary_upload_url: http://xxx/file/upload.json

binary_download_url: https://xiuxiu-xxx.com:443/ios/binary

download_file_type: zip

```

#### `BinConfig.yaml`

配置文件`BinConfig.yaml`主要用来配置制作和使用二进制时一些可选的配置项，该文件可以手动创建并放在`Podfile`同级目录

配置文件`BinConfig.yaml`是一些可选的配置项，每个人可能不同，所以不需要添加到`git`追踪中，如果没有配置需求，可以不配置

该文件主要内容分为2部分：

*   `build_config`：制作二进制时相关的配置
*   `install_config`：使用二进制时相关的配置

`build_config`包括4部分：

*   `pre_build`：制作二进制之前执行的命令
*   `post_build`：制作二进制之后执行的命令
*   `black_list`：黑名单（不制作二进制）
*   `write_list`：白名单（不管是否已经有二进制，都重新制作）

`install_config`包含2部分：

*   `use_binary`：是否开启二进制，与`Podfile`中的`use_binaries!`共同判断，两个值只要有一个是`true`，则开启二进制，否则不开启
*   `black_list`：黑名单，与`Podfile`中的`set_use_source_pods`合并为一个数组，该数组中的库不使用二进制

四、美图秀秀接入二进制实践
-------------

### 开发环境版本管理

使用 [bundle](https://link.juejin.cn/?target=https%3A%2F%2Fbundler.io%2Fman%2Fbundle-install.1.html "https://bundler.io/man/bundle-install.1.html") 管理，通过`Gemfile`文件约束使用的`CocoaPods`版本和`cocoapods-meitu-bin`等插件版本，好处如下：

*   开发环境的一致性，避免使用不同插件版本导致的问题
*   插件版本更新的及时同步
*   `CI`打包机和`GitLab Runner`执行机器可通过读取项目`Gemfile`文件来同步`CocoaPods`和`cocoapods-meitu-bin`版本

使用：只需在`Gemfile`所在目录执行`bundle install/update`即可。

### 开发流程更改

二进制链路实践过程中，除了链路开发外，秀秀的工程管理方式、开发模式、开发环境跟之前会有很大差异，更重要的一件事就是对新的开发流程的推进，我们花了很长的时间来一步一步的推动，从前期的所有仓库规范`tag`，到`mbox`小范围引入使用，再到最后的删掉远端`Pods`文件夹和`Podfile.lock`。客户端研发人员往往对这种变化难以从容面对，尤其是业务工作量比较大时，面对研发流程变化、新工具的学习和适应所带来的成本会倍感压力。一直在强调 “无感知” 正是为了更好的让研发人员完成过渡，但也仅仅是工具层面的无感知，其他流程需要依靠 “工具+文档+在线支持” 的紧密配合来保障整个研发流程的转换。

*   **多仓开发成为常态，因此引入[`mbox`](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2FMBoxPlus%2Fmbox%2Fblob%2Fmain%2FREADME-CN.md "https://github.com/MBoxPlus/mbox/blob/main/README-CN.md")作为开发辅助工具** 由于二进制化开发后，`Pods` 文件夹被从远端删除，那后续的多仓开发必须借由效率工具实现，Mbox便成为我们的首选工具。目前`cocoapods-meitu-bin`插件已对 Mbox 进行了支持。优势简单介绍下如下（具体参考官方[Mbox介绍](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2FMBoxPlus%2Fmbox%2Fblob%2Fmain%2FREADME-CN.md "https://github.com/MBoxPlus/mbox/blob/main/README-CN.md")）：
    
    *   沙盒隔离，方便多仓分支管理
    *   `Git`批量管理：在 Workspace 内的所有仓库，都受 MBox 控制，可以快速查询和修改 Git 状态，依赖库越多效率越高
    *   Workspace 级别的 Git Hooks
    *   Feature 需求模型：不仅仅能够管理所有仓库的分支，还能保留未提交的改动，甚至不同 Feature 下进行仓库的差异化。
    *   Feature 协作：提供 Feature 快速导出与导入，实现多人同步协作
    *   统一的依赖管理：支持 `Bundler` 和 `CocoaPods` 两种依赖管理工具
    *   多容器切换，引入了 Container 概念，允许在同一个 Workspace 下有多个 App，这些 App 可能是同平台，也可能是跨平台的
*   **ignore`Pods`文件夹、 `Podfile.lock`文件**
    
    未使用二进制之前，秀秀是带着`Pods`文件夹和`Podfile.lock`文件提交`Git`的，每次`Podfile`有变更都会产生修改记录，同时每次新增`framework`就会涉及到大文件提交，且在代码合并时`podlock`的冲突是常态，合并效率极低，再加上编译慢的问题会更加放大合并的低效。在使用二进制后会把所有源码组件都制作成`xxxx.framework`，且会自动切换源码和二进制，所以必须忽略`Pods`文件夹和`Podfile.lock`文件，不在`Git`提交，这也切合`mbox`的使用方式，也是`CocoaPods`推荐方式。
    
*   **需要频繁 `pod install/update`**
    
    由于忽略`Pods`文件夹，所以每次拉取代码，切分支、查二进制源码、合并代码等都需要执行`Pod` 操作去更新相关组件。对`pod install/update`的速度就提出了要求，因此针对这一点也做了专项优化，后文会专门介绍。
    
*   **规范组件`tag`流程**
    
    二进制组件的制作，`Podfile`组件依赖需要使用`Release`方式（`pod 'AFNetworking','4.0.1'`）引入才可以制作，所以约定在每次发版打`Appstore`包前需要各业务组件提供`Release`版本，这就涉及到组件打`tag`并推送对应版本的`Podspec`文件到远端`Spec`仓库。
    
    为了规范该流程，提供`cocoapods-tag`一行命令实现`Podspec`文件的版本号修改、校验、打`tag`、推送`Podspec`文件到远端`spec repo`仓库
    
    *   规范打 `tag` 流程
    *   可跳过 `lint`
    *   可上传 `spec`

### 具体接入细节

1.  使用 `Bundler` 作为 `Gem` 沙盒，通过`Gemfile`文件指定`Cocoapods`和`cocoapods-meitu-bin`版本
    
    在项目目录新建`Gemfile`文件，指定`Cocoapods`和`cocoapods-meitu-bin`版本如下：
    
    ```ruby
    source "https://rubygems.org" 
    gem 'cocoapods-meitu-bin', '1.0.0'
    gem 'cocoapods', '1.10.2'
    
    ```
    
    执行 `bundle install`安装依赖环境
    
    ```shell
    $ bundle install
      Fetching gem metadata from https://rubygems.org/.......
      Resolving dependencies...
      Using httpclient 2.8.3
      Using bundler 2.3.16
      ....
      Using cocoapods 1.10.2
      Using cocoapods-meitu-bin 1.0.0
      Bundle complete! 2 Gemfile dependencies, 42 gems now installed.
      Use `bundle info [gemname]` to see where a bundled gem is installed.
    
    ```
    
    执行 `pod plugins installed` 可查看输出的插件版本与`Gemfile`指定的是否一致，一致表示更新成功，否则就需要排查未更新的原因
    
2.  初始化`cocoapods-meitu-bin`环境配置
    
    执行 `pod bin init`来初始化二进制相关配置，各配置项说明如下：
    
    *   `configuration_env` 指定编译环境,暂未使用，后期废弃
    *   `binary_repo_url` 二进制`Podspec`私有源地址
    *   `binary_upload_url` 二进制文件上传地址
    *   `binary_download_url` 二进制文件下载地址，后面会依次传入`Xcode`版本、`configuration`、组件名称与组件版本
    *   `download_file_type` 二进制文件类型
    
    ```shell
    $  pod bin init
    
       设置插件配置信息.
       所有的信息都会保存在 podfile同级目录/.bin_dev.yml 文件中.
       你可以在对应目录下手动添加编辑该文件.
       文件包含的配置信息样式如下：
       ---
       configuration_env: dev
       binary_repo_url: git@github.com:xxx/example-private-spec-bin.git
       binary_upload_url: http://localhost:8080/frameworks
       binary_download_url: http://localhost:8080/frameworks
       download_file_type: zip
       
       编译环境
       可选值：[ dev / debug_iphoneos / release_iphoneos ]
       旧值：dev
        >
       dev
       
       二进制podspec私有源地址
       旧值：git@github.com:xxx/example-private-spec-bin.git
        >
       git@github.com:xxx/example-private-spec-bin.git
       
       二进制文件上传地址
       旧值：http://localhost:8080/frameworks
        >
       http://localhost:8080/frameworks
       
       二进制文件下载地址，后面会依次传入Xcode版本、configuration、组件名称与组件版本
       旧值：http://localhost:8080/frameworks
        >
       http://localhost:8080/frameworks
       
       二进制文件类型
       可选值：[ zip / tgz / tar / tbz / txz / dmg ]
       旧值：zip
        >
       zip
    
       设置完成.
    
    ```
    
3.  `Podfile`使用`cocoapods-meitu-bin`插件和相关配置项说明
    
    ```ruby
    
    
    plugin 'cocoapods-meitu-bin'
    
    
    
    use_binaries!(ENV["MEITU_USE_BINARIES"] != "false")
    
    
    set_configuration(ENV["MEITU_USE_CONFIGURATION"].nil? ? 'Debug' : ENV["MEITU_USE_CONFIGURATION"])
    
    
    set_use_source_pods ['SQLiteRepair', 'OptimizedSQLCipher']
    
    platform :ios, '11.0'
    
    source 'https://github.com/CocoaPods/Specs.git'
    
    '''
    
    
    ```
    
    经过上面三步，通过配置`use_binaries!`方法的入参，再执行 `pod install/update`，即可 “无感知” 实现二进制与源码切换
    
    例如`use_binaries!(true)`时，即为从源码组件切换到二进制组件
    
    ```shell
    $  pod install
    当前configuration: `Debug`
    更新私有源仓库 meitu-mtbinaryspecs
    Analyzing dependencies
    Molinillo resolve耗时:4.5s
    Downloading dependencies
    Installing AFNetworking 4.0.1.bine6dc04 (was 4.0.1 and source changed to `git@techgit.xxx.com/binaryspecs.git` from `https://github.com/CocoaPods/Specs.git`)
    Installing Alamofire 5.4.4.binfe6ccf (was 5.4.4 and source changed to `git@techgit.xxx.com/binaryspecs.git` from `https://github.com/CocoaPods/Specs.git`)
    Installing MTDeviceInfo 1.3.8.binae71e2 (was 1.3.8 and source changed to `git@techgit.xxx.com/binaryspecs.git` from `git@techgit.xxx.com/specs.git`)
    ...
    Generating Pods project
    Pod installation complete! There are 198 dependencies from the Podfile and 295 total pods installed.
    
    pod_time_profiler: pod执行耗时：
    pod_time_profiler: ———————————————————————————————————————————————
    pod_time_profiler: |            Stage             |    Time(s)    |
    pod_time_profiler: ———————————————————————————————————————————————
    pod_time_profiler: |     resolve_dependencies     |     7.873     |
    pod_time_profiler: |    download_dependencies     |     9.472     |
    pod_time_profiler: |       validate_targets       |     0.597     |
    pod_time_profiler: |          integrate           |    15.342     |
    pod_time_profiler: |       write_lockfiles        |     0.336     |
    pod_time_profiler: | perform_post_install_actions |     1.980     |
    pod_time_profiler: ———————————————————————————————————————————————
    
    ```
    

五、二进制接入问题
---------

### 1\. 使用规范问题

#### `Podfile` 组件依赖规范

[官方文档](https://link.juejin.cn/?target=https%3A%2F%2Fguides.cocoapods.org%2Fsyntax%2Fpodfile.html "https://guides.cocoapods.org/syntax/podfile.html") 我们先了解下`Podfile`的几种不同的组件依赖方式有何区别：

*   **`git-branch`** 引用方式: `pod 'A', :git => 'xxx.git', :branch => 'master'`
    
    `pod install/update` 操作会从`git`地址进行`git clone`，命令如下：
    
    ```shell
    # 根据branch获取最新的commit id
    $ /usr/bin/git ls-remote <url> <branch>
    # clone当前仓库
    $ /usr/bin/git clone <url> <target path> --template=
    # 切换到对应的commit id
    $ /usr/bin/git -C <target path> checkout --quiet <commit id>
    
    ```
    
*   **`git-commit`** 引用方式: `pod 'A', :git => 'xxx.git', :commit => 'db61f7e'`
    
    `pod install/update` 操作会从`git`地址进行`git clone`，命令如下：
    
    ```shell
    # clone当前仓库
    $ /usr/bin/git clone <url> <target path> --template=
    # 切换到对应的commit id
    $ /usr/bin/git -C <target path> checkout --quiet <commit id>
    
    ```
    
*   **`git-tag`** 引用方式: `pod 'A', :git => 'xxx.git', :tag => '0.1.0'`
    
    `pod install/update` 操作会从`git`地址进行`git clone`，命令如下：
    
    ```shell
    $ /usr/bin/git clone <url> <target path> --template= --single-branch --depth 1 --branch <tag>
    
    ```
    
*   **`Release`** 引用方式: `pod 'A', '0.1.0'`
    
    `pod install/update` 操作会从`Podfile`中的`source`来进行查找对应仓库版本的`podspec`文件，之后根据`podspec`文件进行`git clone`，命令如下（与`git-tag`方式命令一致）：
    
    ```shell
    $ /usr/bin/git clone <url> <target path> --template= --single-branch --depth 1 --branch <tag>
    
    ```
    
*   **`path`** 引用方式: 直接依赖的本地路径，无下载操作
    

综上可以看出，只有 `tag` 和`Release`这两种方式这样 `Pod` 内部 `clone` 会加上 `-depth 1` 参数，这个参数的官方解释为`--depth <depth>: create a shallow clone of that depth`，即只拉取最近这次的 `commit`，不会全量拉取整个仓库和 `.git` 快照文件了，可有效提高`pod install/update`过程中下载仓库的速度。

采用二进制开发流程时，必须在`Podfile`中对组件依赖使用`Release`方式，即`pod 'AFNetworking','4.0.1'`方式，原因如下：

*   二进制组件制作、识别、使用都需要依赖指定`tag`版本
    
*   二进制版本切换本身是`source`源的切换，需要从`Podfile`中的`source`来进行查找对应仓库版本，使用`Release`方式只需要我们控制仓库查找顺序即可拥有二进制源码切换的能力，且不影响`CocoaPods`的其他功能。
    
*   在`Podfile`指定相关组件依赖版本，可以提高`pod install`依赖分析速度，便于版本依赖回溯追踪
    

#### `Podspec`规范

[官方文档](https://link.juejin.cn/?target=https%3A%2F%2Fguides.cocoapods.org%2Fsyntax%2Fpodspec.html "https://guides.cocoapods.org/syntax/podspec.html")

*   指定`swift`版本，如果不写默认指定`swift 4.0` 版本，目前基本相关swift组件都需要`swift 5.0`及以上版本，参考如下：
    
    ```ruby
     s.swift_versions = ['5.0']
     s.swift_versions = ['5.0', '5.1', '5.2']
     s.swift_version = '5.0'
     s.swift_version = '5.0', '5.1'
    
    ```
    
*   `s.source` 和 `s.version` 规范
    
    ```ruby
    s.version          = 'xxx'
    s.source       = { :git => 'git@techgit.meitu.com:xxx/xxx', :tag => 'xxx' }
    
    ```
    
    在实际应用中，经常会因为提供不规范的组件版本号、源码仓库并未打`tag`就提供该`tag`版本组件、`s.source`未调整成`ssh`方式导致本地或`CI`打包机执行`pod install/update` 报错，需要注意一下几点：
    
    *   `s.version`版本号需要符合`Cocoapods` 版本号命名规范,`Cocoapods`版本命名正则校验如下
        
        ```ruby
        VERSION_PATTERN = '[0-9]+(?>\.[0-9a-zA-Z]+)*(-[0-9A-Za-z-]+(\.[0-9A-Za-z-]+)*)?' 
        ANCHORED_VERSION_PATTERN = /\A\s*(#{VERSION_PATTERN})?\s*\z/ 
        
         begin
          Version.new(basename)
        rescue ArgumentError
          raise Informative, 'An unexpected version directory ' \
           "`#{basename}` was encountered for the " \
           "`#{pod_dir}` Pod in the `#{name}` repository."
        end
        
        ```
        
    *   `s.version` 和 `s.source` 版本号一致且该`tag`实际存在
        
    *   秀秀`CI`打包机权限认证只支持ssh方式，所以`s.source` 需要指定为ssh
        
    *   组件资源管理方式，应使用s.resource_bundles，不使用s.resource
        
        ```ruby
        s.ios.resource_bundle = { 'xxxResourceBundle' => 'xxx/xxx/Resources/*.{xcassets,png,...}' }
        或
        s.resource_bundles = {
        'xxxResourceBundle' => ['xxx/Resources/*.{xcassets,png,...}'],
        'xxxOtherResourcesBundle' => ['xxx/OtherResources/*.{xcassets,png,...}']}
        
        ```
        
    
    用`key-value`可以避免相同名称资源的名称冲突，同时建议`bundle`的名称至少应该包括 `Pod` 库的名称，可以尽量减少同名冲突
    

#### 组件引入规范

由于秀秀项目在`Podfile`中同时使用了`use_modular_headers!` 和 `use_frameworks! :linkage => :static`，所有源码组件都会被开启`Clang Module` 支持， 生成 `modulemap` 文件，应使用如下所示组件引入方式：

*   `@import xxx`
*   `import xxx`
*   `#import <xxx/xxx.h>`

```ruby
platform :ios, '11.0'
    
source 'https://github.com/CocoaPods/Specs.git'
    
inhibit_all_warnings!
use_modular_headers!
    
install! 'cocoapods', :deterministic_uuids => false, :generate_multiple_pod_projects => true
    
target 'MTXX' do
   use_frameworks! :linkage => :static
   pods_mtxx_binary()
   pods_for_debugs()
   
   third_pods_just_in_shell_project()    
end

默认使用static_framework格式打包，特别需求在下面设置 

def pod_names_for_dynamic_framework
    return [
    'WCDBSwift',
    ...
    ]
end
    
def pod_names_for_static_library
    return [
    'CocoaLumberjack',
    ...
    ]
end
    

hook_build_to_dynamic_pod_list = pod_names_for_dynamic_framework()

hook_build_to_library_pod_list = pod_names_for_static_library()
    

pre_install do |installer|
  installer.pod_targets.each do |pod|
    if hook_build_to_dynamic_pod_list.include?(pod.name)
      build_type = Pod::BuildType.dynamic_framework
      pod.instance_variable_set(:@build_type, build_type)
    end
    if hook_build_to_library_pod_list.include?(pod.name)
      build_type = Pod::BuildType.static_library
      pod.instance_variable_set(:@build_type, build_type)
    end
  end
end

```

### 2\. 实际方案接入的常见问题

*   **切换到二进制组件壳工程引用头文件报错问题**
    
    由于秀秀壳工程还有部分代码未下沉和拆分到组件中，同时跟随版本在持续下沉和拆分到对应组件，由于之前都是在壳工程中，头文件引入方式还是 `#import "xxx.h"`，且下沉到组件后使用源码组件也不会报错，当切换为二进制组件后壳工程在以`#import "xxx.h"`就会编译报错，所以需要调整为`#import <xxx/xxx.h>`或者`@import xxx;`
    
*   **混编组件组件内使用`import "xxx-Swift.h"`导致在秀秀使用该组件编译报错问题**
    
    ```go
    /Users/.../Desktop/mtxx_work/mtxx/MTXX/Pods/MeituGemSDK/MeituGemSDK/Classes/Base/UI/View/PageControl/MBCSlidePageControl.m:10:9: fatal error: 'MeituGemSDK-Swift.h' file not found
    #import "MeituGemSDK-Swift.h"
         ^~~~~~~~~~~~~~~~~~~~~
    1 error generated.
    
    ```
    
    由于在`Podfile`中设置了 `use_frameworks! :linkage => :static`，源码组件编译产物都会是`static_framework`，所以在组件内使用`#import "xxx-Swift.h"`就会导致编译报错，两种解决方案如下：
    
    1.  调整组件内引用`#import "xxx-Swift.h"`如下
    
    ```arduino
    #if __has_include(<xxxx/xxx-Swift.h>)
    #import <xxx/xxx-Swift.h>
    #else
    #import "xxx-Swift.h"
    #endif
    
    ```
    
    2.  在`Podfile`通过`pre_install`设置该组件编译产物为`static_library`类型
*   **在支持 `Clang Module`组件使用了不支持`Clang Module`模块或组件，并在前者对外暴露的`.h`中使用`#import <xxx/xxx.h>`引用后者，就会导致编译报错**
    
    报错信息如下：
    
    ![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cdc44044af0c4ff89c5916c34e272c33~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)
    
    可以从上图看到 `oc_Swift_2` 组件通过 `s.dependency 'WechatOpenSDK', '~> 1.8.7'` 和 `tx.vendored_frameworks = 'Sources/TencentOpenAPI.framework'` 依赖组件`WechatOpenSDK` 和 组件内的`TencentOpenAPI.framework`， 且这俩都不支持`Clang Module`，在`test2.h` 使用`#import <xxx/xxx.h>`引入依赖，编译就会报错，两种解决方案如下：
    
    *   手动梳理头文件，把在`test2.h`使用`#import <xxx/xxx.h>`调整到`test2.m`引入依赖即可
        
    *   基于LLVM的配置 `Build Setting — Apple LLVM 8.1 Language Modules — Allow Non-modular Includes In Framework Modules`设置为`YES`，则可以在`Framework`中使用模块外的`Include`，不过这种过于粗暴
        
        在报错组件的`Podspec`添加`s.pod_target_xcconfig`设置
        
        ```ruby
        s.pod_target_xcconfig = { 
        'OTHER_SWIFT_FLAGS' => '-Xcc -Wno-error=non-modular-include-in-framework-module',
        'CLANG_ALLOW_NON_MODULAR_INCLUDES_IN_FRAMEWORK_MODULES' => 'YES' 
        }
        
        ```
        
        或者在`Podfile`中修改
        
        ```ruby
        post_install do |installer|
          installer.pod_target_subprojects.flat_map { |p| p.targets }.each do |target|
            if target.name == 'xxx'
               config.build_settings['CLANG_ALLOW_NON_MODULAR_INCLUDES_IN_FRAMEWORK_MODULES'] = 'YES'
               config.build_settings['OTHER_SWIFT_FLAGS'] = '$(inherited) -D COCOAPODS -suppress-warnings -Xcc -Wno-error=non-modular-include-in-framework-module'
            end
          end
        end
        
        ```
        
        这种具有依赖传递性，一个组件添加该设置项，所有依赖该组件的上层组件也需要添加，同时壳工程也需要添加该设置，只是作为过渡兼容方案，建议使用梳理头文件依赖，把不应该在`.h`中的头文件引入调整到`.m`中
        
*   **因切换不同版本`Xcode`，二进制组件并未切换到该版本对应二进制组件导致的编译报错**
    
    ```vbnet
    error:Module compiled with Swift 5.5.2 cannot be imported by the Swift 5.6 compiler: /Users/xxx/xxx/xxx/xxx..
    
    ```
    
    主要是因为切换`Xcode`版本，使用`Swift`编译器版本不一致导致，`cocoapods-meitu-bin` 针对不同的`Xcode` 版本制作不同的二进制组件，需要在切换`Xcode`版本且`Command Line Tools`中也切换对应版本，再执行 `pod install` 就会去更新相关组件版本
    
    ![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7c043767c3bc4b768fcd3c76e3549083~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)
    
*   **秀秀项目使用的底层组件`Clang Module`支持问题和过渡方案**
    
    秀秀项目很多组件都是`Swift`和`OC`混编的和`Swift`组件，而底层提供的二进制组件大多不支持`Clang Module`，所以就会导致组件的`Swift`代码访问底层组件无法访问的问题，解决方案如下：
    
    *   使用桥接头文件的方式
        
        秀秀使用的过渡方案是在含`Swift`组件中新建个`.h`文件引入底层组件依赖，采用这种桥接的方式支持使用，但这种方式需要在不同组件中重复去新建`.h`文件桥接，同时也会因为这样使用导致一些编译问题不好排查
        
    *   推动底层二进制组件提供同学支持 `Clang Module`
        
        在实际推动中，因为底层组件中多使用`C++`代码，发现如果在对外暴露`.h`包含`C++`代码，在`Swift`组件使用就会编译报错，这也是导致底层同学一直未支持`Clang Module`的原因，针对该问题调整如下：
        
        *   `C++`需要使用`OC`来做中转，同时要将创建的 `.m` 文件后缀改成 `.mm` ，这是告诉 `XCode` 编译该文件时要用到 `C++` 代码，对外暴露`OC`的`.h`文件即可
        *   不需要对外暴露且包含`C++`代码的`.h`,可以通过设置组件的`Podsepc`里 `spec.private_header_files = 'Headers/Private/*.h'` 或者 `spec.project_header_files = 'Headers/Project/*.h'`

### 3.`pod install/update` 速度过慢问题

由于各种历史原因，大部分内部远端库都含 `Pods` 文件夹及各种底层库的大文件，版本累积之后会导致仓库的 `.git` 超级大，导致`pod install/update` 在 `git clone` 非`tag`依赖仓库时会下载整个仓库，实际依赖文件只有 20M 但可能却要下载 7G 的仓库（ 我们甚至还存在 20G 的仓库。。。）。二进制化之前由于主工程直接提交了 `Pods` 文件，每次一个仓库更新并不会影响太大，但在二进制化删除 `Pods` 文件后，频繁的 `pod` 操作会让人直接崩溃，数据量太大了。尤其是在本地没有`pod cache`的前提下，一次`pod`操作是非常耗费时间的。

为了加快`pod`速度，我们在`cocoapods-meitu-bin`中对各阶段耗时进行统计，同时也会提示并输出大于 500M 组件![](https://juejin.cn/post/media/16631461302005/16661668717889.jpg)

针对`pod`速度优化，我们主要从下面几方面进行优化：

*   `Podfile`各组件都明确指定版本，避免`Pod`决策分析过多耗时
*   拆分纯净源码库，规范`Pod`组件使用
*   二进制化
    *   已经是二进制的库，迁移到二进制文件服务器
    *   使用二进制组件库，尽量不使用源码组件
*   对暂时无法拆库的组件进行删除`pods`文件夹，移除历史无用大文件记录等
*   提供并发下载支持，`cocoapods-meitu-bin`提供并发下载支持
*   避免因多`Cocoapods`切换版本导致缓存被清（通过`cocoapods-meitu-bin hook cocoapods` 清理缓存方法实现切换`Cocoapods`不去执行移除`Pod Cache`）
*   `.git` 优化 （由于影响过大，暂未执行）

基于上面优化后，在删除`Podfile.lock`,`Pods`文件夹及`CocoaPods`缓存后，前后对比整体提升 80% ，还有部分组件未优化，仍然后下降空间

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/96f52ed44ca04b42948fc48f5a690b58~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

### 4.M1下ruby环境和ffi依赖导致 pod install 报错

该问题多存在于非`M1`使用迁移助手迁移到`M1 MAC`导致

```shell
正常M1 mac 执行下面命令输出 arm64
$ uname -m
arm64
从非M1迁移的系统执行下面命令输出 
$ uname -m
x86_64

```

而部分相关依赖比如 `Homebrew rvm ruby ffi`等均都提供`arm`版本，且安装判断就是通过`uname -m` 获取系统类型, 因此安装的依赖不是`arm`版本，而是`x86_64`版本，导致出现各种奇奇怪怪的问题，比如 `pod install` 失败

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e64799c286c34e4f9f951aaa5221f0f5~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f3f6b13880c54ae082f48ad0af301427~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

针对这种问题，有两种方式处理：

*   彻底给硬盘格式化并重装系统
*   在用户和群组里新建个用户，再重新安装 `cocoapods-meitu-bin`插件，及相关环境配置。推荐这种方式，不需要借助额外的存储设备就可完成文件迁移

针对`M1 MAC`推荐使用`ruby2.7.2`版本

六、二进制CI集成
---------

### CI制作二进制包

制作二进制包主要通过`CI`提供`Runner`去制作二进制包，制作二进制包主要考虑以下几点：

*   制作时机(通过 `GitLab-CI/CD` 流水线计划设定每天`12`和`24`基于`develop`分支触发制作)
*   制作二进制组件相关配置，比如多`Xcode`版本设置和`Configuration`配置(通过`.gitlab-ci.yml`配置多环境和多版本`Xcode`二进制组件制作流水线任务) ![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f3f44ac6e3cb4b8e875ea6ca1f99ef55~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)
    
*   执行完成后结果通知（企业微信机器人通知）

### CI 打App包相关变动

*   **相比较之前`CI`打`App`包,需要执行`pod install`**
    
    *   `pod install` 慢问题，见上面`Pod`优化
    *   打包机缓存问题：多项目打包会导致缓存空间占用极大，一旦缓存占满就需要清理，会导致清理后 `pod install` 速度极慢
        *   方案1：扩容， 简单粗暴，保证大容量
        *   方案2：共享缓存方案，使多台打包机公用一套 `pod` 缓存，有效提升打包机群的打包缓存命中率，但这个方案是理想方案，实施侧由于各种原因还未能支持。
    *   打包机环境升级导致缓存被清理(`CI`调整使用不同版本`cocoapods`，执行`pod`操作会清`pod cache`，针对这点插件内已做了兼容优化处理)
*   **`bundle` 规范环境 通过在项目新建`Gemfile`文件约束相关使用插件版本，`CI`执行前会读取`Gemfile`相关插件版本约束**
    
    ```bash
    source "https://rubygems.org"
    
    gem 'cocoapods-meitu-bin'，'1.0.0'
    gem 'cocoapods', '1.10.2'
    
    ```
    
*   **基于二进制打包配置**
    
    采用环境变量注入的方式，可以根据打包类型进行配置，灵活支撑各业务场景。
    
    通过`CI`环境变量注入 `MEITU_USE_BINARIES` 结合`Podfile`中`use_binaries!(ENV["MEITU_USE_BINARIES"] != "false")`，来控制`CI`是否在`pod install`使用二进制组件还是源码组件。
    
    本地开发因没有注入该环境变量`pod install`就正常走插件二进制流程，实现开发同学使用无感知，没有使用成本。
    

七、总结
----

### cocoapods-meitu-bin 插件核心能力

*   无侵入：对现有业务无影响。
*   `pod install/update` 无感知：利用双私有源、源码白名单等配置，通过 `hook pod` ，实现对二进制或源码的自动依赖。
*   制作二进制包
    *   基于壳工程：依赖壳工程制作所有依赖库的二进制包，无需各依赖组件自动触发打包，一次性完成所有依赖 `Pods` 版本的制作。同时支持对单个组件基于壳工程依赖打包。
    *   基于 `podspec`: 通过单个 `podspec` 制作二进制包（待处理）
*   支持 `Clang Module`, `Swift`、`OC/Swift`混编。
*   环境配置：可灵活配置`configuration`、上传二进制包地址、下载地址、二进制源等等。
*   自动生成二进制 `podspec`
*   源码白名单
*   CocoaPods 优化：支持并发、优化依赖分析时长、仓库过大预警等。
*   完美利用 CocoaPods 缓存能力。

### 二进制化链路上的其他功能建设

除了插件自身以外，在整个链路的开发过程中，遇到了很多的问题，也为此做了很多的脚手架与优化。简单介绍几个：

#### **cocoapods-tag 插件/GUI工具：** 

一行命令实现打`tag`，提交远端及上传`podspec`至`spec`仓库，命令及GUI均已支持，规范打 `tag` 效率提升 90% 以上。

#### **引入字节 Mbox 的多仓开发管理工具，高效的提升多仓开发效率**

#### **代码仓库治理**

通过迁移二进制仓库，拆库等方式，使组件库与Demo工程独立，从而根本上治理掉历史的大文件导致的`.git`过大问题。整个工程所需下载代码大小优化达 80% 以上。

### 总体效果

1.  编译时长：80%+ ，随着业务的逐步下沉，依然还有很大优化空间。
2.  `pod install/update`速度优化 80% 以上。
3.  CI 打二进制包效率相比源码提升：60% 以上，由于 CI 两种方式打包都有 `pod install` 和 产物的处理环节，所以相比本地编译整体时长主要提升在 `xcodebuild` 部分。
4.  更高效的多仓依赖管理，提升研发流程运转效率，`git`相关操作效率随同步开发仓库数量成倍提升，合并效率更是提升 90% 以上：由于二进制后 ignore 了 `Pods` 与 `Podfile.lock` 文件，不同业务间基本不会有什么冲突，规避了之前 "解决冲突 --\> `pod install` --> 编译 --> 修改 --\> 可能重新 `pod install` --\> 重新编译" 的低效问题。

结束语
---

移动端研发蓬勃发展了十余年，各个大型团队都建立起了一个个内部的独立而成熟工具链体系，且大部分属于定制化工具链路，这种体系是很难做到开源的，即使能够开源也只是其中某一种独立工具。所以对于我们这种项目超大，团队却并不太大的企业来说，更需要结合当前团队的研发流程、人员结构、技术特点，自由选择最适合的工具链组合，将各个工具的优势发挥至极致，有效的连接工具和开发者，博采众长，最大程度的提升研发效能。

基础建设目的是为业务做好支撑，提升效能，需要保持对 “不合理”、“重复劳动” 的敏感性。同时也是一项充满未知、磨炼自身的工作，能够打破自身技术牢笼，应用的技术栈会变得比较模糊，希望大家有机会也能把自身的经验和技术投入的更广阔的未来之中，移动端改变世界！

One More
--------

`cocoapods-meitu-bin`插件已开源，希望能够更方便的提供给 iOS 开发者们更加方便的二进制研发工具。

参考资料
----

*   [cocoapods-meitu-bin](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fmeitu%2Fcocoapods-meitu-bin "https://github.com/meitu/cocoapods-meitu-bin")

> 本文发布自美图秀秀技术团队，文章未经授权禁止任何形式的转载。