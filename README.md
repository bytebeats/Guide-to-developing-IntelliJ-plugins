# Guide-to-developing-IntelliJ-plugins
Advanced guide to developing IntelliJ IDEA plugins.

README: [English](https://github.com/bytebeats/Guide-to-developing-IntelliJ-plugins/blob/main/README.md) | [中文](https://github.com/bytebeats/Guide-to-developing-IntelliJ-plugins/blob/main/README-zh.md)

Previous: [English](https://github.com/bytebeats/Guide-to-developing-IntelliJ-plugins/blob/main/introduction.md) | [中文](https://github.com/bytebeats/Guide-to-developing-IntelliJ-plugins/blob/main/introduction-zh.md)

Reference: [Blog](https://developerlife.com/2021/03/13/ij-idea-plugin-advanced/)

## Overview

This article is a reference covering a lot of advanced topics related to creating IDEA plugins using the JetBrains IntelliJ Platform SDK. However there are many advanced topics that are not covered here (such as [custom language support](https://plugins.jetbrains.com/docs/intellij/custom-language-support.html)).

In order to be successful creating advanced plugins, this is what I suggest:
1. Read thru the [Introduction to creating IntelliJ IDEA plugins](https://developerlife.com/2020/11/21/idea-plugin-example-intro/) in detail. Modify the [idea-plugin-example](https://github.com/nazmulidris/idea-plugin-example), [idea-plugin-example2](https://github.com/nazmulidris/idea-plugin-example2), or [shorty-idea-plugin](https://github.com/r3bl-org/shorty-idea-plugin) to get some hands on experience working with the plugin APIs.
2. Use the [Official JetBrains IntelliJ Platform SDK](https://plugins.jetbrains.com/docs/intellij/welcome.html) docs, along with the source code examples (from the list of repos above) to get a better understanding of how to create plugins. 
3. When you are looking for something not in tutorials here (on developerlife.com) and in the official docsd, then search the [intellij-community](https://github.com/JetBrains/intellij-community) repo itself to find code examples on how to use APIs that might not have any documentation. An effective approach is to find some functionality in IDEA that is similar to what you are looking to build, and then locate the source code for that feature in the repo and see how JetBrains has done it.

## [IDEA threading model](https://github.com/bytebeats/Guide-to-developing-IntelliJ-plugins/blob/main/idea_threading_model.md)

## [PSI access and mutation](https://github.com/bytebeats/Guide-to-developing-IntelliJ-plugins/blob/main/psi_access_and_mutation.md)

## [Dynamic plugins](https://github.com/bytebeats/Guide-to-developing-IntelliJ-plugins/blob/main/dynamic_plugins.md)

## [VFS, Document, PSI](https://github.com/bytebeats/Guide-to-developing-IntelliJ-plugins/blob/main/vfs_document_psi.md)

## [UI(JetBrains UI components, Swing and Kotlin UI DSL)](https://github.com/bytebeats/Guide-to-developing-IntelliJ-plugins/blob/main/intellij_plugin_sdk_ui.md)
