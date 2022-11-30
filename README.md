# Guide-to-developing-IntelliJ-plugins
Advanced guide to developing IntelliJ IDEA plugins.

README: [English](https://github.com/bytebeats/Guide-to-developing-IntelliJ-plugins/blob/main/en/README.md) | [中文](https://github.com/bytebeats/Guide-to-developing-IntelliJ-plugins/blob/main/cn/README-zh.md)

Previous: [English](https://github.com/bytebeats/Guide-to-developing-IntelliJ-plugins/blob/main/en/introduction.md) | [中文](https://github.com/bytebeats/Guide-to-developing-IntelliJ-plugins/blob/main/cn/introduction-zh.md)

Reference: [Blog](https://developerlife.com/2021/03/13/ij-idea-plugin-advanced/)

## Overview

This article is a reference covering a lot of advanced topics related to creating IDEA plugins using the JetBrains IntelliJ Platform SDK. However there are many advanced topics that are not covered here (such as [custom language support](https://plugins.jetbrains.com/docs/intellij/custom-language-support.html)).

In order to be successful creating advanced plugins, this is what I suggest:
1. Read thru the [Introduction to creating IntelliJ IDEA plugins](https://developerlife.com/2020/11/21/idea-plugin-example-intro/) in detail. Modify the [idea-plugin-example](https://github.com/nazmulidris/idea-plugin-example), [idea-plugin-example2](https://github.com/nazmulidris/idea-plugin-example2), or [shorty-idea-plugin](https://github.com/r3bl-org/shorty-idea-plugin) to get some hands on experience working with the plugin APIs.
2. Use the [Official JetBrains IntelliJ Platform SDK](https://plugins.jetbrains.com/docs/intellij/welcome.html) docs, along with the source code examples (from the list of repos above) to get a better understanding of how to create plugins. 
3. When you are looking for something not in tutorials here (on developerlife.com) and in the official docsd, then search the [intellij-community](https://github.com/JetBrains/intellij-community) repo itself to find code examples on how to use APIs that might not have any documentation. An effective approach is to find some functionality in IDEA that is similar to what you are looking to build, and then locate the source code for that feature in the repo and see how JetBrains has done it.

## [IDEA threading model](https://github.com/bytebeats/Guide-to-developing-IntelliJ-plugins/blob/main/en/idea_threading_model.md)

* What was submitTransaction() all about?
* What is a write-safe context?
* What is the replacement for submitTransaction()?
* invokeLater() and ModalityState
* How to use invokeLater to perform asynchronous and synchronous tasks
  * Synchronous execution of Runnable (from an already write-safe context using submitTransaction()/invokeLater())
  * Asynchronous execution of Runnable (from a write-unsafe / write-safe context using submitTransaction())
* Side effect - invokeLater() vs submitTransaction() and impacts on test code

## [PSI access and mutation](https://github.com/bytebeats/Guide-to-developing-IntelliJ-plugins/blob/main/en/psi_access_and_mutation.md)

* How to create a PSIFile or get a reference to one
* PSI Access
  * Visitor pattern for top down navigation (without threading considerations)
  * Threading, locking, and progress cancellation check during PSI access
  * Background tasks and multiple threads - The incorrect way
  * Background tasks and multiple threads - The correct way
* PSI Mutation
  * PsiViewer plugin
  * Generate PSI elements from text
  * Example of walking up and down PSI trees to find elements
  * Finding elements up the tree (parents)
  * Finding elements down the tree (children)
  * Threading considerations
* Additional references

## [Dynamic plugins](https://github.com/bytebeats/Guide-to-developing-IntelliJ-plugins/blob/main/en/dynamic_plugins.md)

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

## [UI(JetBrains UI components, Swing and Kotlin UI DSL)](https://github.com/bytebeats/Guide-to-developing-IntelliJ-plugins/blob/main/en/intellij_plugin_sdk_ui.md)

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

## MIT License

    Copyright (c) 2021 Chen Pan

    Permission is hereby granted, free of charge, to any person obtaining a copy
    of this software and associated documentation files (the "Software"), to deal
    in the Software without restriction, including without limitation the rights
    to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
    copies of the Software, and to permit persons to whom the Software is
    furnished to do so, subject to the following conditions:

    The above copyright notice and this permission notice shall be included in all
    copies or substantial portions of the Software.

    THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
    IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
    FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
    AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
    LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
    OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
    SOFTWARE.
