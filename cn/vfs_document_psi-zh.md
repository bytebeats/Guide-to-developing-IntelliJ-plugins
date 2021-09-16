## VFS, Document, PSI

The virtual file system (VFS) is an abstraction that is available inside the IntelliJ Platform that allows access to files on your computer (on your local file system, or even in JAR files), or from a source code repository, or over the network. There are multiple ways of accessing the contents of files in the IDE:

1. VFS allows access to files at the lowest level (closest the actual file system and not an abstraction like PSI).
2. Document provides an object model to access file contents as plain text (so its somewhere between VFS and PSI).
3. PSI (Program Structure Interface) allows access to the contents of a file in a hierarchical object model which takes the syntax and semantics of specific languages into account (kind of like how DOM represents HTML and CSS content in a web browser). You can learn more about PSI in in this section.

### VFS

The VFS is what the IntelliJ Platform uses to work with files. It provides:

1. A universal API for working with files regardless of where they are located (on disk, in a JAR file, on a HTTP server, in a VCS, etc).
2. Information for tracking file modifications and providing both new and old versions of a file when a change is detected in a file.
3. Ability to associate additional persistent data with a file in the VFS.

The VFS manages a persistent snapshot of the files that are accessed via the IDE. These snapshots store only those files which have been requested at least once via the VFS API. Here are some important things to keep in mind about the nature of VFS:

* The contents of the cache is refreshed asynchronously to match any changes on disk.
* Keep in mind that the snapshot is stored at the application level, as you might expect, so a file that is open in multiple projects will only have one snapshot.
* The snapshot is updated from disk during refresh operations, which generally happen asynchronously. All write operations made through the VFS are synchronous and the contents is saved to disk immediately.

> Read more about VFS from the [official docs](https://plugins.jetbrains.com/docs/intellij/virtual-file-system.html?from=jetbrains.org).

There are quite a few common scenarios that you face when using VFS to work with files that your plugin needs. The following are some of these scenarios with code samples to help you deal with them.

#### Getting a list of all the virtual files in a project

Snippet to get a list of virtual files by name `Lambdas.kt`.

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

Snippet to get a list of virtual files with extension `kt`.

```
fun getListOfProjectVirtualFilesByExt(project: Project,
                                    caseSensitivity: Boolean = true,
                                    extName: String = "kt"
): MutableCollection<VirtualFile> {
val scope = GlobalSearchScope.projectScope(project)
return FilenameIndex.getAllFilesByExt(project, extName, scope)
}
```

Snippet to get a list of all virtual files in a project.

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

#### Attach listeners to see changes to virtual files programmatically

You can attach the listeners programmatically or declaratively.

This is the deprecated way of programmatically attaching a listener for VFS changes:

```VirtualFileManager.getInstance().addVirtualFileListener()```

The new way to add a listener programmatically is to listen to `VirtualFileManager.VFS_CHANGES` events on the bus (aka the topic).

> There is no way to filter for these change events by path or filename, so the logic to filer out unwanted events needs to go in the listener.

The following function shows how to register this in code. Note that this function runs in the EDT.

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

Using this approach, the code attaching the listener itself has to be run at some point. So if this is a listener that should run when your project is opened, then you might need to add a `postStartupActivity`, which is just a class supplied by your plugin that will be run after the IDE opens a project. Please read all about this in the dynamic plugins section.

> 💡 You might want to use this approach over the declarative approach shown below in case you want a reference to the currently open project.

#### Attach listeners to see changes to virtual files declaratively

You can also register a listener declaratively in your `plugin.xml`. There are some differences to using this approach over the code approach shown above. Instead of subscribing to a topic, you can simply register a listener for a specific event class in your plugin.xml.

Here’s a snippet that does something similar to the code above. So the `VirtualFileManager.VFS_CHANGES` topic is equivalent to the `com.intellij.openapi.vfs.newvfs.BulkFileListener` class.

```
<applicationListeners>
  <listener class="MyListener" topic="com.intellij.openapi.vfs.newvfs.BulkFileListener"/>
</applicationListeners>
```

Here’s the code.

```
class MyVfsListener : BulkFileListener {
  @Override
  fun after(@NotNull events: List<VFileEvent?>?) {
    // handle the events
  }
}
```

> Here’s more information on registering VFS listeners in the [official docs](https://plugins.jetbrains.com/docs/intellij/plugin-listeners.html?from=jetbrains.org#defining-application-level-listeners).


> 💡 You can also declaratively attach listeners that are scoped to a project. However, this requires that the interface / class that you will register in the XML can take a project object as a parameter. In the case of VFS listeners it does not take a project parameter, since VFS operations are application level. So if you want to get a hold of the currently open project, then you have to use the programmatic approach.

#### Asynchronously process file system events

It is possible to get these file system events asynchronously (in a background thread). Take a Look at the [`AsyncFileListener.java`](https://upsource.jetbrains.com/idea-ce/file/idea-ce-7cf49a16e4e15a18fa2f742635053647ae94abfb/platform/core-api/src/com/intellij/openapi/vfs/AsyncFileListener.java) class for examples on how to do this. The async versions are not as trivial to implement as the version that runs on the UI thread. Here are some notes on this:

1. You can register an extension for `vfs.asyncListener` that registers your async listener in `plugin.xml`.
2. Or you can call [`VirtualFileManager.java`](https://upsource.jetbrains.com/idea-ce/file/idea-ce-7cf49a16e4e15a18fa2f742635053647ae94abfb/platform/core-api/src/com/intellij/openapi/vfs/VirtualFileManager.java)’s `addVirtualFileListener()` method.

#### Intercept when the currently open file gets saved

This is a code snippet that can use the `AppTopics.FILE_DOCUMENT_SYNC` topic to get an event just before a file is saved. Read more about [Document](https://plugins.jetbrains.com/docs/intellij/documents.html?from=jetbrains.org#how-do-i-get-notified-when-documents-change) to get an understanding of what this event does and where it comes from.

> Note that this listener will fire on any file that is being saved, not just the only one that is currently being edited.
> * So if you’re relying on this to fire JUST for the file that is currently open, then this is not the way to go.
> * However, if you want to do something before any files that are open in editors are going to be saved, then this is the place to trap these events.

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

The Document API is a way to access a file from the IDE as a simple text file. The IntelliJ Platform handles encoding and line break conversions when loading or saving the file transparently. There are a few ways of getting a `Document` object.

* From an action using `e.getRequiredData(CommonDataKeys.EDITOR).document`.
* From a virtual file using `FileDocumentManager.document`.
* From a PSI file using `PsiDocumentManager.getInstance().document`.

#### Example of an action that uses the Document AP

Here’s an example of an action that uses the Document API to replace some selected text in the IDE with another string.

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

> Read more about Documents from the [official docs](https://plugins.jetbrains.com/docs/intellij/documents.html?from=jetbrains.org#how-do-i-get-notified-when-documents-change).
