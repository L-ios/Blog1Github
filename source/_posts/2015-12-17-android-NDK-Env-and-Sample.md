title: android_NDK_Env_and_Sample
date: 2015-12-17 00:38:17
tags:
---

### Android Studio 中NDK的环境搭建

1. 通过Android Studio中 Settings -> Appearance & Behavior -> System Settings -> Android SDK, 通过启动的SDK Platforms 和SDK Tools标签页来下载SDK和NDK
2. 通过 Project Settings,我们为项目设置ndk的路径, 然后会在{project_root}/local.properties 中我们可以看到
```
ndk.dir=/opt/android-sdk/ndk-bundle
```

3. 在{project_root}/gradle.properties 最后需要添加一行
```
android.useDeprecatedNdk=true
```

    添加这行后，gradle编译，建立index的时候就会提供ndk的补全和ndk的一些支持。
4. 为将要生成的so文件定义名字，在{project_root}/{module_name}/build.gradle中的android->defaultConfig中添加一个属性，如下：
```
        ndk {
            moduleName "libMyLib"
        }
```

### 一些ndk编码的技巧
1. 编译一下项目，主要是为了生成头文件做准备。
2. 首先我们需要通过javah来生成我们需要的头文件(*.h)，代码如下
```
$ javah -d jni -classpath /opt/android-sdk/platforms/android-23/android.jar:/opt/android-sdk/extras/android/support/design/libs/android-support-design.jar:/opt/android-sdk/extras/android/support/v7/appcompat/libs/android-support-v4.jar:/opt/android-sdk/extras/android/support/v7/appcompat/libs/android-support-v7-appcompat.jar:../../build/intermediates/classes/debug/ ower.lionseun.lib8.ndksample.MainActivity
```
    - -d 指定生成的头文件所在的目录
    - -classpath 指定在生成头文件时需要加载的一些jar包，多一个jar包我们采用“:”进行隔断添加
    - 最后指定的是需要生成头文件的类

3. 为其头文件建立相对的cpp文件，这样，最好名字一样，这样有利于后面的一个ide的帮助。

4. 在有native本地函数的地方使用
```
    static {
        System.loadLibrary("MyLib");
    }
```
使用静态代码块加载我们将会编译生成的so文件

5. 当我们在有native的类中再次定义一个native函数，且就可以利用Android Studio中特有的机制，来自动的生成函数名
