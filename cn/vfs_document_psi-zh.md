## VFS, Document, PSI

VFS(Virtual File System, 虚拟文件系统)是IntelliJ平台中的一个抽象概念, 它允许访问计算机上的文件(本地文件系统上的文件, 甚至是JAR文件中的文件), 仓库中的文件或网络上的文件. 有多种方法可以访问集成开发环境中的文件内容:

1. VFS 允许访问最底层的文件(最接近实际文件系统, 而不是像 PSI 这样的抽象文件系统).
2. Document提供了一个对象模型, 可以以纯文本的形式访问文件内容(因此它介于 VFS 和 PSI 之间).
3. PSI(程序结构接口)允许以分层对象模型访问文件内容, 该模型考虑了特定语言的语法和语义(有点像 DOM 在网页浏览器中表示 HTML 和 CSS 内容的方式). 有关 PSI 的更多信息, 请参阅本节.

### VFS

VFS 是 IntelliJ 平台用于处理文件的工具. 它提供:

1. 用于处理文件的通用 API, 无论文件位于何处(磁盘, JAR 文件, HTTP 服务器, VCS 等).
2. 用于跟踪文件修改的信息, 并在检测到文件发生变化时提供文件的新旧版本.
3. 能力用于在 VFS 中将附加的持久数据与文件提供关联.

VFS 管理通过集成开发环境访问的文件的持久快照. 这些快照只存储那些通过 VFS API 请求过至少一次的文件. 关于 VFS 的性质, 以下是一些需要注意的重要事项:

* 缓存的内容是异步刷新的, 以匹配磁盘上的任何变化.
* 请记住, 快照是存储在应用程序级别的, 正如你所期望的那样, 因此在多个项目中打开的文件将只有一个快照.
* 快照在刷新操作过程中从磁盘更新, 刷新操作通常是异步进行的. 通过 VFS 进行的所有写入操作都是同步的, 内容会立即保存到磁盘上.

