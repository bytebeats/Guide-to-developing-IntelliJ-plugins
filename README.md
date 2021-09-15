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
** Enable highlight (you might want to disable this when you’re done playing around w/ the tool window)
** Enable properties
** Enable scroll to source
** Enable scroll from source

A great example that can help us understand how PSI modification can work is taking a look at the built-in Markdown plugin actions themselves. There are a few actions in the toolbar of every Markdown editor: “toggle bold”, “toggle italic”, etc.

These are great to walk thru to understand how to make our own. The source files from intellij-community repo on github are:

* Files in [`plugins/markdown/src/org/intellij/plugins/markdown/ui/actions/styling/`](https://github.com/JetBrains/intellij-community/tree/master/plugins/markdown/src/org/intellij/plugins/markdown/ui/actions/styling)
* [`ToggleBoldAction.java`](https://github.com/JetBrains/intellij-community/blob/master/plugins/markdown/src/org/intellij/plugins/markdown/ui/actions/styling/ToggleBoldAction.java)
* [`BaseToggleStateAction.java`](https://github.com/JetBrains/intellij-community/blob/master/plugins/markdown/src/org/intellij/plugins/markdown/ui/actions/styling/BaseToggleStateAction.java)

The example we will build for this section entails finding hyperlinks and replacing the links w/ some modified version of the link string. This will require using a write lock to mutate the PSI in a cancellable action.

#### Generate PSI elements from text
