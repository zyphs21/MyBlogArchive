---
title: 用SwiftGen管理UIImage等的String-based接口
date: 2017-12-08 15:15:21
---

# 问题现状
平时我们使用UIImage，UIFont，UIColor会遇到很多String-based的接口方法，比如常用的UIImage：
```Swift
let testImage = UIImage(named: "test")
```
对于上面的代码，如果我们把 `test` 拼写错了，Xcode 并不会给出提示，只有当我们运行的时候才会收到报错，这样维护起来是有一定成本的。

<!-- more -->

# 代码解决
我们可以用类似如下的代码来解决这个问题：
```Swift
import UIKit.UIImage

struct ImageAssets {
    fileprivate var name: String
    var image: UIImage {
        let image = UIImage(named: name)
        guard let result = image else { fatalError("Unable to load image named \(name).") }
        return result
    }
}
enum Assets {
    enum AppLogo {
        static let appLogo = ImageAssets(name: "appLogo")
        static let grayLogo = ImageAssets(name: "gray_logo")
    }
    enum Arrow {
        static let arrowBlue = ImageAssets(name: "arrow_blue")
        static let arrowBrown = ImageAssets(name: "arrow_brown")
    }
    // ....
}
extension UIImage {
    convenience init!(asset: ImageAssets) {
        self.init(named: asset.name)
    }
}
```

利用上面的代码，我们在新建 UIImage 的时候就不需要去想图片的名字了，而且 Xcode 还会有代码提示：
```Swift
let logo = Asset.AppLogo.appLogo.image
let grayLogo = UIImage(asset: Asset.AppLogo.grayLogo)
```

