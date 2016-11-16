layout: draft

title: learn Android Loader

date: 2015-09-21 20:39:16

tags:
    - Android
    - Loader
---

### 阅读笔记（来之官方文档）
-   1、一个Activity/Fragment仅有一个LoaderManager，但其管理着一个及以上loader，
-   2、应用使用Loader的条件：
    -   a) 有一个Activity/Fragment
    -   b) 有一个LoaderManager实例
    -   c) 存在一个Loader子类实例，例如CursorLoader
    -   d) 一种可以显示Loader数据格式的方式，例如SimpleCursorLoader
    -   e) 数据源，如果采用的是CursorLoader，就应当采用ContentProvider
