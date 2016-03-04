title: CM_Theme_Analysis

date: 2015-10-07 10:08:31

tags:
-----

#### 结构分析
1.主题采用的是内容提供者，也就是采用数据库的方式进行存储，
    a)表结构----framework/base/core/java/android/provider/ThemesContract.ThemesColumns
    b)其中采用了AIDL（现在还不懂这是是什么鬼），有两个AIDL文件在其中涉及的比较多，frameworks/base/core/java/android/content/res/IThemeService.aidl
        其实方法的具体实现在
        frameworks/base/services/core/java/com/android/server/ThemeService.java
        因为aidl的接口，其类为*.Stub类
        （在AIDL中定义的接口函数，可以通过查找
        [name].aidl => [name].java => [name].Stub 类的子类就可以找到具体实现).
        ```
        # 可以利用以下shell代码查找
        $ find [path] -name [expression] | xargs grep [WantToFindExpression]
        ```
### 理解
##### 2015-10-08
根据现在已经看了的代码，其修改框架是：
    1. 在系统（framework）中内置服务进行图标的修改，进行数据的操作，当安装了一个应用的过程中，
    就判断其是否为一个theme应用，若为一个theme应用，就让ThemeService进行处理，并进行一些插入
    更新的数据的操作（现在不了解是否将所有的主题都进行了数据的插入以及解压到/system/theme/中，
    因为在某处为其设置了文件夹的大小限制），并且提供了应用的图标修改要求。
    2. 在应用级（package）中提供一个内容提供者，但是却不知道是否为framework中的服务进行了查询
    服务。