# 存在问题
虽然这种方法在调用的时候简单又安全了，但是项目中的图片往往比较多，如果手动编写维护那段代码也是需要不少精力，而且不能保证后续不会添加新的图片，这样每次都要去维护那段代码不免有些反人类。
那么现在就要介绍这个开源项目---  [`SwiftGen`](https://github.com/SwiftGen/SwiftGen)了！

>SwiftGen is a tool to auto-generate Swift code for resources of your projects, to make them type-safe to use.

利用 SwiftGen 可以帮我们生成这类的代码，但是 SwiftGen 默认生成的代码样式有时候并不是我们想要的，而且默认生成还会有针对 macOS 上的代码，比较好的是 SwiftGen 提供了模板的功能，我们可以按自己的需要来修改模板。

# 集成 SwiftGen 在项目中
SwiftGen 提供了好几种的集成方式，我这里只介绍我自己比较喜欢的方式：就是通过下载它的 Zip 文件解压到项目的目录中，然后通过添加 Run Script 来进行管理。这样可以基本做到不用操心代码。

## 1.修改模板
- 到 SwiftGen仓库的Release页面下载最新的 [swiftgen-5.2.1.zip](https://github.com/SwiftGen/SwiftGen/releases)
- 将解压后的 `swiftgen-5.2.1` 文件夹放到项目所在的目录下(存放`xxx.xcodeproj` 的位置)，可以将文件夹的名字改为`SwiftGen5`简洁一点。
- 进入到 `SwiftGen5` 里的 `templates/xcassets` 目录下，这里面可以看到有不少模板，我们选择 `swift4.stencil` 复制一份，命名为 `my-swift4.stencil` 然后我们就可以在里面修改我们自己想要的模板，我主要是想把 macOS 等其它平台的一些判断代码给删掉:
```Swift
// Generated using SwiftGen, using my-templete created by Hanson

{% if catalogs %}
{% set imageAlias %}{{param.imageAliasName|default:"Image"}}{% endset %}
import UIKit.UIImage

typealias {{imageAlias}} = UIImage

{% set enumName %}{{param.enumName|default:"Asset"}}{% endset %}
{% set imageType %}{{param.imageTypeName|default:"ImageAsset"}}{% endset %}
@available(*, deprecated, renamed: "{{imageType}}")
typealias {{enumName}}Type = {{imageType}}

struct {{imageType}} {
fileprivate var name: String

var image: {{imageAlias}} {
let bundle = Bundle(for: BundleToken.self)
let image = {{imageAlias}}(named: name, in: bundle, compatibleWith: nil)
guard let result = image else { fatalError("Unable to load image named \(name).") }
return result
}
}
{% macro enumBlock assets sp %}
{{sp}}  {% call casesBlock assets sp %}
{{sp}}  {% if not param.noAllValues %}
{{sp}}  {% endif %}
{% endmacro %}
{% macro casesBlock assets sp %}
{{sp}}  {% for asset in assets %}
{{sp}}  {% if asset.type == "color" %}
{{sp}}  static let {{asset.name|swiftIdentifier:"pretty"|lowerFirstWord|escapeReservedKeywords}} = {{colorType}}(name: "{{asset.value}}")
{{sp}}  {% elif asset.type == "image" %}
{{sp}}  static let {{asset.name|swiftIdentifier:"pretty"|lowerFirstWord|escapeReservedKeywords}} = {{imageType}}(name: "{{asset.value}}")
{{sp}}  {% elif asset.items %}
{{sp}}  enum {{asset.name|swiftIdentifier:"pretty"|escapeReservedKeywords}} {
{{sp}}    {% set sp2 %}{{sp}}  {% endset %}
{{sp}}    {% call casesBlock asset.items sp2 %}
{{sp}}  }
{{sp}}  {% endif %}
{{sp}}  {% endfor %}
{% endmacro %}
{% macro allValuesBlock assets filter prefix sp %}
{{sp}}  {% for asset in assets %}
{{sp}}  {% if asset.type == filter %}
{{sp}}  {{prefix}}{{asset.name|swiftIdentifier:"pretty"|lowerFirstWord|escapeReservedKeywords}},
{{sp}}  {% elif asset.items %}
{{sp}}  {% set prefix2 %}{{prefix}}{{asset.name|swiftIdentifier:"pretty"|escapeReservedKeywords}}.{% endset %}
{{sp}}  {% call allValuesBlock asset.items filter prefix2 sp %}
{{sp}}  {% endif %}
{{sp}}  {% endfor %}
{% endmacro %}

enum {{enumName}} {
{% if catalogs.count > 1 %}
{% for catalog in catalogs %}
enum {{catalog.name|swiftIdentifier:"pretty"|escapeReservedKeywords}} {
{% call enumBlock catalog.assets "  " %}
}
{% endfor %}
{% else %}
{% call enumBlock catalogs.first.assets "" %}
{% endif %}
}

extension {{imageAlias}} {
convenience init!(asset: {{imageType}}) {
let bundle = Bundle(for: BundleToken.self)
self.init(named: asset.name, in: bundle, compatibleWith: nil)
}
}

private final class BundleToken {}
{% else %}
// No assets found
{% endif %}
```
## 2.建立RunScript
- 在`Xcode`中，进入到项目的`Target`，选择`Build Phases`,然后点击左上角的 `+` 号后点击 `New Run Script Phase`在新建的RunScript里添加如下内容：
    ```bash
    if which "$PROJECT_DIR"/SwiftGen5/bin/swiftgen >/dev/null;
    then
    set -e
    "$PROJECT_DIR"/SwiftGen5/bin/swiftgen xcassets -t my-swift4 "$PROJECT_DIR/swiftGenExample/Assets.xcassets" --output "$PROJECT_DIR/swiftGenExample/ImageCode/ImageAsset.swift"
    else
    echo "##run echo warning: SwiftGen not installed, download it from https://github.com/SwiftGen/SwiftGen"
    fi
    ```
    这段 `Run Script` 作用就是利用 SwiftGen 生成代码后写入到 `ImageAsset.swift` 文件中。

- Build 一下project，我们就可以在 `/swiftGenExample/ImageCode/` 目录下看到 `ImageAsset.swift`，此时该文件还没有被项目索引，所以把它拖进项目Xcode对应的目录下就行了，之后即使我们添加了新的图片或者删掉旧的图片，只要每次Build一下项目，代码就会自动更新了。

下面是生成的 `ImageAsset.swift` 的代码：
```Swift
// ImageAsset.swift
// Generated using SwiftGen, using my-templete created by Hanson

import UIKit.UIImage

typealias Image = UIImage

@available(*, deprecated, renamed: "ImageAsset")
typealias AssetType = ImageAsset

struct ImageAsset {
    fileprivate var name: String

    var image: Image {
        let bundle = Bundle(for: BundleToken.self)
        let image = Image(named: name, in: bundle, compatibleWith: nil)
        guard let result = image else { fatalError("Unable to load image named \(name).") }
        return result
    }
}

enum Asset {
    static let arrowBlue = ImageAsset(name: "arrow_blue")
    static let arrowBrown = ImageAsset(name: "arrow_brown")
    static let iconLeftBack = ImageAsset(name: "icon_left_back")
    static let startLogo = ImageAsset(name: "start_logo")
}

extension Image {
    convenience init!(asset: ImageAsset) {
    let bundle = Bundle(for: BundleToken.self)
    self.init(named: asset.name, in: bundle, compatibleWith: nil)
    }
}

private final class BundleToken {}
```
# 结语
这里只是利用了 `SwiftGen` 对于 `Image` 的部分。它还有其它的关于 `String` ，`StroyBoard`，`Font`等等的代码生成。原理基本相同，靠大家按需研究啦。

> 到我的博客阅读：[myhanson.com](http://www.myhanson.com/2017/12/08/%E7%94%A8SwiftGen%E7%AE%A1%E7%90%86UIImage%E7%AD%89%E7%9A%84String-based%E6%8E%A5%E5%8F%A3/#more)
> 本文Demo：[SwiftGenExample](https://github.com/zyphs21/SwiftGenExample)


