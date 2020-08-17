---
title: CocoaPods相关笔记
date: '2018-03-22T13:37:22.000Z'
---

# CocoaPods相关笔记\_20180322

\[TOC\]

## Swift3.2 和 Swift4 库共存

```ruby
post_install do |installer|
    installer.pods_project.targets.each do |target|
        if target.name == 'ActiveLabel'
            target.build_configurations.each do |config|
                config.build_settings['SWIFT_VERSION'] = '3.2'
            end
        end
    end
end
```

可以通过 Podfile 设置旧的还没支持 Swift4 的库指定用 Swift3.2 编译。 当然也可以一个个的在 Target 的 Build Settings 中设置 Swift complie language version

## 指定在 Debug 下安装库

```ruby
pod 'GDPerformanceView-Swift', '~> 1.2.0', :configurations => ['Debug']
```

## 指定 Master 版本

```ruby
pod 'ActiveLabel', :git => 'https://github.com/optonaut/ActiveLabel.swift', :branch => 'master'
```

## 建立库，发布 CocoaPod 库

1.如果已经本地已经有了建好的 CocoaTouchFrameWork 的 project，那可以在项目目录下执行：

```text
pod spec create YourFrameworkName
```

如果本地没有建好的project可以执行：

```text
pod lib create YourFrameworkName
```

1. 编辑该目录下自动创建好的`YourFrameworkName.podsepc`文件
2. 发布库：
   1. Github 上创建好库；
   2. clone 到本地，并把之前的代码拷贝到次 clone 的目录；
   3. 更新`.podsepc`文件里对应的 git URL 为你在 Github 的地址  

      ```ruby
      s.source = { :git => "https://github.com/xxx/xxx.git", :tag => "#{s.version}" }
      ```

   4. 把代码 push 上去；
   5. 打标记

      ```text
      git tag 0.1.0
      git push --tags
      ```

   6. 注册 truck

      ```text
      pod trunk register xxx@xxx.com 'xxxxx'
      ```

   7. 上面 truck register 会发一封邮件给你，去邮件里验证。然后就可以

      ```text
      pod trunk push
      ```
3. 管理库：  

    查看自己的信息

   ```text
    pod trunk me
   ```

    添加管理维护者  

   ```text
    pod trunk add-owner [库的名字] [新管理者的邮件]
    pod trunk remove-owner [库名] [邮件]
   ```

    查看库信息  

   ```text
    pod trunk info [库名]
   ```

    小结：

   ```text
    在验证和上传你的podspec文件到trunk之前，需要将你的源码push到Github上，tag一个版本号并发布一个release版本，这样podspec文件中的s.source的值才能是准确的：
    git add -A && git commit -m "Release 1.0.1."  
    git tag '1.0.1'  
    git push --tags  
    git push origin master
    这两条命令是为pod添加版本号并打上tag:
    set the new version to 1.0.1
    set the new tag to 1.0.1
   ```

4. 更新发布：  
   1. 修改podspec文件tag版本号，也就是修改s.version的值，提交修改后的代码和podspec文件到Github仓库，重新发布一个Release版本
   2. 进入项目根目录，校验podspec文件，校验成功后，重新push podspec文件到CocoaPods官方仓库，命令如下：

      ```text
      pod cache clean --all // 清除pod缓存
      pod lib lint xxx.podspec --allow-warnings // 校验
      pod trunk push xxx.podspec --allow-warnings // 提交到CocoaPods官方仓库
      ```

遇到问题：  
pod lib lint 在本地执行是成功的  
但是在 pod trunk push 的时候一直报错：

```text
`source_files` pattern did not match any file.
```

后来把文件路径修改为以.git文件所在级别后，本地lib lint不成功，但是push却成功

```text
"Source/*.{h,m,swift}"修改成：
"HSCycleGalleryView/**/*.{h,m,swift}"
```

==本人预测是pod lib lint 检测的source\_files是针对本地.podspec所在的文件目录；而pod tunk push是检测github上的文件目录==

```text
# 可以详细输出日志信息
pod trunk push xx.podspec --verbose
```

podspec文件指定Swift版本：  
`s.swift_version = '>= 3.2, <= 4.0'`

## SVN 私有库

按照SVN 工作流规范，在项目名称目录下建立三个文件夹

* branches
* tags
* trunk

```text
root
|-- trunk
|-- branches
|   |-- v1.0-dev
|   |-- v1.0-stage
|   |-- v2.1-dev
|   |-- v2.1-stage
|   |-- v2.1.1-stage
|   `-- ...
`-- tags
    |-- v1.0.0
    |-- v1.1.0
    |-- v1.1.1
    |-- v2.0.0
    `-- ...
```

在下面目录下 `pod lib create XXX`

/Users/Hanson/Desktop/ios-svn/ios-app/DingdongExtensionKit/trunk/

当库开发完成后，利用svn工具\(比如cornerstone\)，在右击`runk` 目录建立 `tag` 比如 0.1.1

```ruby
# DingdongExtensionKit.podspec
Pod::Spec.new do |s|
  s.name         = "DingdongExtensionKit"
  s.version      = "0.1.1"
  s.summary      = "DingdongExtensionKit."
  s.homepage     = "https://github.com/zyphs21/"
  s.author       = { "zyphs21" => "hansenhs21@live.com" }
  # 注意这里使用 svn=> 以及指向之前 创建的tag
  s.source       = { :svn => "http://dev.vanyun.cn:8000/svn/vanyun/Dingdong/Projects/ios-app/DingdongExtensionKit/", :tag => s.version.to_s }
  s.source_files  = "DingdongExtensionKit/**/*.{swift}"
end
```

这个库就算创建好了，使用的时候在`Podfile`里

```ruby
# pod 'DingdongUserKit', :path => '/Users/Hanson/Desktop/ios-svn/ios-app/DingdongUserKit/trunk/'
pod 'DingdongExtensionKit', :svn => 'http://dev.vanyun.cn:8000/svn/vanyun/Dingdong/Projects/ios-app/DingdongExtensionKit/', :tag => '0.1.1'
```

