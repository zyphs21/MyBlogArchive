---
title: 通过RunScript给iOS项目自增版本号(Versioin和Build)
date: 2018-03-14 00:38:20
---

# 需求分析
- 在打包应用之后，需要自增 **Version 的最后一位** 和 **Build** 的值。
![](https://user-gold-cdn.xitu.io/2018/3/14/162201d291ebc6c1?w=690&h=136&f=png&s=11268)
- 只在 Archive(Release) 的时候触发该自增。

<!-- more -->

# 添加 RunScript
在 `项目Target` -> `Build Phases` -> `点击+号` -> `New Run Script Phase`

然后添加如下内容：
```Bash
if [ $CONFIGURATION == Release ]; then
echo "当前为 Release Configuration,开始自增 Build"
plist=${INFOPLIST_FILE}
buildnum=$(/usr/libexec/PlistBuddy -c "Print CFBundleVersion" "${plist}")
if [[ "${buildnum}" == "" ]]; then
echo "Error：在Plist文件里没有 Build 值"
exit 2
fi
buildnum=$(expr $buildnum + 1)
/usr/libexec/PlistBuddy -c "Set CFBundleVersion $buildnum" "${plist}"

echo "开始自增 Version 最后一位"
versionNum=$(/usr/libexec/PlistBuddy -c "Print CFBundleShortVersionString" "${plist}")
thirdPartVersonNum=`echo $versionNum | awk -F "." '{print $3}'`
thirdPartVersonNum=$(($thirdPartVersonNum + 1))
newVersionStr=`echo $versionNum | awk -F "." '{print $1 "." $2 ".'$thirdPartVersonNum'" }'`
/usr/libexec/PlistBuddy -c "Set CFBundleShortVersionString $newVersionStr" "${plist}"
else
echo $CONFIGURATION "当前不为 Release Configuration"
fi
```

# 注意
因为我的版本号是`xx.xx.xx`这样的形式，所以我以 `.` 拆分版本号后，取出第三个值来增加，最后再拼接回来。
```
versionNum=$(/usr/libexec/PlistBuddy -c "Print CFBundleShortVersionString" "${plist}")
# 这里取出第三个值
thirdPartVersonNum=`echo $versionNum | awk -F "." '{print $3}'`
```
