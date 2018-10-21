---
title: Android多渠道打包-变种打包配置
date: 2018-05-18 15:25:41
tags:
---
**通过productFlavor用来为app创建不同的版本，如：免费版和付费版、不同应用市场的渠道包等。**
创建方式：

### Gradle中配置
```
android {
    productFlavors {
        nbsylzxVersion {           //渠道1
            applicationId "com.wangtao.nbsylzx"
            manifestPlaceholders = [package_name        : "com.wangtao.nbsylzx",
                                    umeng_appkey        : "5a1b7cb2f29d98",
                                    umeng_message_secret: "46b6a40c6a749f",
                                    baiduMap_appKey     : "MnaZOYCPOIqozGoSle"]
                versionCode 1
                versionName "V1.0.1"
            }
        fhzyyVersion {             //渠道2
            applicationId "com.wangtao.fhzyy"
            manifestPlaceholders = [package_name        : "com.wangtao.fhzyy",
                                    umeng_appkey        : "59b8da5f734b",
                                    umeng_message_secret: "1b652c87cd9d",
                                    baiduMap_appKey     : "U0pNbvVRZHwTfP"]
            versionCode 2
            versionName "V1.0.2"
        }
    }
}
```
> 通过这种方式可以配置不同的包名、友盟key、百度key等。

### AndroidManfest.xml中获取key
```
<!--百度地图-->
<meta-data
    android:name="com.baidu.lbsapi.API_KEY"
    android:value="${baiduMap_appKey}" />
<!--友盟-->
<meta-data
    android:name="UMENG_APPKEY"
    android:value="${umeng_appkey}" />
<meta-data
    android:name="UMENG_MESSAGE_SECRET"
    android:value="${umeng_message_secret}" />
<meta-data
    android:name="UMENG_CHANNEL"
    android:value="Umeng" />
```
> 通过${名称}来获取gradle中的配置

### 创建对应文件夹
- 在app-src中创建与main同层级的文件夹
- 文件夹名称与productFlavors中配置的要一样

   -- main
   -- nbsylzxVersion
   -- fhzyyVersion
- 对应文件夹中也可以创建java/res

    -- main
    -----java
    -----res
    --nbsylzxVersion
    -----java
    -----res
> main-java中的文件与变种文件夹中的.java文件不能重名
> main-res中的资源文件与变种文件夹中的资源文件相同，优先使用变种文件夹资源文件

### 不同依赖配置
在Gradle中dependencies{}中配置依赖，渠道名称+Compile
```
dependencies{
    nbsylzxVersionCompile project(':baiduMap')
    fhzyyVersionCompile 'com.github.bumptech.glide:glide:4.0.0'
}
```
### 解决友盟多包名无法推送问题问题
如果：
productFlavors中多包名

- applicationId：com.test.a
- applicationId：com.test.b
- AndroidManfest.xml中
 package=”com.test.c”

那么在Application中初始化友盟时增加以下：
```
mPushAgent.setResourcePackageName("com.test.c");
```

