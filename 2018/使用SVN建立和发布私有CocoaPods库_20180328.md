---
title: 使用SVN建立和发布私有CocoaPods库
date: 2018-03-28 15:32:42
---

[TOC]


# SVN 项目结构

首先，一般 SVN 项目有如下的结构：

- trunk：项目主干
- branches：开发或者 Bug 分支
- tags：发布的版本

比如说我们已经写好了一个 CocoaPods 的库，它的名字是 xxxKit，那么它应该在看起来是这样的：
![](https://raw.githubusercontent.com/zyphs21/MyBlogArchive/master/2018/Resources/20180328_struct.jpeg)


# 建立 CocoaPods 库

我们在 trunk 的目录下执行 

`pod lib create xxxKit`

按照提示输入后，在该目录下就会利用 CocoaPods 的模板生成了一个项目。我们主要关注`xxxKit.podspec` 这个文件。

修改 `xxxKit.podspec`，比如：   

```
Pod::Spec.new do |s|
  s.name         = "xxxKit"
  s.version      = "0.1.1"
  s.summary      = "xxxKit."
  s.homepage     = "https://github.com/zyphs21/"
  s.author       = { "zyphs21" => "hansenhs21@live.com" }
  s.source       = { :svn => "http://xxxx/xxxKit/", :tag => s.version.to_s }
  s.source_files  = "xxxKit/**/*.{swift}"
end
```

主要注意是指定 source 那里，路径填的是 SVN 仓库的地址，并加上 tag。

`s.source = { :svn => "http://xxxx/xxxKit/", :tag => s.version.to_s }`


# 打 tags 发布一个版本

这里以 `Cornerstone` 这个 Mac 端的 SVN 工具来说明。

1. 去到远程库里进行打 tags，注意只有在远程库操作才能打 tag。
    
    ![](https://raw.githubusercontent.com/zyphs21/MyBlogArchive/master/2018/Resources/20180328_tag.jpeg)
    
2. 选择在 trunk 主干上 `右键` -> `Tag…`，然后输入 tag 标签，比如 v0.1.1

    ![](https://raw.githubusercontent.com/zyphs21/MyBlogArchive/master/2018/Resources/20180328_createtag.png)
    
之后只要有开发到了新的版本了，按照这样先打 tag。
![](https://raw.githubusercontent.com/zyphs21/MyBlogArchive/master/2018/Resources/20180328_tags.png)

    
# 使用私有库
    
去到需要使用该库的项目里，在 Podfile 里指定该版本：

```
pod 'xxxKit', :svn => 'http://xxxx/xxxKit/', :tag => '0.1.1'
```

然后执行 `pod install`

