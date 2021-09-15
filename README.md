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

## UI (JetBrains UI components, Swing and Kotlin UI DSL)

There are many ways for a plugin to interact w/ an end user - keyboard shortcuts that bind to actions, and UI components that the plugin provides, which can be totally custom, or integrated into the existing IDE’s UI components. JetBrains also provides many UI controls that are applicable for different scenarios.

There is a range of components that span from simple fire and forget notifications, to more sophisticated UI that allows intense interaction w/ the user. There is even a Kotlin DSL for UI components that makes it relatively easy to create forms in IDEA.

There is even a way to create forms using Swing, which is being phased out in favor of the Kotlin UI DSL. In addition to all of this, you can even create custom themes for IDEA.

One effective approach to writing UI code for plugins is to take a look at how some UI code is written in `intellij-community` repo itself, to see how JetBrains does it. Documentation is sparse to non existent, and the best way sometimes is to take a look at the source for IDEA itself to find out how certain UI is created.

Here are good resources to take a look when considering writing UI code for plugins.

* [Swing Layout Managers](https://docs.oracle.com/javase/tutorial/uiswing/layout/visual.html)
* [Introduction to Swing](https://docs.oracle.com/javase/tutorial/uiswing/index.html)
* [MigLayout, used in Kotlin UI DSL](https://github.com/mikaelgrev/miglayout)

### Simple UI components - notifications, popups, dialogs

This section covers some simple UI components, and the next sections get into more sophisticated interactions w/ users (via forms, dialogs, and the settings UI).

#### Notifications

Notifications are a really simple way to provide some information to the user. You can even attach actions to notifications in case you want the user to perform an action when they get notified.

* Notifications show up in the “Event Log” tool window in the IDE.
* You can choose to have any shown notifications to be logged as well.
* Notifications can be project specific (and be shown in the IDE window containing a project), or be a general notification for the entire IDE.
* Here are the [official docs](https://plugins.jetbrains.com/docs/intellij/notifications.html?from=jetbrains.org) on notifications.

This is an example of a simple action that displays two notifications using slightly different ways. Here is the snippet for this action that goes into `plugin.xml`.

```
<actions>
  <!-- Add notification action. -->
  <action id="MyPlugin.actions.ShowNotificationSample" class="ui.ShowNotificationSampleAction"
      description="Show sample notifications" text="Sample Notification" icon="/icons/ic_extension.svg" />
</actions>
```

Here’s the start of the class that implements this action, to set things up.

```
package ui

class ShowNotificationSampleAction : AnAction() {
  private val GROUP_DISPAY_ID = "UI Samples"
  private val messageTitle = "Title of notification"
  private val messageDetails = "Details of notification"

  override fun actionPerformed(e: AnActionEvent) {
    aNotification()
    anotherNotification(e)
  }

  private fun anotherNotification(e: AnActionEvent) {...}
  private fun aNotification() {...}
```

Approach 1 - The following is an example of using a static method on the `Notifications.Bus` to display a notification.

```
/**
 * One way to show notifications.
 * 1) This notification won't be logged to "Event Log" tool window.
 * 2) And it is project specific.
 */
private fun aNotification() {
  val notification = Notification(GROUP_DISPAY_ID,
                                  "1 .$messageTitle",
                                  "1 .$messageDetails",
                                  NotificationType.INFORMATION)
  Notifications.Bus.notify(notification)
}
```

Approach 2 - Here’s an another way to show notifications by creating a notification group.

```
/**
 * Another way to show notifications.
 * 1) This will be logged to "Event Log" and is not tied to a specific project.
 */
private fun anotherNotification(e: AnActionEvent) {
  val project = e.getRequiredData(CommonDataKeys.PROJECT)
  val notificationGroup = NotificationGroup(GROUP_DISPAY_ID, NotificationDisplayType.BALLOON, false)
  val notification = notificationGroup
      .createNotification("2. $messageTitle",
                          "2. $messageDetails",
                          NotificationType.INFORMATION,
                          null)
  notification.notify(project)
}
```

#### Popups

Popups are a way to get some input from the user w/out interrupting what they’re doing. Popups are displayed w/out any chrome (UI to close them), and they disappear automatically when they lose keyboard focus. IDE uses them extensively in type ahead completion, auto complete, etc.

* Popups allow the user to make a single choice from a list of options that are displayed. Once the user makes a choice you can provide some code (`itemChosenCallback`) that will be run on this selection.
* Popups can display a title.
* They can be movable and resizable (and support remembering their size).
* They can even be nested (show another popup when an item is selected).

The following is an example of an action that shows two nested popups. Here’s the snippet from `plugin.xml` for registering the action.

```
<!-- Add popup action. -->
<action id="MyPlugin.actions.ShowPopupSample" class="ui.ShowPopupSampleAction" text="Sample Popup"
    description="Shows sample popups" icon="/icons/ic_extension.svg" />
```

Here’s the class that implements the action (which shows one popup, and when the user selects an option in the first popup, the second one is shown). It shows multiple ways in which a popup can be created to display a list of items. One approach is to use the `createPopupChooserBuilder()`, and the other is to use `createListPopup()`, both of which are methods on [JBPopupFactory.getInstance()](https://github.com/JetBrains/intellij-community/blob/master/platform/platform-api/src/com/intellij/openapi/ui/popup/JBPopupFactory.java) method.

```
package ui

class ShowPopupSampleAction : AnAction() {
  lateinit var editor: Editor

  override fun actionPerformed(e: AnActionEvent) {
    editor = e.getRequiredData(CommonDataKeys.EDITOR)

    // One way to show a list.
    JBPopupFactory
        .getInstance()
        .createListPopup(MyList {
          // Another way to show a list, using the builder.
          JBPopupFactory
              .getInstance()
              .createPopupChooserBuilder(mutableListOf("one", "two", "three"))
              .setTitle("PopupChooserBuilder")
              .setItemChosenCallback { println("PopupChooserBuilder.onChosen $it") }
              .createPopup()
              .showInBestPositionFor(editor)
        })
        .showInBestPositionFor(editor)
  }
}

class MyList(val onChosenHandler: (String) -> Unit) : BaseListPopupStep<String>() {
  init {
    init("Popup title", mutableListOf("Choice 1", "Choice 2", "Choice 3"), null)
  }

  override fun getTextFor(value: String): String {
    return "TEXT: $value"
  }

  override fun onChosen(selectedValue: String, finalChoice: Boolean): PopupStep<*>? {
    println("MyList.onChosen $selectedValue")
    onChosenHandler(selectedValue)
    return PopupStep.FINAL_CHOICE
  }
}
```

#### Dialogs

When user input is required by your plugin dialogs might be the right component to use. They are modal unlike notifications and popups. IDEA allows a simple boolean response to be returned from a dialog.

In order to create sophisticated UIs that go inside the dialog, it is best to use the Kotlin UI DSL.

Here’s a really simple example of an action that shows a dialog displaying “Press OK or Cancel” and allowing the user and showing OK and Cancel buttons. Here’s the snippet for `plugin.xml`.

```
<!-- Add dialog action. -->
<action id="MyPlugin.actions.ShowDialogSample" class="ui.ShowDialogSampleAction" text="Sample Dialog"
    description="Show sample dialog" icon="/icons/ic_extension.svg" />
```

Here’s the action that uses the [DialogWrapper](https://github.com/JetBrains/intellij-community/blob/master/platform/platform-api/src/com/intellij/openapi/ui/DialogWrapper.java) class to actually show the dialog.

```
package ui

class ShowDialogSampleAction : AnAction() {
  override fun actionPerformed(e: AnActionEvent) {
    val response = SampleDialogWrapper().showAndGet()
    println("Response selected:${if (response) "Yes" else "No"}")
  }
}

class SampleDialogWrapper : DialogWrapper(true) {
  init {
    init()
    title = "Sample Dialog"
  }

  // More info on layout managers: https://docs.oracle.com/javase/tutorial/uiswing/layout/visual.html
  override fun createCenterPanel(): JComponent {
    val panel = JPanel(BorderLayout())
    val label = JLabel("Press OK or Cancel")
    label.preferredSize = Dimension(100, 100)
    panel.add(label, BorderLayout.CENTER)
    return panel
  }
}
```

Note that in this case, `showAndGet()` is used to both show the dialog, and get the response. You can also break this into two steps with `show()` and `isOK()`.

> Please refer to the [official docs](https://github.com/JetBrains/intellij-community/blob/master/platform/platform-api/src/com/intellij/openapi/ui/DialogWrapper.java) for more information on dialogs.
> For more information on Swing layout managers, please refer to this [Java Tutorial article](https://docs.oracle.com/javase/tutorial/uiswing/layout/visual.html).

### Create complex UIs using Kotlin UI DSL (forms, dialogs, settings screens)

In the sections below, we will be using `DialogWrapper` to take the UI we create using the Kotlin UI DSL and show them.

In order to create more complex UIs, you can use the Kotlin UI DSL. You can display these forms in the IDEA Settings UI, or directly in a dialog box. You can also bind the forms to data objects, that you can persist across IDE restarts as well.

#### Understanding the structure of the DSL (layout, row, and cell)

The Kotlin UI DSL allows UIs to be expressed in a layout that contains rows and cells. Things are left to right aligned inside of each row, but you can specify if an item needs to be right aligned. Type ahead completion in the IDE also provides guidance on how to use this DSL.

Here’s an example of a simple UI using this DSL, which contains the following UI components.

1. `textField`
2. `spinner`
3. `checkBox`
4. `noteRow`

```
data class MyData(var myFlag: Boolean, var myString: String, var myInt: Int, var myStringChoice: String)
val myDataObject = MyData()

fun createDialogPanel(): DialogPanel = panel {
  // Restore the selection state of the combo box.
  val comboBoxChoices = listOf("choice1", "choice2", "choice3")
  val savedSelection = myDataObject.myStringChoice // This is an empty string by default.
  val selection = if (savedSelection.isEmpty()) comboBoxChoices.first() else savedSelection
  val comboBoxModel = CollectionComboBoxModel(comboBoxChoices, selection)

  noteRow("This is a row with a note")

  row {
    label("a string")
    textField(myDataObject::myString)
  }

  row {
    label("an int")
    spinner(myDataObject::myInt, minValue = 0, maxValue = 50, step = 5)
  }

  row("ComboBox / Drop down list") {
    comboBox(comboBoxModel, myDataObject::myStringChoice)
  }

  row {
    cell {
      checkBox("", myDataObject::myFlag)
      label("Boolean state::myFlag")
    }
  }

  noteRow("""Note with a link. <a href="http://github.com">Open source</a>""") {
    BrowserUtil.browse(it)
  }
}
```

#### How to bind data to the form UI

One really simple way of binding the UI to a data object that you create is by passing a reference to the `KProperty` for each property that should be rendered by a UI component. This ensures that:

1. The data object populates the UI component to its correct initial state before it is shown.
2. When the user manipulates the UI component and its state changes, this is reflected in the data object’s property.

In the example above, note that some UI components that hold state information require a Kotlin property (from a data object that you provide) to bind to. This is an easy way that the DSL handles data binding in the forms for you.

It is possible to provide explicit setters and getters as well instead of just using the [KProperty](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/-k-property/). This makes it really simple to create forms, since the state is loaded and saved for you, automatically.

#### What UI elements are available for use in the forms

1. The panel function creates a [LayoutBuilder](https://github.com/JetBrains/intellij-community/blob/master/platform/platform-impl/src/com/intellij/ui/layout/LayoutBuilder.kt) and inside of this you can add row (or `noteRow`, or `titledRow`, etc). Check out [RowBuilder](https://github.com/JetBrains/intellij-community/blob/master/platform/platform-impl/src/com/intellij/ui/layout/Row.kt) to see all the row based components you can add here.
2. To see what objects can be added inside each `row`, check out [Cell](https://github.com/JetBrains/intellij-community/blob/master/platform/platform-impl/src/com/intellij/ui/layout/Cell.kt). You can add things like `browserLink`, `comboBox`, `checkBox`, etc.

> Please make sure to read the [official docs](https://plugins.jetbrains.com/docs/intellij/kotlin-ui-dsl.html?from=jetbrains.org) on the DSL.

#### How to display a form in a dialog

The form that’s created using this DSL can be displayed anywhere a `DisplayPanel` can be used (which is just a subclass of JPanel). This includes the Settings UI (shown in the section below) and in a dialog. Here’s an sample action that takes the object returned by `createDialogPanel()` above, and displays it in a dialog, using a `DialogWrapper`.

```
class ShowKotlinUIDSLSampleInDialogAction : AnAction() {
  override fun actionPerformed(e: AnActionEvent) {
    MyDialogWrapper().show()
  }
}

private class MyDialogWrapper : DialogWrapper(true) {
  init {
    init()
    title = "Sample Dialog with Kotlin UI DSL"
  }

  override fun createCenterPanel(): JComponent = createDialogPanel()
}
```

#### How to persist the data created/selected by the user, between IDE restarts (PersistentStateComponent)

If you want the data that you’ve created in your form to be used in the rest of the IDE, it makes sense to persist this data and make it available to other pieces of your plugin. This is where [PersistentStateComponent](https://github.com/JetBrains/intellij-community/blob/master/platform/projectModel-api/src/com/intellij/openapi/components/PersistentStateComponent.java) comes in. It allows you serialize the state of your data object to disk, and deserialize it from disk. In order to have IDEA persist your data object, simply do two things:

1. Make your data class extend `PersistentStateComponent`.
2. Wrap this in a `Service` so that you can access it in any code in your plugin.

Here’s an example that shows this. The `KotlinDSLUISampleService` service contains a `State` object called `myState` that actually holds the data which is bound to the forms. State simply holds 4 properties (which can be bound to various UI components in a form): `myFlag: Boolean`, `myString: String`, `myInt: Int`, `myStringChoice: String`.

```
@Service
@State(name = "KotlinDSLUISampleData", storages = [Storage("kotlinDSLUISampleData.xml")])
class KotlinDSLUISampleService : PersistentStateComponent<KotlinDSLUISampleService.State> {
  companion object {
    val instance: KotlinDSLUISampleService
      get() = getService(KotlinDSLUISampleService::class.java)
  }

  var myState = State()

  // PersistentStateComponent methods.
  override fun getState(): State {
    println("KotlinDSLUISampleData.getState, state: $myState")
    return myState
  }

  override fun loadState(stateLoadedFromPersistence: State) {
    println("KotlinDSLUISampleData.loadState, stateLoadedFromPersistence: $stateLoadedFromPersistence")
    myState = stateLoadedFromPersistence
  }

  // Properties in this class are bound to the Kotlin DSL UI.
  class State {
    var myFlag: Boolean by object : LoggingProperty<State, Boolean>(false) {}
    var myString: String by object : LoggingProperty<State, String>("") {}
    var myInt: Int by object : LoggingProperty<State, Int>(0) {}
    var myStringChoice: String by object : LoggingProperty<State, String>("") {}

    override fun toString(): String =
        "State{ myFlag: '$myFlag', myString: '$myString', myInt: '$myInt', myStringChoice: '$myStringChoice' }"

    /** Factory class to generate synthetic properties, that log every access and mutation to each property. */
    open class LoggingProperty<R, T>(initValue: T) : ReadWriteProperty<R, T> {
      var backingField: T = initValue

      override fun getValue(thisRef: R, property: KProperty<*>): T {
        println("State::${property.name}.getValue(), value: '$backingField'")
        return backingField
      }

      override fun setValue(thisRef: R, property: KProperty<*>, value: T) {
        backingField = value
        println("State::${property.name}.setValue(), value: '$backingField'")
      }
    }
  }
}
```

In the form UI example above, we just bound the data for the UI components to an object created from a simple data class. Here’s an example.

```
row {
  label("a string")
  textField(myDataObject::myString)
}
```

In order to replace that with the `PersistentStateComponent` instance, we would do something like this instead.

```
row {
  label("a string")
  textField(KotlinDSLUISampleService.instance.myState::myString)
}
```

In other words, `myDataObject` is replaced with `KotlinDSLUISampleService.instance.myState`.

There is one very important thing to note in the code above. The `getState()` and `oadState(...)` methods should not be called by your code. These are hooks for IDEA to call into the service in order to manage loading and saving the state to persistence. You should provide accessors to your data properties that do not involve the use of `getState()`. And make sure that these accessors can handle `loadState(...)` providing the “initial” state (if there is anything stored in persistence). And you must make sure that by default, your initial state has to be defined (if there is no customized data to load from persistence, then your state will contain default values). IDEA uses this default state to figure out if the user changed anything in the form UI (the diffs will deviate from the default state).

Also, you can specify many options to IDEA on where to store the persistent data. In this example, we have told IDEA to store any user modified data to an XML file called `kotlinDSLUISampleData.xml` in the `config/options/` folder where IDEA settings are stored. You can also specify options for `storages` that determine whether this data should be roaming disabled or not, and even if you should only store the data on specific platforms.

#### Example of a form UI

Here’s a form that uses Kotlin DSL and the `State` object (provided by the `PersistentStateComponent` service).

```
fun createDialogPanel(): DialogPanel {
  val comboBoxChoices = listOf("choice1", "choice2", "choice3")

  // Restore the selection state of the combo box.
  val savedSelection = instance.myState.myStringChoice // This is an empty string by default.
  val selection = if (savedSelection.isEmpty()) comboBoxChoices.first() else savedSelection
  val comboBoxModel = CollectionComboBoxModel(comboBoxChoices, selection)

  return panel {
    noteRow("This is a row with a note")

    row("[Boolean]") {
      row {
        cell {
          checkBox("", instance.myState::myFlag)
          label("Boolean state::myFlag")
        }
      }
    }

    row("[String]") {
      row {
        label("String state::myString")
        textField(instance.myState::myString)
      }
    }

    row("[Int]") {
      row {
        label("Int state::myInt")
        spinner(instance.myState::myInt, minValue = 0, maxValue = 50, step = 5)
      }
    }

    row("ComboBox / Drop down list") {
      comboBox(comboBoxModel, instance.myState::myStringChoice)
    }

    noteRow("""Note with a link. <a href="http://github.com">Open source</a>""") {
      println("link url: '$it' clicked")
      BrowserUtil.browse(it)
    }
  }
}
```

#### MigLayout, panel, and grow

The panel function (in Kotlin UI DSL) actually creates a [MigLayout](https://github.com/jetbrains/intellij-community/blob/master/platform/platform-impl/src/com/intellij/ui/layout/migLayout/patched/MigLayout.kt#L43). `MigLayout` can produce flowing, grid based, absolute (with links), grouped and docking layouts.

* `panel` accepts `row` functions, which in turn accepts `cell` functions.
* You can pass `grow` and `fill` attributes to `panel` and `cell` functions.
* You can also wrap `JComponents` using the `component` function, which can also take `grow` and `fill` attributes.
* `panel` automatically sizes the dialog, however, you can override it using your own predefined `width` and `height`.

Here’s an example (`myJComponent` is just a `JComponent` that is created somewhere else).

```
override fun createCenterPanel(): JComponent {
    return panel() {
        row {
            cell {
                label("Type into this editor component")
            }
        }
        row {
            cell {
                component(myJComponent).constraints(growX, growY, pushX, pushY)
            }
        }
    }.apply {
        preferredSize = JBUI.size(WIDTH, HEIGHT)
    }
}
companion object {
    const val WIDTH = 1000
    const val HEIGHT = 400
}
```

> Note that `JComponent` objects become callable within the `panel` function and they accept `CCFLags` that constrain their growth.

> Please note that it might be simpler at times to use the [Swing layout managers](https://docs.oracle.com/javase/tutorial/uiswing/layout/visual.html) directly w/out using this DSL.

### Create IDE Settings UI for plugin

Let’s say that we wanted to display the form created above (by `createDialogPanel()`) and we want to display it in a Settings UI, in addition to it being available in a `Dialog`. Note that you can display the form in both places, and have it update the data in the same `PersistentStateComponent` service, which is really convenient.

In order to tell IDEA that you want a form to be displayed in the Settings UI, you can use one of two extension points. In `plugin.xml` you can declare 2 types of “configurables” that allow you to customize the IDE settings UI, where a “configurable” is a base class provided by the JetBrains platform that allows you show your form UI.

1. `projectConfigurable` - these are settings that are specific to a given project.
2. `applicationConfigurable` - these are settings that apply to the entire IDE.

References:
* [Old JB docs](https://plugins.jetbrains.com/docs/intellij/settings.html).
* [New JB docs](https://plugins.jetbrains.com/docs/intellij/kotlin-ui-dsl.html?from=jetbrains.org#configurables).
* [MacOS dark mode sync plugin source](https://github.com/gilday/dark-mode-sync-plugin)

When using the Kotlin UI DSL instead of implementing `Configurable` interface, simply extend `BoundConfigurable`. When doing this you can:

1. Pass a `displayName` to the constructor of the `BoundConfigurable` that will actually show up in the Settings UI, (and you can type-ahead search for).
2. Pass any number of JB platform objects in the constructor, eg:
```
class MyConfigurable(private val lafManager: LafManager) : BoundConfigurable("Display Name")
```
The following is an example of this (`plugin.xml` entry, and a class that extends `BoundConfigurable`).

```
<!-- Add Settings Dialog that is similar to what ShowKotlinUIDSLSampleAction does. -->
<extensions defaultExtensionNs="com.intellij">
  <applicationConfigurable instance="ui.KotlinDSLUISampleConfigurable" />
</extensions>
```

The code from the previous section (Kotlin UI DSL)) is used here to generate the form itself.

```
package ui

/**
 * This application level configurable shows up the in IDE Settings UI.
 */
class KotlinDSLUISampleConfigurable : BoundConfigurable("Kotlin UI DSL") {
  override fun apply() {
    println("KotlinDSLUISampleConfigurable apply() called")
    super.apply()
  }

  override fun cancel() {
    println("KotlinDSLUISampleConfigurable cancel() called")
    super.cancel()
  }

  /** When the form is changed by the user, this returns `true` and enables the "Apply" button. */
  override fun isModified(): Boolean {
    println("KotlinDSLUISampleConfigurable isModified() called, return ${super.isModified()}")
    return super.isModified()
  }

  override fun reset() {
    println("KotlinDSLUISampleConfigurable reset() called")
    super.reset()
  }

  override fun createPanel(): DialogPanel = createDialogPanel()
}
```

### Complex UI creation in dialogs

There are cases where complex UI components need to be created in a `DialogWrapper`. In this case, Kotlin UI DSL can be used or directly creating Swing components using Swing layout managers.

> Please take a look at the [idea-plugin-example2](https://github.com/nazmulidris/idea-plugin-example2) repo that contains a plugin that has some of the functionality shown below.

The following is an example of doing the latter to create a dialog that allows the user to paste text from the clipboard into an `Editor` component. The paste operation is actually implemented as an undoable command. It automatically pastes any text that is in the clipboard into the editor when the dialog is shown.

```
class ComplexDialog(private val project: Project) : DialogWrapper(true) {
    private lateinit var editor: Editor

    init {
        createEditorPanel()
        init()
        title = "Type a bunch of text into the editor in this dialog"
    }

    // More info on layout managers: https://docs.oracle.com/javase/tutorial/uiswing/layout/visual.html
    override fun createCenterPanel(): JComponent {
        val panel = JPanel()
        panel.preferredSize = JBUI.size(WIDTH, HEIGHT)
        panel.layout = GridLayout(0, 1)
        panel.add(editor.component)
        return panel
    }

    fun createEditorPanel() {
        val editorFactory: EditorFactory = EditorFactory.getInstance()
        val document: Document = editorFactory.createDocument("")
        editor = editorFactory.createEditor(document, project)
        val settings: EditorSettings = editor.settings
        with(settings) {
            isFoldingOutlineShown = false
            isLineMarkerAreaShown = false
            isIndentGuidesShown = false
            isLineNumbersShown = false
            isRightMarginShown = false
        }
        Disposer.register(myDisposable, Disposable {
            EditorFactory.getInstance().releaseEditor(editor)
        })
        pasteTextFromClipboard()
    }

    fun pasteTextFromClipboard() {
        getTextInClipboard()?.let(::setText)
    }

    fun setText(text: String) {
        val runnable = Runnable {
            ApplicationManager.getApplication().runWriteAction {
                editor.document.let {
                    it.replaceString(0, it.textLength, StringUtil.convertLineSeparators(text))
                }
            }
        }
        // Undoable command to set the text.
        CommandProcessor.getInstance().executeCommand(project, runnable, "", this)
    }

    private fun getTextInClipboard(): String? {
        return CopyPasteManager.getInstance().getContents(DataFlavor.stringFlavor)
    }

    companion object {
        const val WIDTH = 1000
        const val HEIGHT = 400
    }
}
```

### Adding your plugin UI in Tool windows

Tool windows provide in IDEA access to useful development tasks such as viewing your project structure, running and debugging your code, Git integration, and so on. Your plugin can create UI that will fit in tool windows in IDEA, instead of for example displaying it in a dialog box.

> Read more about Tool windows in the [official docs](https://www.jetbrains.com/help/idea/tool-windows.html).

For both of these types of Tool windows, the following applies:

1. Each tool window can have multiple tabs (aka “contents”).
2. Each side of the IDE can only show 2 tool windows at any given time, as the primary or the secondary. For eg: you can move the “Project” tool window to “Left Top”, and move the “Structure” tool window to “Left Bottom”. This way you can open both of them at the same time. Note that when you move these tool windows to “Left Top” or “Left Bottom” how they actually move to the top or bottom of the side of the IDE.

There are two main types of tool windows: 1) **Declarative**, and 2) **Programmatic**.

> Please take a look at the [idea-plugin-example2](https://github.com/nazmulidris/idea-plugin-example2) repo that contains a plugin that has some of the functionality shown below.

#### 1. Declarative tool window

Always visible and the user can interact with it at anytime (eg: Gradle plugin tool window).

1. This type of tool window must be registered in `plugin.xml` using the `com.intellij.toolWindow` extension point. You can specify things to register this in XML:
   * `id`: Text displayed in the tool window button.
   * `anchor`: Side of the screen in which the tool window is displayed (“left”, “right”, or “bottom”).
   * `secondary`: Specify whether it is displayed in the primary or secondary group.
   * `icon`: Icon displayed in the tool window button (`13px` x `13px`).
   * `factoryClass`: A class implementing `ToolWindowFactory` interface, which is used to instantiate the tool window when the user clicks on the tool window button (by calling `createToolWindowContent()`). Note that if a user does not interact with the button, then a tool window doesn’t get created.
   * For versions `2020.1` and later, also implement the `isApplicable(Project)` method if there’s no need to display a tool window for all projects. Note this condition is only evaluated the first time a project is loaded.

Here’s an example.

The factory class.

```
class DeclarativeToolWindowFactory : ToolWindowFactory, DumbAware {
  override fun createToolWindowContent(project: Project, toolWindow: ToolWindow) {
    val contentManager = toolWindow.contentManager
    val content = contentManager.factory.createContent(createDialogPanel(), null, false)
    contentManager.addContent(content)
  }
}

fun createDialogPanel(): DialogPanel = panel {
  noteRow("""Note with a link. <a href="http://github.com">Open source</a>""") {
    colorConsole {
      printLine {
        span(Colors.Purple, "link url: '$it' clicked")
      }
    }
    BrowserUtil.browse(it)
  }
}
```

The `plugin.xml` snippet.

```
<extensions defaultExtensionNs="com.intellij">
  <toolWindow
      icon="/icons/ic_toolwindow.svg"
      id="Declarative tool window"
      anchor="left"
      secondary="true"
      factoryClass="ui.DeclarativeToolWindowFactory" />
</extensions>
```

#### 2. Programmatic tool window

Only visible when a plugin creates it to show the results of an operation (eg: Analyze Dependencies action). This type of tool window must be added programmatically by calling `ToolWindowManager.getInstance().registerToolWindow(RegisterToolWindowTask)`.

A couple of things to remember.

1. You have to register the tool window (w/ the tool window manager) before using it. This is a one time operation. There’s no need to register the tool window if it’s already been registered. Registering simply shows the tool window in the IDEA UI. Unregistering removes it from the UI.
2. You can tell the tool window to auto hide itself when there are no contents inside of it.
3. You can create as many “contents” as you want and add it to the tool window. Each content is basically a tab. You can also specify that the content is closable.
4. You can also attach a disposer to a content so that you can take some action when the content or tab is closed. For eg you can just unregister the tool window when there are no contents left in the tool window.

Here’s an example of all of the things listed above.

```
internal class AnotherToolWindow : AnAction() {
  override fun actionPerformed(e: AnActionEvent) {
    val project: Project = e.getRequiredData(CommonDataKeys.PROJECT)
    val toolWindowManager = ToolWindowManager.getInstance(project)
    var toolWindow = toolWindowManager.getToolWindow(ID)

    // One time registration of the tool window (does not add any content).
    if (toolWindow == null) {
      toolWindow = toolWindowManager.registerToolWindow(
          RegisterToolWindowTask(
              id = ID,
              icon = IconLoader.getIcon("/icons/ic_extension.svg", javaClass),
              component = null,
              canCloseContent = true,
          ))
      toolWindow.setToHideOnEmptyContent(true)
    }

    val contentManager = toolWindow.contentManager
    val contentFactory: ContentFactory = ContentFactory.SERVICE.getInstance()
    val contentTab = contentFactory.createContent(
        createComponent(project),
        "${LocalDate.now()}",
        false)
    contentTab.setDisposer {
      colorConsole {
        printLine {
          span(Colors.Purple, "contentTab is disposed, contentCount: ${contentManager.contentCount}")
        }
      }
      if (contentManager.contents.isEmpty()) {
        toolWindowManager.unregisterToolWindow(ID)
      }
    }
    contentManager.addContent(contentTab)

    toolWindow.show { contentManager.setSelectedContent(contentTab, true) }
  }

  private fun createComponent(project: Project): JComponent {
    val panel = JPanel(BorderLayout())
    panel.add(BorderLayout.CENTER, JLabel("TODO add component"))
    return panel
  }

  companion object {
    const val ID = "AnotherToolWindow"
  }
}
```

The `plugin.xml` snippet, to register the action.

```
<actions>
  <action id="MyPlugin.AnotherToolWindow" class="actions.AnotherToolWindow" text="Open Tool Window"
      description="Opens tool window programmatically" icon="/icons/ic_extension.svg">
    <add-to-group group-id="EditorPopupMenu" anchor="first" />
  </action>
</actions>
```

#### Indices and dumb aware

Displaying the contents of many tool windows requires access to the indices. Because of that, tool windows are normally disabled while building indices, unless true is passed as the value of `canWorkInDumbMode` to the `registerToolWindow()` function (for programmatic tool windows). You can also implement `DumbAware` in your factory class to let IDEA know that your tool window can be shown while indices are being built.

#### Creating a content for any kind of tool window

Regardless of the type of tool window (declarative or programmatic) here is the sequence of operations that you have to perform in order to add a content:

1. Create the component / UI that you need for the content, ie, a Swing component (eg: `createDialogPanel()` above).
2. Add the component / UI to the content to the `ToolWindowManager` (programmatic) or the tool window `ContentManager` (declarative).

#### Content closeability

A plugin can control whether the user is allowed to close tabs either 1) **globally** or 2) **on a per content basis**.

1. **Globally**: This is done by passing the `canCloseContents` parameter to the `registerToolWindow()` function, or by specifying `canCloseContents="true"` in `plugin.xml`. The default value is `false`. Note that calling `setClosable(true)` on `ContentManager` content will be ignored unless `canCloseContents` is explicitly set.
2. **Per content basis**: This is done by calling `setCloseable(Boolean)` on each content object itself.

If closing tabs is enabled in general, a plugin can disable closing of specific tabs by calling `Content.setCloseable(false)`.

### Add Line marker provider in your plugin

Line marker providers allow your plugin to display an icon in the gutter of an editor window. You can also provide actions that can be run when the user interacts with the gutter icon, along with a tooltip that can be generated when the user hovers over the gutter icon.

> Please take a look at the [idea-plugin-example2](https://github.com/nazmulidris/idea-plugin-example2) repo that contains a plugin that has some of the functionality shown below.

In order to use line marker providers, you have to do two things:

1. Create a class that implements `LineMarkerProvider` that generates the `LineMarkerInfo` for the correct `PsiElement` that you want IDEA to highlight in the IDE.
2. Register this `provider` in `plugin.xml` and associate it to be run for a specific language.

When your plugin is loaded, IDEA will then run your line marker provider when a file of that language type is loaded in the editor. This happens in two passes for performance reasons.

1. IDEA will first call your provider implementation with the `PsiElements` that are currently visible.
2. IDEA will then call your provider implementation with the `PsiElements` that are currently hidden.

It is very important that you only return a `LineMarkerInfo` for the more specific `PsiElement` that you wish IDEA to highlight, as if you scope it too broadly, there will be scenarios where your gutter icon will blink! Here’s a detailed explanation as to why (a comment from the source for [LineMarkerProvider.java](https://github.com/JetBrains/intellij-community/blob/master/platform/lang-api/src/com/intellij/codeInsight/daemon/LineMarkerProvider.java) source file).

> Please create line marker info for leaf elements only - i.e. the smallest possible elements. For example, instead of returning method marker for `PsiMethod`, create the marker for the `PsiIdentifier` which is a name of this method.
> Highlighting (specifically, `LineMarkersPass`) queries all `LineMarkerProviders` in two passes (for performance reasons):
> 1. first pass for all elements in visible area
> 2. second pass for all the rest elements If provider returned nothing for both areas, its line markers are cleared.
> So imagine a `LineMarkerProvider` which (incorrectly) written like this:

```
class MyBadLineMarkerProvider implements LineMarkerProvider {
  public LineMarkerInfo getLineMarkerInfo(PsiElement element) {
    if (element instanceof PsiMethod) { // ACTUALLY DONT!
       return new LineMarkerInfo(element, element.getTextRange(), icon, null,null, alignment);
    }
    else {
      return null;
    }
  }
  ...
}
```

> Note that it create `LineMarkerInfo` for the whole method body. Following will happen when this method is half-visible (e.g. its name is visible but a part of its body isn’t):
> 1. the first pass would remove line marker info because the whole `PsiMethod` isn’t visible
> 2. the second pass would try to add line marker info back because `LineMarkerProvider` was called for the `PsiMethod` at last
> As a result, line marker icon will blink annoyingly. Instead, write this:

```
class MyGoodLineMarkerProvider implements LineMarkerProvider {
  public LineMarkerInfo getLineMarkerInfo(PsiElement element) {
    if (element instanceof PsiIdentifier &&
        (parent = element.getParent()) instanceof PsiMethod &&
        ((PsiMethod)parent).getMethodIdentifier() == element)) { // aha, we are at method name
         return new LineMarkerInfo(element, element.getTextRange(), icon, null,null, alignment);
    }
    else {
      return null;
    }
  }
  ...
}
```


### Example of a provider for Markdown language

Let’s say that for Markdown files that are open in the IDE, we want to highlight any lines that have links in them. We want an icon to show up in the gutter area that the user can see and click on to take some actions. For example, they can open the link.

### 1. Declare dependencies

Also, because we are relying on the `Markdown` plugin, in our plugin, we have to add the following dependencies.

To `plugin.xml`, we must add.

```
<!-- please see http://www.jetbrains.org/intellij/sdk/docs/basics/getting_started/build_number_ranges.html for description -->
<idea-version since-build="2020.1" until-build="2020.*" />
<!--
  Declare dependency on IntelliJ module `com.intellij.modules.platform` which provides the following:
  Messaging, UI Themes, UI Components, Files, Documents, Actions, Components, Services, Extensions, Editors
  More info: https://www.jetbrains.org/intellij/sdk/docs/basics/getting_started/plugin_compatibility.html
-->
<depends>com.intellij.modules.platform</depends>
<!-- Markdown plugin. -->
<depends>org.intellij.plugins.markdown</depends>
```

To `build.gradle.kts` we must add:

```
// See https://github.com/JetBrains/gradle-intellij-plugin/
intellij {
    // Information on IJ versions https://www.jetbrains.org/intellij/sdk/docs/reference_guide/intellij_artifacts.html
    // You can use release build numbers or snapshot name for the version.
    // 1) IJ Release Repository w/ build numbers https://www.jetbrains.com/intellij-repository/releases/
    // 2) IJ Snapshots Repository w/ snapshot names https://www.jetbrains.com/intellij-repository/snapshots/
    version = "2020.1" // You can also use LATEST-EAP-SNAPSHOT here.

    // Declare a dependency on the markdown plugin to be able to access the
    // MarkdownRecursiveElementVisitor.kt file. More info:
    // https://www.jetbrains.org/intellij/sdk/docs/basics/plugin_structure/plugin_dependencies.html
    // https://plugins.jetbrains.com/plugin/7793-markdown/versions
    setPlugins("java", "org.intellij.plugins.markdown:201.6668.27")
}
```

### 2. Register the provider in XML

The first thing we need to do is register our line marker provider in `plugin.xml`.

```
<extensions defaultExtensionNs="com.intellij">
  <codeInsight.lineMarkerProvider language="Markdown" implementationClass="ui.MarkdownLineMarkerProvider" />
</extensions>
```

### 3. Provide an implementation of LineMarkerProvider

Then we have to provide an implementation of `LineMarkerProvider` that returns a `LineMarkerInfo` for the most fine grained `PsiElement` that it successfully matches against. In other words, we can either match against the `LINK_DESTINATION` or the `LINK_TEXT` elements.

Here’s an example. For the string containing an inline `Markdown` link:

```
[`LineMarkerProvider.java`](https://github.com/JetBrains/intellij-community/blob/master/platform/lang-api/src/com/intellij/codeInsight/daemon/LineMarkerProvider.java)
```

This is what its PSI looks like:

```
ASTWrapperPsiElement(Markdown:Markdown:INLINE_LINK)(1644,1810)
        ASTWrapperPsiElement(Markdown:Markdown:LINK_TEXT)(1644,1671)
          PsiElement(Markdown:Markdown:[)('[')(1644,1645)
          ASTWrapperPsiElement(Markdown:Markdown:CODE_SPAN)(1645,1670)
            PsiElement(Markdown:Markdown:BACKTICK)('`')(1645,1646)
            PsiElement(Markdown:Markdown:TEXT)('LineMarkerProvider.java')(1646,1669)
            PsiElement(Markdown:Markdown:BACKTICK)('`')(1669,1670)
          PsiElement(Markdown:Markdown:])(']')(1670,1671)
        PsiElement(Markdown:Markdown:()('(')(1671,1672)
        MarkdownLinkDestinationImpl(Markdown:Markdown:LINK_DESTINATION)(1672,1809)
          PsiElement(Markdown:Markdown:GFM_AUTOLINK)('https://github.com/JetBrains/intellij-community/blob/master/platform/lang-api/src/com/intellij/codeInsight/daemon/LineMarkerProvider.java')(1672,1809)
        PsiElement(Markdown:Markdown:))(')')(1809,1810)
```

Here’s what the implementation of the line marker provider that matches `INLINE_LINK` might look like.

```
package ui

import com.intellij.codeInsight.daemon.LineMarkerInfo
import com.intellij.codeInsight.daemon.LineMarkerProvider
import com.intellij.openapi.editor.markup.GutterIconRenderer
import com.intellij.openapi.util.IconLoader
import com.intellij.psi.PsiElement
import com.intellij.psi.tree.TokenSet
import org.intellij.plugins.markdown.lang.MarkdownElementTypes

internal class MarkdownLineMarkerProvider : LineMarkerProvider {
  override fun getLineMarkerInfo(element: PsiElement): LineMarkerInfo<*>? {

    val node = element.node
    val tokenSet = TokenSet.create(MarkdownElementTypes.INLINE_LINK)
    val icon = IconLoader.getIcon("/icons/ic_linemarkerprovider.svg")

    if (tokenSet.contains(node.elementType))
      return LineMarkerInfo(element,
                            element.textRange,
                            icon,
                            null,
                            null,
                            GutterIconRenderer.Alignment.CENTER)

    return null
  }
}
```

You can add the `ic_linemarkerprovider.svg` icon here (create this file in the `$PROJECT_DIR/src/main/resources/icons/` folder.

```
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" height="13px" width="13px">
  <path d="M0 0h24v24H0z" fill="none" />
  <path
      d="M3.9 12c0-1.71 1.39-3.1 3.1-3.1h4V7H7c-2.76 0-5 2.24-5 5s2.24 5 5 5h4v-1.9H7c-1.71 0-3.1-1.39-3.1-3.1zM8 13h8v-2H8v2zm9-6h-4v1.9h4c1.71 0 3.1 1.39 3.1 3.1s-1.39 3.1-3.1 3.1h-4V17h4c2.76 0 5-2.24 5-5s-2.24-5-5-5z" />
</svg>
```

### 4. Provide a more complex implementation of LineMarkerProvider

The example we have so far, simply shows a gutter icon beside the lines in the editor window, that match our matching criteria. Let’s say that we want to show some relevant actions that can be performed on the `PsiElement(s)` that matched and are associated with the gutter icon. In this case we have to delve a little deeper into the `LineMarkerInfo` class.

If you look at [LineMarkerInfo.java](https://github.com/jetbrains/intellij-community/blob/master/platform/lang-api/src/com/intellij/codeInsight/daemon/LineMarkerInfo.java#L137), you will find a `createGutterRenderer()` method. We can actually override this method and create our own `GutterIconRenderer` objects that have an action group inside of them which will hold all our related actions.

The following class [RunLineMarkerProvider.java](https://github.com/jetbrains/intellij-community/blob/master/platform/execution-impl/src/com/intellij/execution/lineMarker/RunLineMarkerProvider.java#L115) actually provides us some clue of how to use all of this. In IDEA, when there are targets that you can run, a gutter icon (play button) that allows you to execute the run target. This class actually provides an implementation of that functionality. Using it as inspiration, we can create the more complex version of our line marker provider.

We are going to change our initial implementation of `MarkdownLineMarkerProvider` quite drastically. First we have to add a class that is our new `LineMarkerInfo` implementation called `RunLineMarkerInfo`. This class simply allows us to return an `ActionGroup` that we will now have to provide.

```
class RunLineMarkerInfo(element: PsiElement,
                        icon: Icon,
                        private val myActionGroup: DefaultActionGroup,
                        tooltipProvider: Function<in PsiElement, String>?
) : LineMarkerInfo<PsiElement>(element,
                               element.textRange,
                               icon,
                               tooltipProvider,
                               null,
                               GutterIconRenderer.Alignment.CENTER) {
  override fun createGutterRenderer(): GutterIconRenderer? {
    return object : LineMarkerGutterIconRenderer<PsiElement>(this) {
      override fun getClickAction(): AnAction? {
        return null
      }

      override fun isNavigateAction(): Boolean {
        return true
      }

      override fun getPopupMenuActions(): ActionGroup? {
        return myActionGroup
      }
    }
  }
}
```

Next, is the new version of `MarkdownLineMarkerProvider` class itself.

```
class MarkdownLineMarkerProvider : LineMarkerProvider {

  override fun getLineMarkerInfo(element: PsiElement): LineMarkerInfo<*>? {
    val node = element.node
    val tokenSet = TokenSet.create(MarkdownElementTypes.INLINE_LINK)
    if (tokenSet.contains(node.elementType))
      return RunLineMarkerInfo(element,
                               IconLoader.getIcon("/icons/ic_linemarkerprovider.svg"),
                               createActionGroup(element),
                               createToolTipProvider(element))
    else return null
  }

  private fun createToolTipProvider(inlineLinkElement: PsiElement): Function<in PsiElement, String> {
    val tooltipProvider =
        Function { element1: PsiElement ->
          val current = LocalDateTime.now()
          val formatter = DateTimeFormatter.ofLocalizedDateTime(FormatStyle.SHORT)
          val formatted = current.format(formatter)
          buildString {
            append("Tooltip calculated at ")
            append(formatted)
          }
        }
    return tooltipProvider
  }

  fun createActionGroup(inlineLinkElement: PsiElement): DefaultActionGroup {
    val linkDestinationElement =
        findChildElement(inlineLinkElement, MarkdownTokenTypeSets.LINK_DESTINATION, null)
    val linkDestination = linkDestinationElement?.text
    val group = DefaultActionGroup()
    group.add(OpenUrlAction(linkDestination))
    return group
  }

}
```

The `createActionGroup(...)` method actually creates an `ActionGroup` and adds a bunch of actions that will be available when the user clicks on the gutter icon for this plugin. Note that you can also add actions that are registered in your `plugin.xml` using something like this.

```group.add(ActionManager.getInstance().getAction("ID of your plugin action"))```

Finally, here’s the action to open a URL that is associated with the `INLINE_LINK` that is highlighted in the gutter.

```
class OpenUrlAction(val linkDestination: String?) :
  AnAction("Open Link", "Open URL destination in browser", IconLoader.getIcon("/icons/ic_extension.svg")) {
  override fun actionPerformed(e: AnActionEvent) {
    linkDestination?.apply {
      BrowserUtil.open(this)
    }
  }

}
```
