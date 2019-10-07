---
title: ApkChannelPackage插件接入文档
date: 2017-04-09 22:47:02
type: "Android"
tags:
 - v2
 - ApkChannelPackage
categories:
 - Android
---
## 多渠道打包方式选择
[APKChannelPackage](https://github.com/ltlovezh/ApkChannelPackage)是一个多渠道打包插件，该插件会自动检测基础包是V1签名还是V2签名，并使用不同的多渠道打包方式。
目前Gradle Plugin 2.2以上默认开启V2签名，所以如果想关闭V2签名，可将下面的`v2SigningEnabled`设置为false。
``` groovy
signingConfigs {
        release {
            ...
            v1SigningEnabled true
            v2SigningEnabled false
        }

        debug {
            ...
            v1SigningEnabled true
            v2SigningEnabled false
        }
}
```

<!--more-->

## 接入流程

### 1.在根工程的`build.gradle`中，添加对打包Plugin的依赖：
``` groovy
dependencies {
        classpath 'com.android.tools.build:gradle:2.2.0'
        classpath 'com.leon.channel:plugin:1.0.4'
}
```

### 2.在主App工程的`build.gradle`中，添加对ApkChannelPackage Plugin的引用：
``` groovy
apply plugin: 'channel'
```

### 3.在主App工程的`build.gradle`中，添加读取渠道信息的helper类库依赖：
``` groovy
dependencies {
    compile 'com.leon.channel:helper:1.0.4'
}
```

### 4.在gradle.properties文件中，配置渠道文件名称
``` groovy
channel_file=channel.txt
```
其中channel.txt即为包含渠道信息的文件，需放置在根工程目录下，一行一个渠道信息。

### 5.渠道包信息配置
若是直接编译生成多渠道包，则通过`channel`标签配置：
``` groovy
channel{
    //多渠道包的输出目录，默认为new File(project.buildDir,"channel")
    baseOutputDir = new File(project.buildDir,"xxx")
    //多渠道包的命名规则，默认为：${appName}-${versionName}-${versionCode}-${flavorName}-${buildType}
    apkNameFormat ='${appName}-${versionName}-${versionCode}-${flavorName}-${buildType}'
}
```

其中，多渠道包的命名规则中，可使用以下字段：

* appName ： 当前project的name
* versionName ： 当前Variant的versionName
* versionCode ： 当前Variant的versionCode
* buildType ： 当前Variant的buildType，即debug or release
* flavorName ： 当前的渠道名称
* appId ： 当前Variant的applicationId

若是根据已有基础包生成多渠道包，则通过`rebuildChannel`标签配置：
``` groovy
rebuildChannel {
baseDebugApk = 已有Debug APK    
baseReleaseApk = 已有Release APK
//默认为new File(project.buildDir, "rebuildChannel/debug")
debugOutputDir = Debug渠道包输出目录   
//默认为new File(project.buildDir, "rebuildChannel/release")
releaseOutputDir = Release渠道包输出目录
}
```
这里要注意一下，已有APK的名字必须包含`base`字符串，这样插件生成多渠道包时，会用当前的渠道替换`base`字符串，形成新的渠道包。

### 6.生成多渠道包
若没有通过Gradle Plugin的 `productFlavors`配置多渠道，那么通过以下Task
`channelDebug` 、`channelRelease`分别负责生成Debug和Release的多渠道包。

若是配置了`productFlavors`，那么对应的Task则是`channelFlavorXDebug`、`channelFlavorXRelease`，FlavorX表示在`productFlavors`中配置的渠道名称。

除此之外，如果是根据已有基础包生成多渠道包，那么对应的Task则是`reBuildChannel`。

### 7.读取渠道信息
通过helper类库中的`ChannelReaderUtil`类读取渠道信息。
``` java
String channel = ChannelReaderUtil.getChannel(getApplicationContext());
```
如果没有渠道信息，那么这里返回`null`，开发者需要自己判断。

### 最新版本

ApkChannelPackage的最新版本请在[GitHub](https://github.com/ltlovezh/ApkChannelPackage)上及时获取哈！
