---
title: 升级 Android plugin 3.0 & Gradle 4.1
date: 2018-01-22 23:09:33
categories:
- 技术向
tags:
- Android
- Gradle
---

Android Studio 3.0 正式版本已经发布许久，公司的多个项目也需要升级工具并且更新项目的 Gradle 了。先跟着官方的升级说明学习一下。以下为部分内容的摘要，更多内容还请查阅：

* [Android Plugin for Gradle Release Notes](https://developer.android.google.cn/studio/releases/gradle-plugin.html)
* [Migrate to Android Plugin for Gradle 3.0.0](https://developer.android.google.cn/studio/build/gradle-plugin-3-0-0-migration.html)
* [Use Java 8 Language Features](https://developer.android.google.cn/studio/write/java8-support.html)

### Gradle 4.1

Gradle 4.1 开始支持 Google 的 Maven 仓库了，只需要在仓库列表中添加 google() 即可。最新的 gradle Android plugin 也是通过这个仓库下载的。

```gradle
buildscript {
    repositories {
        google()
        ...
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.0.1'
    }
}
```

记得，把 gradle 相应的版本更新一下：

`distributionUrl = https\://services.gradle.org/distributions/gradle-4.1-all.zip`

<!--more-->

在最新版本的 Gradle 4.1 及 Android plugin 3.0 以上版本的配合下，提高了对大型项目的编译性能。根据官方介绍在一个有着 130 多个模块的工程中，速度提升效果如下所示：

|Android plugin version + Gradle version |	Android plugin 2.2.0 + Gradle 2.14.1|	Android plugin 2.3.0 + Gradle 3.3|	Android plugin 3.0.0 + Gradle 4.1|
|--|--|--|
|Configuration (e.g. running ./gradlew --help)	|~2 mins|	~9 s	|~2.5 s|
|1-line Java change (implementation change)|	~2 mins 15 s	|~29 s	|~6.4 s|
 
默认情况下，Android plugin 使用最低要求的 buildToolsVersion。所以，你不再需要为构建工具指定版本现在可以 **删除 android.buildToolsVersion** 属性。


### 构建优化

1. 优化了 multi module 的项目的并行构建速度
2. 使用新的依赖配置可以提高构建性能：implementation, api, compileOnly, runtimeOnly
  >在下一版本中，将会移除相应的 compile, provided, apk（这一项内容具体查看相应的迁移链接）

  ![](http://upload-images.jianshu.io/upload_images/2946447-bbe5bb275a281bae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3. 每个 Class 都被编译成单独的 DEX 文件，所以编译时只需要重编译修改的 Class
4. 通过 chached outputs 来提高再次构建的速度，只要在 gradle.properties 中配置：`org.gradle.caching=true`
5. 默认使用 AAPT2 来支持资源的增量编译。如果编译发生失败，可以报告 Bug 并将其关闭：`android.enableAapt2=false`


### Java8
支持了 Java8 的特性，例如：

|Java 8 Language Feature|	Compatible minSdkVersion|
|--|--|
|Lambda expressions|	Any. Note that Android does not support the serialization of lambda expressions.|
|Method References|	Any.|
|Type Annotations	|Any. However, type annotation information is available at compile time, but not at runtime. Also, the platform supports TYPE in API level 24 and below, but not ElementType.TYPE_USE or ElementType.TYPE_PARAMETER.|
|Default and static interface methods	|Any.|
|Repeating annotations	|Any.|

另外，还有一些特性的支持对 minSdkVersion 有要求，具体的查看对应说明即可。

简单原理示意图：

![](https://developer.android.google.cn/studio/images/write/desugar_2x.png)

为了支持 Java8 的特性，你需要做的也很简单：

1. 指定 Java8

  在使用 Java8 语法的 Module 中添加以下配置
  ```
  android.compileOptions {
      sourceCompatibility JavaVersion.VERSION_1_8
      targetCompatibility JavaVersion.VERSION_1_8
  }
  ```
2. 移除 Jack

  事实上，Android 已经弃用 Jack 了。当然，你只需要移除即可。
  ```
  android.defaultConfig {
      ...
      // Remove this block.
      jackOptions {
          enabled true
          ...
      }
  }
  ```

  对于应用开发者来说，可以指定 dexer 为 D8，这是 Google 推出的工具。你的开发不需要关注 D8 的细节。

  ```
  android.enableD8=true
  ```
  D8 更快，编译产物更小，甚至能够提高运行时的性能。

3. 移除 Retrolambda
  相信添加了 Retrolambda 相关配置的知道如何移除吧，把对应的语句删除即可。

如果你在使用中发现了问题，可以禁用掉它：

```
android.enableDesugar=false
```

### 行为变更

1. 移除了 outputFile() 及 processManifest.manifestOutputFile() 的使用

2. crunchPngs

  因为 crunchPngs 会增加构建时间 ，所以在大量 PNG 的项目中，非 Release 版本可以禁用处理。Debug 版本默认为关闭。
  ```
  android {
    buildTypes {
      release {
        // Disables PNG crunching for the release build type.
        crunchPngs false
      }
    }
  }
  ```
  对于这一点我的理解是如果有一个 buildTypes 是非 Realease 版本的而又不是 Debug 版本，那么为了提高构建速度，需要手动设置该属性。

3. 以前使用的第三方 android-apt plugin 不被支持，可以直接在依赖中使用 annotationProcessor

4. 对于构建变体的约束更强了

  必需指定 productFlavors 所属的 flavorDimensions。
  ```
  // Specifies two flavor dimensions.
  flavorDimensions "tier", "minApi"

  productFlavors {
      free {
        // Assigns this product flavor to the "tier" flavor dimension. Specifying
        // this property is optional if you are using only one dimension.
        dimension "tier"
        ...
      }

      paid {
        dimension "tier"
        ...
      }

      minApi23 {
          dimension "minApi"
          ...
      }

      minApi18 {
          dimension "minApi"
          ...
      }
  }
  ```

  另外，如果 App 和 Library 模块的 buildtype 和 flavors 不完全匹配，则需要使用 DSL 描述来解决。
  ![](http://upload-images.jianshu.io/upload_images/2946447-6afe0abb71715433.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  ![](http://upload-images.jianshu.io/upload_images/2946447-120680f144255c73.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  ![](http://upload-images.jianshu.io/upload_images/2946447-2258b7cb451453ba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



5. 遍历：all

  如果之前使用 each 遍历 Variants，则需要开始使用 all()。 这是因为 each() 仅在 configuration time 遍历已经存在的对象，但在新的体系中，这些对象还不存在。 而 all() 通过在执行过程中添加对象来遍历。
  ```
  android.applicationVariants.all { variant ->
      variant.outputs.all {
          outputFileName = "${variant.name}-${variant.versionName}.apk"
      }
  }
  ```

6. proguard 

  为 Library Module 中设置 ProGuard 不起作用。只需要在 app module 中设置即可。

### 其它

还有一些其他内容，由于了解不多，就不展开介绍，可以自行查看。

* 一些对 androidTest 的支持
* 字体资源的支持，Oreo 的特性之一（国内没有这个条件）
* Instant Apps 支持
* NDK项目可指定输出目录
  ```
  android {
      ...
      externalNativeBuild {
          // For ndk-build, instead use the ndkBuild block.
          cmake {
              ...
              // Specifies a relative path for outputs from external native
              // builds. You can specify any path that's not a subdirectory
              // of your project's temporary build/ directory.
              buildStagingDirectory "./outputs/cmake"
          }
      }
  }
  ```

* 支持使用 Cmake 3.7 及以上版本进行 native 项目构建

* 支持自定义的 Lint 检查项目。

  lintChecks project(':lint-chcks')，这个对该 Lint 工程有要求。