> 从[官方文档](https://plugins.jetbrains.com/docs/intellij/virtual-file-system.html?from=jetbrains.org) 阅读有关 VFS 的更多信息.

使用 VFS 处理插件所需的文件时, 会遇到很多常见情况. 以下是其中一些情况, 并附有代码示例, 可帮助你处理这些情况.

#### 获取项目中所有虚拟文件的列表

按名称`Lambdas.kt`获取虚拟文件列表的代码段.

```
fun getListOfProjectVirtualFilesByName(project: Project,
                                     caseSensitivity: Boolean = true,
                                     fileName: String = "Lambdas.kt"
): MutableCollection<VirtualFile> {
val scope = GlobalSearchScope.projectScope(project)
return FilenameIndex.getVirtualFilesByName(
    project, fileName, caseSensitivity, scope)
}
```

获取扩展名为`kt`的虚拟文件列表的代码段.

```
fun getListOfProjectVirtualFilesByExt(project: Project,
                                    caseSensitivity: Boolean = true,
                                    extName: String = "kt"
): MutableCollection<VirtualFile> {
val scope = GlobalSearchScope.projectScope(project)
return FilenameIndex.getAllFilesByExt(project, extName, scope)
}
```

获取项目中所有虚拟文件列表的代码段.

```
fun getListOfAllProjectVFiles(project: Project): MutableCollection<VirtualFile> {
  val collection = mutableListOf<VirtualFile>()
  ProjectFileIndex.getInstance(project).iterateContent {
    collection += it
    // Return true to process all the files (no early escape).
    true
  }
  return collection
}
```

#### 以编程方式附加监听器以查看虚拟文件的变化

你可以通过编程或声明的方式附加监听器.

这是以编程方式附加VFS变化监听器的过时方法:

```VirtualFileManager.getInstance().addVirtualFileListener()```

以编程方式添加监听器的新方法是监听总线(又称主题)上的`VirtualFileManager.VFS_CHANGES`事件.

> 由于无法按路径或文件名过滤这些更改事件, 因此需要在监听器中加入过滤不需要的事件的逻辑.

下面的函数展示了如何在代码中进行注册. 请注意, 该函数在 EDT 中运行.

```
/**
 * VFS listeners are application level and will receive events for changes happening in all the projects opened by the
 * user. You may need to filter out events that aren’t relevant to your task (e.g., via
 * `ProjectFileIndex#isInContent()`). A listener for VFS events, invoked inside write-action.
 */
private fun attachListenerForProjectVFileChanges(): Unit {
  println("MyPlugin: attachListenerForProjectFileChanges()")

  val connection = project.messageBus.connect(/*parentDisposable=*/ project)
  connection.subscribe(
      VirtualFileManager.VFS_CHANGES,
      object : BulkFileListener {
        override fun after(events: List<VFileEvent>) = doAfter(events)
      })
}

fun handleEvent(event: VFileEvent) {
  when (event) {
    is VFilePropertyChangeEvent -> {
      println("VFile property change event: $event")
    }
    is VFileContentChangeEvent  -> {
      println("VFile content change event: $event")
    }
  }
}

fun doAfter(events: List<VFileEvent>) {
  println("VFS_CHANGES: #events: ${events.size}")
  val projectFileIndex = ProjectRootManager.getInstance(project).fileIndex
  events.withIndex().forEach { (index, event) ->
    println("$index. VFile event: $event")
    // Filter out file events that are not in the project's content.
    events
        .filter { it.file != null && projectFileIndex.isInContent(it.file!!) }
        .forEach { handleEvent(it) }
  }
}
```
使用这种方法, 必须在某个时刻运行附加监听器本身的代码. 因此, 如果这是一个应在项目打开时运行的监听器, 则可能需要添加一个`postStartupActivity`, 这只是一个由插件提供的类, 将在 IDE 打开项目后运行. 请在动态插件部分阅读所有相关内容.

> 💡 如果你希望引用当前打开的项目, 你可能希望使用这种方法, 而不是下图所示的声明式方法.

#### 以声明方式附加监听器查看虚拟文件的更改

你也可以在`plugin.xml`中声明式地注册监听器. 使用这种方法与上述代码方法有一些不同之处. 你可以在`plugin.xml`中为特定事件类注册一个监听器, 而不是订阅一个主题.

下面的代码段与上面的代码类似. 因此, `VirtualFileManager.VFS_CHANGES`主题相当于`com.intellij.openapi.vfs.newvfs.BulkFileListener`类.

```
<applicationListeners>
  <listener class="MyListener" topic="com.intellij.openapi.vfs.newvfs.BulkFileListener"/>
</applicationListeners>
```

代码在这里.

```
class MyVfsListener : BulkFileListener {
  @Override
  fun after(@NotNull events: List<VFileEvent?>?) {
    // handle the events
  }
}
```

> [官方文档](https://plugins.jetbrains.com/docs/intellij/plugin-listeners.html?from=jetbrains.org#defining-application-level-listeners) 中有关注册 VFS 监听器的更多信息.

> 💡 也可以声明地附加项目级的监听器. 不过, 这要求你将在 XML 中注册的接口/类可以将项目对象作为参数. 就 VFS 监听器而言, 它不接受项目参数, 因为 VFS 操作是应用级的. 因此, 如果要获取当前打开的项目, 就必须使用编程方法.

#### 异步处理文件系统事件

可以异步(在后台线程中)获取这些文件系统事件. 请查看[`AsyncFileListener.java`](https://upsource.jetbrains.com/idea-ce/file/idea-ce-7cf49a16e4e15a18fa2f742635053647ae94abfb/platform/core-api/src/com/intellij/openapi/vfs/AsyncFileListener.java)类中的示例, 了解如何做到这一点. 异步版本不像在UI线程上运行的版本那样容易实现. 下面是一些相关说明:

1. 你可以为`vfs.asyncListener`注册一个扩展, 在`plugin.xml`中注册你的异步监听器.
2. 或者可以调用[`VirtualFileManager.java`](https://upsource.jetbrains.com/idea-ce/file/idea-ce-7cf49a16e4e15a18fa2f742635053647ae94abfb/platform/core-api/src/com/intellij/openapi/vfs/VirtualFileManager.java)的`addVirtualFileListener()`方法.

#### 在保存当前打开的文件时进行拦截

这是一个代码片段, 可使用`AppTopics.FILE_DOCUMENT_SYNC`主题在保存文件前获取事件. 请阅读有关[Document](https://plugins.jetbrains.com/docs/intellij/documents.html?from=jetbrains.org#how-do-i-get-notified-when-documents-change) 的更多信息, 以了解此事件的作用和来源.

> 请注意, 该监听器将触发任何正在保存的文件, 而不仅仅是当前正在编辑的文件.
> * 因此, 如果你要依靠它来触发当前打开文件的 JUST, 那就不能使用这个方法.
> * 然而, 如果你想在编辑器中打开的文件被保存之前做一些事情, 那么这里就是捕获这些事件的地方.

```
/**
 * - [Tutorial](http://arhipov.blogspot.com/2011/04/code-snippet-intercepting-on-save.html)
 * - [FileDocumentManagerListener]
 * - [AppTopics]
 */
private fun attachFileSaveListener() {
  printHeader()
  val connection = project.messageBus.connect(/*parentDisposable=*/ project)
  connection.subscribe(
      AppTopics.FILE_DOCUMENT_SYNC,
      object : FileDocumentManagerListener {
        override fun beforeDocumentSaving(document: Document) {
          val vFile = FileDocumentManager.getInstance().getFile(document)
          println(StringBuilder().apply {
              append("A VirtualFile is about to be saved\n")
              append("\tvFile: $vFile\n")
              append("\tdocument: $document\n")
          })
        }
      })
}
```

### Document

Document API 是一种将文件作为简单文本文件从IDE中访问的方法. 在加载或保存文件时, IntelliJ 平台会透明地处理编码和换行. 有几种获取`Document`对象的方法.

* 使用`e.getRequiredData(CommonDataKeys.EDITOR).document`从操作中获取.
* 使用`FileDocumentManager.document`从虚拟文件获取.
* 使用`PsiDocumentManager.getInstance().document`从 PSI 文件获取.

#### 使用Document API 的操作示例

下面是一个使用 Document API 将 IDE 中的某些选定文本替换为另一个字符串的操作示例.

```
class EditorReplaceTextAction : AnAction() {
  /**
   * [Tutorial](https://www.jetbrains.org/intellij/sdk/docs/tutorials/editor_basics/working_with_text.html)
   */
  override fun actionPerformed(e: AnActionEvent) {
    val editor: Editor = e.getRequiredData(CommonDataKeys.EDITOR)
    val project: Project = e.getRequiredData(CommonDataKeys.PROJECT)
    val document: Document = editor.document
    val caretModel: CaretModel = editor.caretModel
    val primaryCaret = caretModel.primaryCaret

    // start and end offsets of the selected text (based on the primaryCaret).
    val selection =
        Pair<Int, Int>(primaryCaret.selectionStart, primaryCaret.selectionEnd)

    // Actual content that is selected at the caret.
    val selectedText: String = primaryCaret.selectedText!!

    // Change the document in a write action in a command (for undo).
    WriteCommandAction.runWriteCommandAction(project) {
      document.replaceString(selection.first, selection.second, ">> $selectedText <<")
    }

    // Deselect the selection of the text that that was just replaced.
    primaryCaret.removeSelection()
  }

  override fun update(e: AnActionEvent) {
    val project: Project? = e.project
    val editor: Editor? = e.getData(CommonDataKeys.EDITOR)
    // Action visible only if the editor in the open project has text selected.
    e.presentation.isEnabledAndVisible =
        project != null
        && editor != null
        && editor.selectionModel.hasSelection()
  }
}
```

> 要了解更多关于Document的信息, 请查看[官方文档](https://plugins.jetbrains.com/docs/intellij/documents.html?from=jetbrains.org#how-do-i-get-notified-when-documents-change).
