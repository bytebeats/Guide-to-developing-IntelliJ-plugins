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

## IDEA threading model

For the most part, the code that you write in your plugins is executed on the main thread. Some operations such as changing anything in the IDE’s “data model” (PSI, VFS, project root model) all have to done in the main thread in order to keep race conditions from occurring. Here’s the [official docs](https://plugins.jetbrains.com/docs/intellij/general-threading-rules.html#modality-and-invokelater) on this.

There are many situations where some of your plugin code needs to run in a background thread so that the UI doesn’t get frozen when long running operations occur. The plugin SDK has quite a few strategies to allow you to do this. However, one thing that you should not do is use `SwingUtilities.invokeLater()`.
1.  One of the ways that you could run background tasks is by using `TransactionGuard.submitTransaction()` but that has been deprecated.
2.  The new way is to use `ApplicationManager.getApplication().invokeLater()`.

> # API deprecations
> * You can find a detailed list of the major API deprecations and changes in various versions of IDEA [here](https://plugins.jetbrains.com/docs/intellij/api-notable.html).
> * You can search the JB platform codebase for deprecations by looking for `@ApiStatus.ScheduledForRemoval(inVersion = <version>)`, where `<version>` can be `2020.1`, etc.

However, before we can understand what all of this means exactly, there are some important concepts that we have to understand - “modality”, and “write lock”, and “write-safe context”. So we will start with talking about `submitTransaction()` and get to each of these concepts along the way to using `invokeLater()`.

## What was submitTransaction() all about?

The now deprecated `TransactionGuard.submitTransaction(Runnable)` basically runs the passed `Runnable` in:
1.  a write-safe context (which simply ensures that no one will be able to perform an unexpected IDE model data changes using `SwingUtilities#invokeLater()`, or equivalent APIs, while a dialog is shown).
2.  in a write thread (eg the EDT).

However, `submitTransaction()` does not acquire a write lock or start a write action (this must be done explicitly by your code if you need to modify IDE model data).

> What does a write action have to do with a write-safe context? Nothing. However, it easy to conflate the two concepts and think that a write-safe context has a write lock; it does not. These are two separate things -
> 1.  write-safe context,
> 2.  write lock.
> You can be in a write-safe context, and you will still need to acquire a write lock to mutate IDE model data. You can also use the following methods to check if your execution context already holds the write lock:
> 1.  `ApplicationManager.getApplication().isWriteAccessAllowed()`.
> 2.  `ApplicationManager.getApplication().assertWriteAccessAllowed()`.

## What is a write-safe context?
The [JavaDocs](https://github.com/JetBrains/intellij-community/blob/master/platform/core-api/src/com/intellij/openapi/application/TransactionGuard.java#L25) from `TransactionGuard.java` explain what a write-safe context is:
* A mechanism to ensure that no one will be able to perform an unexpected IDE model data changes using `SwingUtilities#invokeLater()` or analogs while a dialog is shown.

Here are some examples of write-safe contexts:

* `Application#invokeLater(Runnable, ModalityState)` calls with a modality state that’s either non-modal or was started inside a write-safe context. The use cases shown in the sections below are related to the non-modal scenario.
* Direct user activity processing (key/mouse presses, actions) in non-modal state.
* User activity processing in a modality state that was started (e.g. by showing a dialog or progress) in a write-safe context.

Here is more information about how to handle code running on the EDT:

* Code running in the EDT is not necessarily in a write-safe context, which is why `submitTransaction()` provided the mechanism to execute a given `Runnable` in a write-safe context (on the EDT itself).
* There are some exceptions to this, for example code running in actions in a non-modal state, while running on the EDT are also running in a write-safe context (described above).
* The JavaDocs from [`@DirtyUI` annotation source](https://github.com/JetBrains/intellij-community/blob/master/platform/util/ui/src/com/intellij/ui/DirtyUI.java#L8) also provide some more information about UI code running in the EDT and write actions.
```
/**
* <p>
* This annotation specifies code which runs on Swing Event Dispatch Thread and accesses IDE model (PSI, etc.)
* at the same time.
* <p>
* Accessing IDE model from EDT is prohibited by default, but many existing components are designed without
* such limitations. Such code can be marked with this annotation which will cause a dedicated instrumenter
* to modify bytecode to acquire Write Intent lock before the execution and release after the execution.
* <p>
* Marked methods will be modified to acquire/release IW lock. Marked classes will have a predefined set of their methods
* modified in the same way. This list of methods can be found at {@link com.intellij.ide.instrument.LockWrappingClassVisitor#METHODS_TO_WRAP}
*
* @see com.intellij.ide.instrument.WriteIntentLockInstrumenter
* @see com.intellij.openapi.application.Application
*/
```

Here are some best practices for code that runs in the EDT:

1.  Any writes to the IDE data model must happen on the write thread, which is the EDT.
2.  Even though you can safely read IDE model data from the EDT (without acquiring a read lock), you will still need to acquire a write lock in order to modify IDE model data.
3.  Code running in the EDT must explicitly acquire write locks explicitly (by wrapping that code in a write action) in order to change any IDE model data. This can be done by wrapping the code in a write action with `ApplicationManager.getApplication().runWriteAction()` or, `WriteAction run()/compute()`.
4.  If your code is running in a dialog, and you don’t need to perform any write operations to IDE data models, then you can simply use `SwingUtilities.invokeLater()` or analogs. You only need a write-safe context to run in the right modality if you plan to change the IDE data models.
5.  The IntelliJ Platform SDK docs have more details on IDEA threading [here](https://plugins.jetbrains.com/docs/intellij/general-threading-rules.html?from=jetbrains.org).

## What is the replacement for submitTransaction()?

The replacement for using `TransactionGuard.submitTransaction(Runnable, ...)` is `ApplicationManager.getApplication().invokeLater(Runnable, ...)`. `invokeLater()` makes sure that your code is running in a write-safe context, but you still need to acquire a write lock if you want to modify any IDE model data.

## invokeLater() and ModalityState

`invokeLater()` can take a `ModalityState` parameter in addition to a `Runnable`. Basically you can choose when to actually execute your `Runnable`, either ASAP, during the period when a dialog box is displayed, or when all dialog boxes have been closed.

> To see source code examples of how `ModailtyState` can be used in a dialog box used in a plugin (that is written using the Kotlin UI DSL) check out this sample [ShowKotlinUIDSLSampleInDialogAction.kt](https://github.com/nazmulidris/idea-plugin-example/blob/main/src/main/kotlin/ui/ShowKotlinUIDSLSampleInDialogAction.kt).
> To learn more about what the official docs say about this, check out the [JavaDocs](https://github.com/JetBrains/intellij-community/blob/master/platform/core-api/src/com/intellij/openapi/application/ModalityState.java#L23) in `ModalityState.java`. The following is an in-depth explanation of how this all works.

There’s a user flow that is described in the docs that goes something like this:

1.  Some action runs in a plugin, which enqueues `Runnable A` on the EDT for later invocation, using `SwingUtilities.invokeLater()`.
2.  Then, the action shows a dialog box, and waits for user input.
3.  While the user is thinking about what choice to make on the dialog box, that `Runnable A` has already started execution, and it does something drastic to the IDE data models (like delete a project or something).
4.  The user finally makes up their mind and makes a choice. At this point the code that is running in the action is not aware of the changes that have been made by `Runnable A`. And this can cause some big problems in the action.
5.  `ModalityState` allows you to exert control over when the `Runnable A` is actually executed at a later time.

Here’s some more information about this scenario. If you set a breakpoint at the start of the action’s execution, before it enqueued Runnable A, you might see the following if you look at `ApplicationManager.getApplication().myTransactionGuard.myWriteSafeModalities` in the debugger. This is BEFORE the dialog box is shown.

```
 myWriteSafeModalities = {ConcurrentWeakHashMap@39160}  size = 3
 {ModalityStateEx@39091} "ModalityState.NON_MODAL" -> {Boolean@39735} true
 {ModalityStateEx@41404} "ModalityState:{}" -> {Boolean@39735} true
 {ModalityStateEx@40112} "ModalityState:{}" -> {Boolean@39735} true
```

**AFTER** the dialog box is shown you might see something like the following for the same `myWriteSafeModalities` object in the debugger (if you set a breakpoint after the dialog box has returned the user selected value).

```
 myWriteSafeModalities = {ConcurrentWeakHashMap@39160}  size = 4
 {ModalityStateEx@39091} "ModalityState.NON_MODAL" -> {Boolean@39735} true
 {ModalityStateEx@41404} "ModalityState:{}" -> {Boolean@39735} true
 {ModalityStateEx@43180} "ModalityState:{[...APPLICATION_MODAL,title=Sample..." -> {Boolean@39735} true
 {ModalityStateEx@40112} "ModalityState:{}" -> {Boolean@39735} true
```

Notice that a new `ModalityState` has been added, which basically points to the dialog box being displayed.
So this is how the modality state parameter to `invokeLater()` works.

1.  If you don’t want the `Runnable` that you pass to it to be executed immediately, then you can specify `ModalityState.NON_MODAL`, and this will run it after the dialog box has closed (there are no more modal dialogs in the stack of active modal dialogs).
2.  Instead if you wanted to run it immediately, then you can run it using `ModalityState.any()`.
3.  However, if you pass the `ModalityState` of the dialog box as a parameter, then this `Runnable` will only execute while that dialog box is being displayed.
4.  Now, even when the dialog box is displayed, if you enqueued another `Runnable` to run with `ModalityState.NON_MODAL`, then it will be run after the dialog box is closed.

## How to use invokeLater to perform asynchronous and synchronous tasks

The following sub sections demonstrate how to get away from using `submitTransaction()` and switch to using `invokeLater()` for the given use cases, an even remove the use of them altogether when possible.

### Synchronous execution of Runnable (from an already write-safe context using submitTransaction()/invokeLater())

In the scenario where your code is already running in a write-safe context, there is no need to use `submitTransaction()` or `invokeLater()` to queue your `Runnable`, and you can execute its contents directly.
> From `TransactionGuard.java#submitTransaction()` JavaDoc on [Line 76](https://github.com/JetBrains/intellij-community/blob/master/platform/core-api/src/com/intellij/openapi/application/TransactionGuard.java#L76): “In a definitely write-safe context, just replace this call with {@code transaction} contents. Otherwise, replace with {@link Application#invokeLater} and take care that the default or explicitly passed modality state is write-safe.”

Let’s say that you have a button which has a listener, in a dialog, or tool window. When the button is clicked, it should execute a `Runnable` in a write-safe context. Here’s the code that registers the action listener to the button.

```
init {
  button.addActionListener {
    processButtonClick()
  }
}
```

Here’s the code that queues the `Runnable`. If you run `TransactionGuard.getInstance().isWriteSafeModality(ModalityState.NON_MODAL)` inside the following function it returns true.

```
private fun processButtonClick() {
  TransactionGuard.submitTransaction(project, Runnable {
    // Do something to the IDE data model in a write action.
  })
}
```

However this code is running in the EDT, and since `processButtonClick()` uses a write action to perform its job, so there’s really no need to wrap it in a `submitTransaction()` or `invokeLater()` call. Here we have code that is:

1.  Running the EDT,
2.  Already running in a write-safe context,
3.  And, processButtonClick() itself wraps its work in a write action.

In this case, it is possible to safely remove the `Runnable` and just directly call its contents. So, we can replace it with something like this.

```
ApplicationManager.getApplication().assertIsDispatchThread()
// Do something to the IDE data model in a write action.
```

### Asynchronous execution of Runnable (from a write-unsafe / write-safe context using submitTransaction())

The simplest replacement for the `Runnable` and `Disposable` that are typically passed to `submitTransaction()`, is simply to replace the call:

```
TransactionGuard.submitTransaction(Disposable, Runnable)
```

With

```
ApplicationManager.ApplicationManager.getApplication().invokeLater(
  Runnable, ComponentManager.getDisposed())
```

1.  Note that IDE data model objects like project, etc. all expose a `Condition` object from a call to `getDisposed()`.
2.  If you have to create a `Condition` yourself, then you can just wrap your `Disposable` in a call to `Disposer.isDisposed(Disposable)`.

Let’s take a look at a simple example of replacing old code that uses `submitTransaction()`.

1.  OId code that uses `submitTransaction`. <br>```TransactionGuard.submitTransaction(myProject, ()->{ /* Runnable */ }```
2.  New code that uses `invokeLater()`. <br>```ApplicationManager.getApplication().invokeLater(()->{ /* Runnable */ }, myProject.getDisposed());```

Here’s a slightly more complex example where a `Condition` object must be created.

1.  OId code that uses `submitTransaction`. <br>```TransactionGuard.submitTransaction(myComponent.getModel(), ()->{ /* Runnable */ })```
2.  New code that uses `invokeLater()`. <br>```ApplicationManager.getApplication().invokeLater(()->{ /* Runnable */ }, ignore -> Disposer.isDisposed(myComponent.getModel()));```
3.  You can also just check for the condition `Disposer.isDisposed(myComponent.getModel())` at the start of the `Runnable`. And not even pass the `Condition` in `invokeLater()` if you like.

### Side effect - invokeLater() vs submitTransaction() and impacts on test code

When moving from `submitTransaction()` to `invokeLater()` some tests might need to be updated, due to one of the major differences between how these two functions actually work.

In some cases, one of the consequences of using `invokeLater()` instead of `submitTransaction()` is that in tests, you might have to call `PlatformTestUtil.dispatchAllInvocationEventsInIdeEventQueue()` to ensure that the `Runnable`s that are queued for execution on the EDT are actually executed (since the tests run in a headless environment). You can also call `UIUtil.dispatchAllInvocationEvents()` instead of `PlatformTestUtil.dispatchAllInvocationEventsInIdeEventQueue()`. When using `submitTransaction()` in the past, this was not necessary.

Here’s old code, and test code.

```
@JvmStatic
fun myFunction(project: Project) {
  TransactionGuard.submitTransaction(
    project, Runnable {
      // Do stuff.
    })
})
}

fun testMyFunction() {
  myFunction(project)
  verify(/* that myFunction() runs */)
}
```

Here’s the new code, and test code.

```
@JvmStatic
fun myFunction(project: Project) {
  ApplicationManager.getApplication().invokeLater(
    Runnable {
      // Do stuff.
    })
}, project.disposed)
}

fun testMyFunction() {
  myFunction(project)
  PlatformTestUtil.dispatchAllInvocationEventsInIdeEventQueue()
  verify(/* that myFunction() runs */)
}
```


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

### Dynamic plugins

One of the biggest changes that JetBrains has introduced to the platform SDK in 2020 is the introduction of [Dynamic Plugins](https://plugins.jetbrains.com/docs/intellij/dynamic-plugins.html?from=jetbrains.org). Moving forwards the use of components of any kind are banned.

Here are some reasons why:

1. The use of components result in plugins that are unloadable (due to it being impossible to dereference the Plugin component classes that was loaded by a classloader when IDEA itself launched).
2. Also, they impact startup performance since code is not lazily loaded if its needed, which slows down IDEA startup.
3. Plugins can be kept around for a long time even after they might be unloaded, due to attaching disposer to a parent that might outlive the lifetime of the project itself.

In the new dynamic world, everything is loaded lazily and can be garbage collected. Here is more information on the [deprecation of components](https://plugins.jetbrains.com/docs/intellij/plugin-components.html?from=jetbrains.org).

There are some caveats of doing this that you have to keep in mind if you are used to working with components.

1. Here’s a [very short migration guide from JetBrains](https://plugins.jetbrains.com/docs/intellij/plugin-components.html?from=jetbrains.org) that provides some highlights of what to do in order to move components over to be services, [startup activities](https://www.plugin-dev.com/intellij/general/plugin-initial-load/), listeners, or extensions.
2. You have to pick different parent disposables for services, extensions, or listeners (in what used to be a component). You can’t scope a [Disposable](https://plugins.jetbrains.com/docs/intellij/disposers.html?from=jetbrains.org#choosing-a-disposable-parent) to the project anymore, since the plugin can be unloaded during the life of a project.
3. Don’t cache copies of the implementations of registered extension points, as these might cause leaks due to the dynamic nature of the plugin. Here’s more information on [dynamic extension points](https://plugins.jetbrains.com/docs/intellij/plugin-extension-points.html?from=jetbrains.org#dynamic-extension-points). These are extension points that are marked as dynamic so that IDEA can reload them if needed.
4. Please read up on [the dynamic plugins restrictions and troubleshooting guide](https://plugins.jetbrains.com/docs/intellij/dynamic-plugins.html?from=jetbrains.org) that might be of use as you migrate your components to be dynamic.
5. Plugins now support [auto-reloading](https://plugins.jetbrains.com/docs/intellij/ide-development-instance.html?from=jetbrains.org#enabling-auto-reload), which you can disable if this causes you issues.

#### Extension points postStartupActivity, backgroundPostStartupActivity to initialize a plugin on project load

There are 2 extension points to do just this `com.intellij.postStartupActivity` and `com.intellij.backgroundPostStartupActivity`.

* [Official doc](https://plugins.jetbrains.com/docs/intellij/plugin-components.html?from=jetbrains.org)
* [Usage samples](https://intellij-support.jetbrains.com/hc/en-us/community/posts/360002476840-How-to-auto-start-initialize-plugin-on-project-loaded-)

Here are all the ways in which to use a `StartupActivity`:

* Use a `postStartupActivity` to run something on the EDT during project open.
* Use a `postStartupActivity` implementing `DumbAware` to run something on a background thread during project open in parallel with other dumb-aware post-startup activities. Indexing is not complete when these are running.
* Use a `backgroundPostStartupActivity` to run something on a background thread approx 5 seconds after project open.
* More information from the IntelliJ platform codebase about these [startup activities](https://github.com/JetBrains/intellij-community/blob/165e3b323c90e884972999e546f1e7085995ef7d/platform/service-container/overview.md).

> 💡 You wil find many examples of how these can be used in the migration strategies section.

#### Light services

A light service allows you to declare a class as a service simply by using an annotation and not having to create a corresponding entry in `plugin.xml`.

> Read all about light services [here](https://plugins.jetbrains.com/docs/intellij/plugin-services.html?from=jetbrains.org#light-services).

The following are some code examples of using the `@Service` annotation for a very simple service that isn’t project or module scoped (but is application scoped).

```
import com.intellij.openapi.components.Service
import com.intellij.openapi.components.ServiceManager

@Service
class LightService {

  val instance: LightService
    get() = ServiceManager.getService(LightService::class.java)

  fun serviceFunction() {
    println("LightService.serviceFunction() run")
  }

}
```

Notes on the snippet:

* There is no need to register these w/ `plugin.xml` making them really easy to use.
* Depending on the constructor that is overloaded, IDEA will figure out whether this is a project, module, or application scope service.
* The only restriction to using light services is that they must be final (which all Kotlin classes are by default).

> ⚠️ The use of module scoped light services are discouraged, and not supported.
> ⚠️ You might find yourself looking for a projectService declaration that’s missing from `plugin.xml` but is still available as a service, in this case, make sure to look out for the following annotation on the service class `@Service`.

Here’s a code snippet for a light service that is scoped to a project.

```
@Service
class LightService(private val project: Project) {

  companion object {
    fun getInstance(project: Project): LightService {
      return ServiceManager.getService(project, LightService::class.java)
    }
  }

  fun serviceFunction() {
    println("LightService.serviceFunction() run w/ project: $project")
  }
}
```

> 💡️ You can save a reference to the open project, since a new instance of a service is created per project (for project-scope services).

#### Migration strategies

There are a handful of ways to go about removing components and replacing them w/ services, startup activities, listeners, etc. The following is a list of common refactoring strategies that you can use depending on your specific needs.

#### 1. Component -> Service

In many cases you can just replace the component w/ a service, and get rid of the project opened and closed methods, along w/ the component name and dispose component methods.

Another thing to watch out for is to make sure that the `getInstance()` methods all make a `getService()` call and not `getComponent()`. Look at tests as well to see if they are using `getComponent()` instead of `getService()` to get an instance of the migrated component.

Here’s an XML snippet of what this might look like:

```<projectService serviceImplementation="MyServiceClass" />```

> 💡️ If you use a light service then you can skip registering the service class in `plugin.xml`.

Here’s the code for the service class:

```
class MyServiceClass : Disposable {
  fun dispose() { /** Custom logic that runs when the project is closed. */ }

  companion object {
    @JvmStatic
    fun getInstance(project: Project) = project.getService(MyServiceClass::class.java)
  }
}
```

> 💡️ If you don’t need to perform any custom login in your service when the project is closed, then there is no need to implement `Disposable` and you can just remove the `dispose()` method.

> #### Disposing the service and choosing a parent disposable
> In order to clean up after the service, it can simply implement the `Disposable` interface and put the logic for clean up in the `dispose()` method. This should suffice for most situations, since IDEA will [automatically take care of cleaning up](https://plugins.jetbrains.com/docs/intellij/disposers.html?from=jetbrains.org#automatically-disposed-objects) the service instance.
> 1. Application-level services are automatically disposed by the platform when the IDE is closed, or the plugin providing the service is unloaded.
> 2. Project-level services are automatically disposed when the project is closed or the plugin is unloaded.
> However, if you still want to exert finer control over when you want your service to be disposed, you can use `Disposer.register()` by passing a `Project` or `Application` service instance as the parent argument.

> 💡️ Here’s more information from [JetBrains official docs on choosing a disposable parent](https://plugins.jetbrains.com/docs/intellij/disposers.html?from=jetbrains.org#choosing-a-disposable-parent).

> #### Summary
> 1. For resources required for the entire lifetime of a plugin use an application-level or project-level service.
> 2. For resources required while a dialog is displayed, use a `DialogWrapper.getDisposable()`.
> 3. For resources required while a tool window is displayed, pass your instance implementing `Disposable` to `Context.setDisposer()`.
> 4. For resources w/ a shorter lifetime, create a disposable using a `Disposer.newDisposable()` and dispose it manually using `Disposable.dispose()`.
> 5. Finally, when passing our own parent object be careful about non-capturing-lambdas.

#### 2. Component -> postStartupActivity

This is a very straightforward replacement of a component w/ a startup activity. The logic that is in `projectOpened()` simply goes into the `runActivity(project: Project)` method. The same approach used in Component -> Service still applies (w/ removing needless methods and using `getService()` calls).

#### 3. Component -> postStartupActivity + Service

This is a combination of the two strategies above. Here’s a pattern that you can use to detect if this is the right approach or not. If the component had some logic that executed in `projectOpened()` which requires a `Project` instance then you can do the following:

1. Make the component a service in the `plugin.xml` file. Also, add a startup activity.
2. Instead of your component extending `ProjectComponent` have it implement `Disposable` if you need to run some logic when it is disposed (when the project is closed). Or just have it not implement any interface or extend any class. Make sure to accept a parameter of `Project` in the constructor.
3. Rename the `projectOpened()` method to `onProjectOpened()`. Add any logic you might have had in any `init{}` block or any other constructors to this method.
4. Create a `getInstance(project: Project)` function that looks up the service instance from the given project.
5. Create a startup activity inner class called eg: `MyStartupActivity` which simply calls `onProjectOpened()`.

This is roughly what things will end up looking like:

```
<projectService serviceImplementation="MyServiceClass" />
<postStartupActivity implementation="MyServiceClass$MyStartupActivity"/>
```

And the Kotlin code changes:

```
class MyServiceClass {
  fun onProjectOpened() { /** Stuff. */ }

  class MyStartupActivity : StartupActivity.DumbAware {
    override fun runActivity(project: Project) = getInstance(project).onProjectOpened()
  }

  companion object {
    @JvmStatic
    fun getInstance(project: Project): MyServiceClass = project.getService(YourServiceClass::class.java)
  }
}
```

#### 4. Component -> projectListener

Many components just subscribe to a topic on the message bus in the `projectOpened()` method. In these cases, it is possible to replace the component entirely by (declaratively) registering a [projectListener](https://plugins.jetbrains.com/docs/intellij/plugin-listeners.html?from=jetbrains.org) in your module’s `plugin.xml`.

Here’s an XML snippet of what this might look like (goes in `plugin.xml`):

```
<listener class="MyListenerClass"
          topic="com.intellij.execution.testframework.sm.runner.SMTRunnerEventsListener"/>
<listener class="MyListenerClass"
          topic="com.intellij.execution.ExecutionListener"/>
```

And the listener class itself:

```
class MyListenerClass(val project: Project) : SMTRunnerEventsAdapter(), ExecutionListener {}
```

#### 5. Component -> projectListener + Service

Sometimes a component can be replaced w/ a service and a `projectListener`, which is simply combining two of the strategies shown above.

#### 6. Delete Component

There are some situations where the component might have been deprecated already. In this case simply remove it from the appropriate module’s `plugin.xml` and you can delete those files as well.

#### 7. Component -> AppLifecycleListener

There are some situations where an application component has to be launched when IDEA starts up and it has to be notified when it shuts down. In this case you can use [AppLifecycleListener](https://github.com/JetBrains/intellij-community/blob/master/platform/platform-impl/src/com/intellij/ide/AppLifecycleListener.java) to attach a [listener](https://plugins.jetbrains.com/docs/intellij/plugin-listeners.html?from=jetbrains.org) to IDEA that does just [this](https://plugins.jetbrains.com/docs/intellij/plugin-components.html?from=jetbrains.org).

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

