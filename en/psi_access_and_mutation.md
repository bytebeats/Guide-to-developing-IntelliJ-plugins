## PSI access and mutation

This section covers Program Structure Interface (PSI) and how to use it to access files in various languages. And how to use it to create language specific content. Threading considerations that must be taken into account to make sure the IDE UI is responsive is also covered here.

> Please read the [official docs on PSI](https://plugins.jetbrains.com/docs/intellij/psi.html) before proceeding further.

### How to create a PSI File or get a reference to one

There are many ways of creating a PSI file. Here are some examples of you can get a reference to a PSI file in your plugin.

1.  From an action’s event: `e.getData(LangDataKeys.PSI_FILE)`
2.  From a `VirtualFile`: `PsiManager.getInstance(myProject).findFile(vFile)`
3.  From a `Document`: `PsiDocumentManager.getInstance(project).getPsiFile(document)`
4.  From a `PsiElement`: `psiElement.getContainingFile()`
5.  From any project you can get a `PSIFile[]`: `FilenameIndex .getFilesByName(project, "file.ext", GlobalSearchScope.allScope(project))`

There is some information on navigating PSI trees [here](https://plugins.jetbrains.com/docs/intellij/navigating-psi.html?from=jetbrains.org).

### PSI Access

#### Visitor pattern for top down navigation (without threading considerations)

A very common thing to do when you have a reference to a PSI file is to navigate it from the root node to find something inside of the file. This is where the visitor pattern comes into play.

1.  In order to get access to all the classes you might need for this, you might need to import the right set of [built in modules](https://plugins.jetbrains.com/docs/intellij/plugin-compatibility.html?from=jetbrains.org#modules-specific-to-functionality).
> here are scenarios where you might want to add functionality in the IJ platform itself. For example, let’s say you wanted to add JavaRecursiveElementVisitor to your project to visit a PSI tree. Well, this class isn’t available to the plugin by default, and needs to be “imported” in 2 steps, just as you would for a published plugin.
> 1.  `setPlugins()` in `build.gradle.kts`, and
> 2.  `<depends>` in `plugin.xml`.
> Here’s what you have to do to add support for the Java language.
> 1.  `build.gradle.kts`: `setPlugins("java", <other plugins, if they exist>)`

2.  In many cases, you can also use more specific APIs for top-down navigation. For example, if you need to get a list of all methods in a Java class, you can do that using a visitor, but a much easier way to do that is to call `PsiClass.getMethods()`.
3.  [`PsiTreeUtil.java`](https://github.com/JetBrains/intellij-community/blob/master/platform/core-api/src/com/intellij/psi/util/PsiTreeUtil.java) contains a number of general-purpose, language-independent functions for PSI tree navigation, some of which (for example, `findChildrenOfType()`) perform top-down navigation.
> ⚠️ [Threading issues](https://plugins.jetbrains.com/docs/intellij/performance.html?from=jetbrains.org#avoiding-ui-freezes) might arise, since all this computation is done in the EDT, and for a really long PSI tree, this can problematic. Especially w/ the use of a write lock to make some big changes in the tree. There are many sections below which deal w/ cancellation and read and write locking.

The basic visitor pattern for top down navigation looks like this. The following snippet counts the number of header and paragraph elements in a `Markdown` file, where `psiFile` is a reference to this file.

```
val count = object {
    var paragraph: Int = 0
    var header: Int = 0
}
psiFile.accept(object : MarkdownRecursiveElementVisitor() {
  override fun visitParagraph(paragraph: MarkdownParagraphImpl) {
    count.paragraph++
    // The following line ensures that ProgressManager.checkCancelled()
    // is called.
    super.visitParagraph(paragraph)
  }
  override fun visitHeader(header: MarkdownHeaderImpl) {
    count.header++
    // The following line ensures that ProgressManager.checkCancelled()
    // is called.
    super.visitHeader(header)
  }
})
```

This is great sample code to navigate the tree for a Java class: [PsiNavigationDemoAction.java](https://github.com/JetBrains/intellij-sdk-docs/blob/main/code_samples/psi_demo/src/main/java/org/intellij/sdk/psi/PsiNavigationDemoAction.java).

#### Threading, locking, and progress cancellation check during PSI access

When creating long running actions that work on a PSI tree, you have to take care of the following things:

1.  Make sure that you are accessing the PSI tree after acquiring a read lock (in a read action).
2.  Make sure that your long running tasks can be cancelled by the user.
3.  Make sure that you don’t freeze the UI and make it unresponsive for a long running task that uses the PSI tree.

Here is an example of running a task in a background thread in IDEA that uses a read action, but is NOT cancellable. Better ways of doing this are shown below.

```
ApplicationManager.getApplication().executeOnPooledThread {
  ApplicationManager.getApplication().runReadAction {
    // Your potentially long running task that uses something in the PSI tree.
  }
}
```

#### Background tasks and multiple threads - The incorrect way

The following is a bad example of using `Task.Backgroundable` object. It is cancellable, and it works, but it accesses some PSI structures inside of this task (in a background thread) which is actually a bad thing that leads to race conditions w/ the data in the EDT and the data that the background thead is currently getting from the PSI objects.

Here’s the code for an action that creates the task and runs it in the background (without acquiring a read lock in a read action):

```
override fun actionPerformed(e: AnActionEvent) {
  val psiFile = e.getRequiredData(CommonDataKeys.PSI_FILE)
  val psiFileViewProvider = psiFile.viewProvider
  val project = e.getRequiredData(CommonDataKeys.PROJECT)
  val progressTitle = "Doing heavy PSI computation"

  val task = object : Backgroundable(project, progressTitle) {
    override fun run(indicator: ProgressIndicator) {
      doWorkInBackground(project, psiFile, psiFileViewProvider, indicator)
    }
  }
  task.queue()
}
```

Here’s the code for the `doWorkInBackground(...)`. As you can see this function does not acquire a read lock, and simply runs the `navigateXXXTree()` functions based on whether the current file has `Markdown` or `Java` code in it.

```
private data class Count(var paragraph: Int = 0, var links: Int = 0, var header: Int = 0)

private val count = Count()

private fun doWorkInBackground(project: Project,
                               psiFile: PsiFile,
                               psiFileViewProvider: FileViewProvider,
                               indicator: ProgressIndicator,
                               editor: Editor
) {
  indicator.isIndeterminate = true
  val languages = psiFileViewProvider.languages
  val message = buildString {
    when {
      languages.contains("Markdown") -> navigateMarkdownTree(psiFile, indicator, project)
      languages.contains("Java")     -> navigateJavaTree(psiFile, indicator, project, editor)
      else                           -> append("No supported languages found")
    }
    append("languages: $languages\n")
    append("count.header: ${count.header}\n")
    append("count.paragraph: ${count.paragraph}\n")
    checkCancelled(indicator, project)
  }
  println(message)
}
```

Let’s look at the `navigateXXXTree()` functions next. They simply generate some analytics depending on whether a Java or Markdown file is open in the editor.

1.  For Markdown files, it simply generates the number of headers and paragraphs that exist in the document in the editor.
2.  For Java files, if a method is selected, it generates information about the enclosing class and any local variables that are declared inside that method.

For Markdown files, here’s the function.

```
private fun navigateMarkdownTree(psiFile: PsiFile,
                                 indicator: ProgressIndicator,
                                 project: Project
) {
  psiFile.accept(object : MarkdownRecursiveElementVisitor() {
    override fun visitParagraph(paragraph: MarkdownParagraphImpl) {
      this.count.paragraph++
      checkCancelled(indicator, project)
      // The following line ensures that ProgressManager.checkCancelled() is called.
      super.visitParagraph(paragraph)
    }
    override fun visitHeader(header: MarkdownHeaderImpl) {
      this.count.header++
      checkCancelled(indicator, project)
      // The following line ensures that ProgressManager.checkCancelled()  is called.
      super.visitHeader(header)
    }
  })
}
```

For Java files, here’s the function.

```
private fun navigateJavaTree(psiFile: PsiFile,
                             indicator: ProgressIndicator,
                             project: Project,
                             editor: Editor
) {
  val offset = editor.caretModel.offset
  val element: PsiElement? = psiFile.findElementAt(offset)
  val javaPsiInfo = buildString {
    checkCancelled(indicator, project)
    element?.apply {
      append("Element at caret: $element\n")
      val containingMethod: PsiMethod? = PsiTreeUtil.getParentOfType(element, PsiMethod::class.java)
      containingMethod?.apply {
        append("Containing method: ${containingMethod.name}\n")
        containingMethod.containingClass?.apply {
          append("Containing class: ${this.name} \n")
        }
        val list = mutableListOf<PsiLocalVariable>()
        containingMethod.accept(object : JavaRecursiveElementVisitor() {
          override fun visitLocalVariable(variable: PsiLocalVariable) {
            list.add(variable)
            // The following line ensures that ProgressManager.checkCancelled() is called.
            super.visitLocalVariable(variable)
          }
        })
        if (list.isNotEmpty())
          append(list.joinToString(prefix = "Local variables:\n", separator = "\n") { it -> "- ${it.name}" })
      }
    }
  }
  checkCancelled(indicator, project)
  val message = if (javaPsiInfo == "") "No PsiElement at caret!" else javaPsiInfo
  println(message)
  ApplicationManager.getApplication().invokeLater {
    Messages.showMessageDialog(project, message, "PSI Java Info", null)
  }
}
```

Now, when you run this code, the `navigateMarkdownTree()` does not throw an exception because the PSI was accessed outside of a read lock. However, `navigateJavaTree()` does throw the exception below.

```
java.lang.Throwable: Read access is allowed from event dispatch thread or inside
read-action only (see com.intellij.openapi.application.Application.runReadAction())
```

This exception will most likely get thrown for any complicated access of the PSI from a non-EDT thread without a read lock.

> Note that if you access the PSI data from the EDT itself there is no need to acquire a read lock.

Notes on `navigateMarkdownTree()`:

* The code above doesn’t trigger any read lock assertion exceptions, and it is probably because it is too simple. There are a few flavors of `assertReadAccessAllowed()`, one even in `PsiFileImpl`, and they are called from various methods accessing the PSI. Maybe they’re not called everywhere (I assume it’d be expensive to check read access for everything), just in the common APIs, and maybe this example didn’t get to any.
* Also, it’s probably possible to read state without a read lock, i.e the IDE won’t necessarily throw an exception, it’s just that you’re not guaranteed any consistent results as you’re racing with the UI thread (making changes to the data within write actions).
* So just because an exception isn’t throw when failing to acquire a read lock to access PSI data doesn’t mean the results are consistent due to possibly racing w/ the UI thread making changes to that data within write actions.
* Here are the official JetBrains [docs](https://plugins.jetbrains.com/docs/intellij/general-threading-rules.html?from=jetbrains.org) on threading and locking.

Notes on `navigateJavaTree()`:
* This code throws the exception right away, which is why it is important to wrap all PSI read access inside of a read action.

#### Background tasks and multiple threads - The correct way

These are the steps we can take to fix the code above.
1.  We don’t have to deal w/ read locks in the implementation of these two functions (`navigateJavaTree()` and `navigateMarkdownTree()`). The `doWorkInBackground()` function should really be dealing with this.
2.  Wrapping the code in `doWorkInBackground()` that calls these two functions in a read lock by using `runReadAction {}` eliminates these exceptions.

> Note that both functions `navigateMarkdownTree()` or `navigateJavaTree()` do not need to be modified in any way!

```
private fun doWorkInBackground(project: Project,
                               psiFile: PsiFile,
                               psiFileViewProvider: FileViewProvider,
                               indicator: ProgressIndicator,
                               editor: Editor
) {
  indicator.isIndeterminate = true
  val languages = psiFileViewProvider.languages
  buildString {
    when {
      languages.contains("Markdown") -> runReadAction { navigateMarkdownTree(psiFile, indicator, project) }
      languages.contains("Java")     -> runReadAction { navigateJavaTree(psiFile, indicator, project, editor) }
      else                           -> append("No supported languages found")
    }
    append("languages: $languages\n")
    append("count.header: ${count.header}\n")
    append("count.paragraph: ${count.paragraph}\n")
    checkCancelled(indicator, project)
  }.printlnAndLog()
}
```

For any long running background task, we have to check for cancellation to ensure that unnecessary work does nto get done after the user has requested that our long running background task be cancelled.

For tasks that require a lot of virtual files, or PSI elements to be looped over it becomes necessary to check for cancellation (in addition to acquiring a read lock). Here’s an example from `MarkdownRecursiveElementVisitor` which can be overridden to walk the PSI tree.

```
open class MarkdownRecursiveElementVisitor : MarkdownElementVisitor(), PsiRecursiveVisitor {
  override fun visitElement(element: PsiElement) {
    ProgressManager.checkCanceled()
    element.acceptChildren(this)
  }
}
```

Note about the call to `ProgressManager.checkCanceled()` above - A subclass would have to make sure to call `super.visitElement(...)` to ensure that this cancellation check is actually made.

> #### Dispatching UI events during checkCanceled().
> There is a mechanism for updating the UI during write actions, but it seems quite limited.
> * A `ProgressIndicator` may choose to implement the `PingProgress` interface. If it does, then `PingProgress.interact()` will be called whenever `checkCanceled()` is called. For details see `ProgressManagerImpl.executeProcessUnderProgress()` and `ProgressManagerImpl.addCheckCanceledHook()`.
> * The `PingProgress` mechanism is used by `PotemkinProgress`: “A progress indicator for write actions. Paints itself explicitly, without resorting to normal Swing’s delayed repaint API. Doesn’t dispatch Swing events, except for handling manually those that can cancel it or affect the visual presentation.” I.e., `PotemkinProgress` dispatches certain UI events during a write action to ensure that the progress dialog remains responsive. But, note the comment, “Repaint just the dialog panel. We must not call custom paint methods during write action, because they might access the model, which might be inconsistent at that moment.”

> #### Don’t try to acquire read or write locks directly, use actions instead
> Instead of trying to acquire read or write locks directly, Use lambdas and read or write actions instead. For example: `runReadAction()`, `runWriteAction()`, `WriteCommandAction#runWriteCommandAction()`, etc.
> The APIs for acquiring the read or write lock are deprecated and marked for removal in 2020.3. It is good they’re being deprecated, because if you think about offering nonblocking read action semantics (from the platform perspective), if a “read” actions is done via a lock acquire / release, how can one interrupt and re-start it?
> 1.  [`Application.java#acquireReadActionLock()`](https://github.com/JetBrains/intellij-community/blob/2b40e2ffe3cdda51990979b81567764965d890ed/platform/core-api/src/com/intellij/openapi/application/Application.java#L430)
> 2.  [`Application.java#acquireWriteActionLock()`](https://github.com/JetBrains/intellij-community/blob/2b40e2ffe3cdda51990979b81567764965d890ed/platform/core-api/src/com/intellij/openapi/application/Application.java#L438)

### PSI Mutation
#### PsiViewer plugin

One of the most important things to do before modifying the PSI is to understand its structure. And the best way to do this is to install the [PsiViewer plugin](https://plugins.jetbrains.com/plugin/227-psiviewer) and use it to study what the PSI tree looks like.

In this example, we will be modifying a Markdown document. So we will use this plugin to examine a Markdown file that’s open in an IDE editor window. Here are the options that you should enable for the plugin while you’re browsing around the editor:

1.  Open Settings -> PSIViewer and change the highlight colors to something that you like and make sure to set the alpha to 255 for both references and highlighting.
2.  Open the PSIViewer tool window and enable the following options in the toolbar:
    * Enable highlight (you might want to disable this when you’re done playing around w/ the tool window)
    * Enable properties
    * Enable scroll to source
    * Enable scroll from source

A great example that can help us understand how PSI modification can work is taking a look at the built-in Markdown plugin actions themselves. There are a few actions in the toolbar of every Markdown editor: “toggle bold”, “toggle italic”, etc.

These are great to walk thru to understand how to make our own. The source files from intellij-community repo on github are:

* Files in [`plugins/markdown/src/org/intellij/plugins/markdown/ui/actions/styling/`](https://github.com/JetBrains/intellij-community/tree/master/plugins/markdown/src/org/intellij/plugins/markdown/ui/actions/styling)
* [`ToggleBoldAction.java`](https://github.com/JetBrains/intellij-community/blob/master/plugins/markdown/src/org/intellij/plugins/markdown/ui/actions/styling/ToggleBoldAction.java)
* [`BaseToggleStateAction.java`](https://github.com/JetBrains/intellij-community/blob/master/plugins/markdown/src/org/intellij/plugins/markdown/ui/actions/styling/BaseToggleStateAction.java)

The example we will build for this section entails finding hyperlinks and replacing the links w/ some modified version of the link string. This will require using a write lock to mutate the PSI in a cancellable action.

#### Generate PSI elements from text

It might seem strange but the preferred way to [create PSI elements](https://plugins.jetbrains.com/docs/intellij/modifying-psi.html?from=jetbrains.org) by generating text for the new element and then having IDEA parse this into a PSI element. Kind of like how a browser parses text (containing HTML) into DOM elements by setting `innerHTML()`.

Here is some code that does this for Markdown elements.

```
private fun createNewLinkElement(project: Project, linkText: String, linkDestination: String): PsiElement? {
  val markdownText = "[$linkText]($linkDestination)"
  val newFile = MarkdownPsiElementFactory.createFile(project, markdownText)
  val newParentLinkElement = findChildElement(newFile, MarkdownTokenTypeSets.LINKS)
  return newParentLinkElement
}
```

#### Example of walking up and down PSI trees to find elements

This is a dump from the PSI viewer of a snippet from this [README.md](https://raw.githubusercontent.com/sonar-intellij-plugin/sonar-intellij-plugin/master/README.md) file.

```
MarkdownParagraphImpl(Markdown:PARAGRAPH)(1201,1498)
  PsiElement(Markdown:Markdown:TEXT)('The main goal of this plugin is to show')(1201,1240)
  PsiElement(Markdown:WHITE_SPACE)(' ')(1240,1241)
  ASTWrapperPsiElement(Markdown:Markdown:INLINE_LINK)(1241,1274)  <============[🔥 WE WANT THIS PARENT 🔥]=========
    ASTWrapperPsiElement(Markdown:Markdown:LINK_TEXT)(1241,1252)
      PsiElement(Markdown:Markdown:[)('[')(1241,1242)
      PsiElement(Markdown:Markdown:TEXT)('SonarQube')(1242,1251)  <============[🔥 EDITOR CARET IS HERE 🔥]========
      PsiElement(Markdown:Markdown:])(']')(1251,1252)
    PsiElement(Markdown:Markdown:()('(')(1252,1253)
    MarkdownLinkDestinationImpl(Markdown:Markdown:LINK_DESTINATION)(1253,1273)
      PsiElement(Markdown:Markdown:GFM_AUTOLINK)('http://sonarqube.org')(1253,1273)
    PsiElement(Markdown:Markdown:))(')')(1273,1274)
  PsiElement(Markdown:WHITE_SPACE)(' ')(1274,1275)
  PsiElement(Markdown:Markdown:TEXT)('issues directly within your IntelliJ IDE.')(1275,1316)
  PsiElement(Markdown:Markdown:EOL)('\n')(1316,1317)
  PsiElement(Markdown:Markdown:TEXT)('Currently the plugin is build to work in IntelliJ IDEA,')(1317,1372)
  PsiElement(Markdown:WHITE_SPACE)(' ')(1372,1373)
  PsiElement(Markdown:Markdown:TEXT)('RubyMine,')(1373,1382)
  PsiElement(Markdown:WHITE_SPACE)(' ')(1382,1383)
  PsiElement(Markdown:Markdown:TEXT)('WebStorm,')(1383,1392)
  PsiElement(Markdown:WHITE_SPACE)(' ')(1392,1393)
  PsiElement(Markdown:Markdown:TEXT)('PhpStorm,')(1393,1402)
  PsiElement(Markdown:WHITE_SPACE)(' ')(1402,1403)
  PsiElement(Markdown:Markdown:TEXT)('PyCharm,')(1403,1411)
  PsiElement(Markdown:WHITE_SPACE)(' ')(1411,1412)
  PsiElement(Markdown:Markdown:TEXT)('AppCode and Android Studio with any programming ... SonarQube.')(1412,1498)
PsiElement(Markdown:Markdown:EOL)('\n')(1498,1499)
```

There are a few things to note from this tree.
1. The caret in the editor selects a PSI element that is at the leaf level of the selection.
2. This will require us to walk up the tree (navigate to the parents, and their parents, and so on). We have to use a `MarkdownTokenTypes` (singular) or `MarkdownTokenTypeSets` (a set).
* An example is that we start w/ a `TEXT`, then move up to `LINK_TEXT`, then move up to `INLINE_LINK`.
1. This will require us to walk down the tree (navigate to the children, and their children, and so on). Similarly, we can use the same token types or token type sets above.
* An example is that we start w/ a `INLINE_LINK` and drill down the kids, then move down to the `LINK_DESTINATION`.

Here’s the code that does exactly this. And it stores the result in a `LinkInfo` data class object.

```
data class LinkInfo(var parentLinkElement: PsiElement, var linkText: String, var linkDestination: String)

/**
 * This function tries to find the first element which is a link, by walking up the tree starting w/ the element that
 * is currently under the caret.
 *
 * To simplify, something like `PsiUtilCore.getElementType(element) == INLINE_LINK` is evaluated for each element
 * starting from the element under the caret, then visiting its parents, and their parents, etc, until a node of type
 * `INLINE_LINK` is found, actually, a type contained in [MarkdownTokenTypeSets.LINKS].
 */
private fun findLink(editor: Editor, project: Project, psiFile: PsiFile): LinkInfo? {
  val offset = editor.caretModel.offset
  val elementAtCaret: PsiElement? = psiFile.findElementAt(offset)

  // Find the first parent of the element at the caret, which is a link.
  val parentLinkElement = findParentElement(elementAtCaret, MarkdownTokenTypeSets.LINKS)

  val linkTextElement = findChildElement(parentLinkElement, MarkdownTokenTypeSets.LINK_TEXT)
  val textElement = findChildElement(linkTextElement, MarkdownTokenTypes.TEXT)
  val linkDestinationElement = findChildElement(parentLinkElement, MarkdownTokenTypeSets.LINK_DESTINATION)

  val linkText = textElement?.text
  val linkDestination = linkDestinationElement?.text

  if (linkText == null || linkDestination == null || parentLinkElement == null) return null

  println("Top level element of type contained in MarkdownTokenTypeSets.LINKS found! 🎉")
  println("linkText: $linkText, linkDest: $linkDestination")
  return LinkInfo(parentLinkElement, linkText, linkDestination)
}
```

#### Finding elements up the tree (parents)

```
private fun findParentElement(element: PsiElement?, tokenSet: TokenSet): PsiElement? {
  if (element == null) return null
  return PsiTreeUtil.findFirstParent(element, false) {
    callCheckCancelled()
    val node = it.node
    node != null && tokenSet.contains(node.elementType)
  }
}
```

#### Finding elements down the tree (children)

```
private fun findChildElement(element: PsiElement?, token: IElementType?): PsiElement? {
  return findChildElement(element, TokenSet.create(token))
}

private fun findChildElement(element: PsiElement?, tokenSet: TokenSet): PsiElement? {
  if (element == null) return null

  val processor: FindElement<PsiElement> =
      object : FindElement<PsiElement>() {
        // If found, returns false. Otherwise returns true.
        override fun execute(each: PsiElement): Boolean {
          callCheckCancelled()
          if (tokenSet.contains(each.node.elementType)) return setFound(each)
          else return true
        }
      }

  element.accept(object : PsiRecursiveElementWalkingVisitor() {
    override fun visitElement(element: PsiElement) {
      callCheckCancelled()
      val isFound = !processor.execute(element)
      if (isFound) stopWalking()
      else super.visitElement(element)
    }
  })

  return processor.foundElement
  }
```

#### Threading considerations

Here are the threading rules:

1. PSI access can happen in any thread (EDT or background), as long as the read lock (via its read action) is acquired before doing so.
2. PSI mutation can only happen in the EDT (not a background thread), since as soon as the write lock (via its write action) is acquired, that means that code is now running in the EDT.

Here’s the action implementation that calls the code shown above. And it uses a very optimized approach to acquiring read and write locks, and using the background thread for blocking network IO.

1. The background thread w/ read action is used to find the hyperlink.
2. The background thread is used to shorten this long hyperlink w/ a short one. This entails making blocking network IO call. And no locks are held during this phase.
3. Finally, a write action is acquired in the EDT to actually mutate the PSI tree w/ the information from the first 2 parts above. This is where the long link is replaced w/ the short link and the PSI is mutated.

And all 3 operations are done in a `Task.Backgroundable` which can be cancelled at anytime and it will end as soon as it can w/out changing anything under the caret in the editor.

* If the task actually goes to completion then a notification is shown reporting that the background task was run successfully.
* And if it gets cancelled in the meantime, then a dialog box is shown w/ an error message.

```
class EditorReplaceLink(val shortenUrlService: ShortenUrlService = TinyUrl()) : AnAction() {
  /**
   * For some tests this is not initialized, but accessed when running [doWorkInBackground]. Use [callCheckCancelled]
   * instead of a direct call to `CheckCancelled.invoke()`.
   */
  private lateinit var checkCancelled: CheckCancelled
  @VisibleForTesting
  private var myIndicator: ProgressIndicator? = null

  override fun actionPerformed(e: AnActionEvent) {
    val editor = e.getRequiredData(CommonDataKeys.EDITOR)
    val psiFile = e.getRequiredData(CommonDataKeys.PSI_FILE)
    val project = e.getRequiredData(CommonDataKeys.PROJECT)
    val progressTitle = "Doing heavy PSI mutation"

    object : Task.Backgroundable(project, progressTitle) {
      var result: Boolean = false

      override fun run(indicator: ProgressIndicator) {
        if (PluginManagerCore.isUnitTestMode) {
          println("🔥 Is in unit testing mode 🔥️")
          // Save a reference to this indicator for testing.
          myIndicator = indicator
        }
        checkCancelled = CheckCancelled(indicator, project)
        result = doWorkInBackground(editor, psiFile, project)
      }

      override fun onFinished() {
        Pair("Background task completed", if (result) "Link shortened" else "Nothing to do").notify()
      }
    }.queue()
  }

  enum class RunningState {
    NOT_STARTED, IS_RUNNING, HAS_STOPPED, IS_CANCELLED
  }

  @VisibleForTesting
  fun isRunning(): RunningState {
    if (myIndicator == null) {
      return NOT_STARTED
    }
    else {
      return when {
        myIndicator!!.isCanceled -> IS_CANCELLED
        myIndicator!!.isRunning  -> IS_RUNNING
        else                     -> HAS_STOPPED
      }
    }
  }

  @VisibleForTesting
  fun isCanceled(): Boolean = myIndicator?.isCanceled ?: false

  /**
   * This function returns true when it executes successfully. If there is no work for this function to do then it
   * returns false. However, if the task is cancelled (when wrapped w/ a [Task.Backgroundable], then it will throw
   * an exception (and aborts) when [callCheckCancelled] is called.
   */
  @VisibleForTesting
  fun doWorkInBackground(editor: Editor, psiFile: PsiFile, project: Project): Boolean {
    // Acquire a read lock in order to find the link information.
    val linkInfo = runReadAction { findLink(editor, project, psiFile) }

    callCheckCancelled()

    // Actually shorten the link in this background thread (ok to block here).
    if (linkInfo == null) return false
    linkInfo.linkDestination = shortenUrlService.shorten(linkInfo.linkDestination) // Blocking call, does network IO.

    callCheckCancelled()

    // Mutate the PSI in this write command action.
    // - The write command action enables undo.
    // - The lambda inside of this call runs in the EDT.
    WriteCommandAction.runWriteCommandAction(project) {
      if (!psiFile.isValid) return@runWriteCommandAction
      replaceExistingLinkWith(project, linkInfo)
    }

    callCheckCancelled()

    return true
  }

  // All the other functions shown in paragraphs above about going up and down the PSI tree.
}
```

Notes on `callCheckCancelled()`:

1. It is also important to have checks to see whether the task has been cancelled or not through out the code. You will find these calls in the functions to walk and up down the tree above. The idea is to include these in every iteration in a loop to ensure that the task is aborted if this task cancellation is detected as soon as possible.
2. Also, in some other places in the code where make heavy use of the PSI API, we can avoid making these checks, since the JetBrains code is doing these cancellation checks for us. However, in places where we are mainly manipulating our own code, we have to make these checks manually.

Here’s the `CheckCancelled` class that is used throughout the code.

```
fun callCheckCancelled() {
  try {
    checkCancelled.invoke()
  }
  catch (e: UninitializedPropertyAccessException) {
    // For some tests [checkCancelled] is not initialized. And accessing a lateinit var will throw an exception.
  }
}

/**
 * Both parameters are marked Nullable for testing. In unit tests, a class of this object is not created.
 */
class CheckCancelled(private val indicator: ProgressIndicator?, private val project: Project?) {
  operator fun invoke() {
    if (indicator == null || project == null) return

    println("Checking for cancellation")

    if (indicator.isCanceled) {
      println("Task was cancelled")
      ApplicationManager
          .getApplication()
          .invokeLater {
            Messages.showWarningDialog(
                project, "Task was cancelled", "Cancelled")
          }
    }

    indicator.checkCanceled()
    // Can use ProgressManager.checkCancelled() as well, if we don't want to pass the indicator around.
  }
}
```

#### Additional references

Please take a look at the JetBrains (JB) Platform SDK DevGuide [section on PSI](https://plugins.jetbrains.com/docs/intellij/psi.html?from=jetbrains.org).

Also, please take a look at the VFS and Document section to get an idea of the other ways to access file contents in your plugin.

1. [JB docs on threading](https://plugins.jetbrains.com/docs/intellij/general-threading-rules.html?from=jetbrains.org)
2. [Threading issues](https://plugins.jetbrains.com/docs/intellij/performance.html?from=jetbrains.org#avoiding-ui-freezes)
3. [`ExportProjectZip.java` example](https://github.com/JetBrains/android/blob/master/android/src/com/android/tools/idea/actions/ExportProjectZip.java)

