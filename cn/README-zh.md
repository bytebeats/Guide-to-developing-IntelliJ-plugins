# Guide-to-developing-IntelliJ-plugins
IntelliJ IDEA 插件开发高级指南.

README: [English](https://github.com/bytebeats/Guide-to-developing-IntelliJ-plugins/blob/main/README.md) | [中文](https://github.com/bytebeats/Guide-to-developing-IntelliJ-plugins/blob/main/README-zh.md)

前置文章: [English](https://github.com/bytebeats/Guide-to-developing-IntelliJ-plugins/blob/main/introduction.md) | [中文](https://github.com/bytebeats/Guide-to-developing-IntelliJ-plugins/blob/main/introduction-zh.md)

引用: [博文](https://developerlife.com/2021/03/13/ij-idea-plugin-advanced/)

## 预览

本文是一篇集子, 覆盖了许多关于通过JetBrains IntelliJ Platform SDK 开发 IntelliJ 平台插件的高级主题. 然而依然有许多高级主题未能引入(例如[自定义语言支持](https://plugins.jetbrains.com/docs/intellij/custom-language-support.html)).

为了能够成功地创建高级插件, 我建议:
1. 详细阅读[IntelliJ IDEA插件开发简介](https://developerlife.com/2020/11/21/idea-plugin-example-intro/). 修改[idea-plugin-example](https://github.com/nazmulidris/idea-plugin-example), [idea-plugin-example2](https://github.com/nazmulidris/idea-plugin-example2), 或者[shorty-idea-plugin](https://github.com/r3bl-org/shorty-idea-plugin)以获取插件 API 的一手使用经验.
2. 使用[官方JetBrains IntelliJ Platform SDK](https://plugins.jetbrains.com/docs/intellij/welcome.html)文档, 以及示例代码(来自以上仓库列表)以更好地理解如何开发插件. 
3. 当你要查找一些东西不在使用说明(developerlife.com)和官方文档的时候, 请搜索[intellij-community](https://github.com/JetBrains/intellij-community)仓库本身, 来查找没有任何文档的 API 的代码示例. 有效的方式是: 在 IDEA 中找到一些与你正在查找的代码相似的函数, 然后在仓库中定位源码, 看看 JetBrains 是如何使用的.

## [IDEA线程模型](https://github.com/bytebeats/Guide-to-developing-IntelliJ-plugins/blob/main/en/idea_threading_model.md)

* 什么是 submitTransaction() 函数?
* 什么是写安全的上下文?
* submitTransaction()的替代函数是什么?
* invokeLater()和ModalityState
* 如何使用invokeLater执行异步和同步任务?
  * Runnable的同步执行 (在写安全的上下文中使用submitTransaction()/invokeLater())
  * Runnable的异步执行 (在写安全和写不安全的上下文中使用submitTransaction())
* 负作用 - invokeLater() vs submitTransaction() 以及对测试代码的影响

## [PSI访问和状态改变](https://github.com/bytebeats/Guide-to-developing-IntelliJ-plugins/blob/main/en/psi_access_and_mutation.md)

* 如何创建和引用PSI文件?
* PSI 访问
  * 自顶向下导航的访问者模式(无线程顾虑)
  * PSI访问期间的线程, 阻塞和进度取消检测
  * 后台任务和多线程 - 错误方式
  * 后台任务和多线程 - 正确方式
* PSI状态改变
  * PsiViewer插件
  * 从文本生成 PSI 元素
  * 示例: 遍历 PSI 树查找元素
  * 向上查找父元素
  * 向下查找子元素
  * 线程顾虑
* 更多引用

## [动态插件](https://github.com/bytebeats/Guide-to-developing-IntelliJ-plugins/blob/main/en/dynamic_plugins.md)

* Extension points postStartupActivity, backgroundPostStartupActivity to initialize a plugin on project load
* Light services
* Migration strategies
  1. Component -> Service
  2. Component -> postStartupActivity
  3. Component -> postStartupActivity + Service
  4. Component -> projectListener
  5. Component -> projectListener + Service
  6. Delete Component
  7. Component -> AppLifecycleListener

## [VFS, Document, PSI](https://github.com/bytebeats/Guide-to-developing-IntelliJ-plugins/blob/main/en/vfs_document_psi.md)

* VFS
  * Getting a list of all the virtual files in a project
  * Attach listeners to see changes to virtual files programmatically
  * Attach listeners to see changes to virtual files declaratively
  * Asynchronously process file system events
  * Intercept when the currently open file gets saved
* Document
  * Example of an action that uses the Document API

## [UI(JetBrains UI components, Swing and Kotlin UI DSL)](https://github.com/bytebeats/Guide-to-developing-IntelliJ-plugins/blob/main/enintellij_plugin_sdk_ui.md)

* Simple UI components - notifications, popups, dialogs
  * Notifications
  * Popups
  * Dialogs
* Create complex UIs using Kotlin UI DSL (forms, dialogs, settings screens)
  * Understanding the structure of the DSL (layout, row, and cell)
  * How to bind data to the form UI
  * What UI elements are available for use in the forms
  * How to display a form in a dialog
  * How to persist the data created/selected by the user, between IDE restarts (PersistentStateComponent)
  * Example of a form UI
  * MigLayout, panel, and grow
* Create IDE Settings UI for plugin
* Complex UI creation in dialogs
* Adding your plugin UI in Tool windows
  * 1. Declarative tool window
  * 2. Programmatic tool window
  * Indices and dumb aware
  * Creating a content for any kind of tool window
  * Content closeability
* Add Line marker provider in your plugin
  * Example of a provider for Markdown language
  * 1. Declare dependencies
  * 2. Register the provider in XML
  * 3. Provide an implementation of LineMarkerProvider
  * 4. Provide a more complex implementation of LineMarkerProvider
