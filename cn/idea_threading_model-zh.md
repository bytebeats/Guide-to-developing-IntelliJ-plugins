## IDEA 线程模型

大多数情况下你在插件中写的代码都在主线程中执行. 一些操作, 诸如在 IDE 的"数据模型"(PSI, VFS, 项目根模型)中修改数据等, 必须在主线程中有序完成, 以避免竞争条件的发生. 这是[官方文档](https://plugins.jetbrains.com/docs/intellij/general-threading-rules.html#modality-and-invokelater)关于这些内容的详细说明.

然而在许多情况下, 插件中的部分代码需要在后台运行, 以免长时间运行的操作导致 UI 卡顿. 插件 SDK 恰好提供了一些策略来实现代码的后台运行. 然而, 后台线程运行却不应该使用`SwingUtilities.invokeLater()`.
1.  运行后台任务的方式之一是使用 `TransactionGuard.submitTransaction()`, 但是它却已经被废弃了.
2.  新的方式是使用 `ApplicationManager.getApplication().invokeLater()`.

> ### 废弃 API
> * 你可以找到主要废弃 API 的详细列表和在不同版本 IDEA 中变化[点击这里](https://plugins.jetbrains.com/docs/intellij/api-notable.html).
> * 你可以在 JB 平台代码库中通过 `@ApiStatus.ScheduledForRemoval(inVersion = <version>)` 查找废弃 API, 其中 `<version>` 可以是 `2020.1` 等.

然而, 我们能够准确地理解所有这些之前, 有几个重要概念我们需要理解: - “modality”和“写锁”和“写安全上下文”. 所以我们从 `submitTransaction()` 和 `invokeLater()` 开始接触每一个概念.

## 什么是 submitTransaction() ?

现在废弃的 `TransactionGuard.submitTransaction(Runnable)` 基本上是运行传入的 `Runnable`:
1.  在写安全的上下文中 (这能够保证在对话框打开的时候, 没有人能够通过 `SwingUtilities#invokeLater()`或者等价的 API 来执行意外的数据模型修改).
2.  在写线程中 (如EDT).

然而, `submitTransaction()` 并不能获得写锁或者开启写任务(如果需要修改 IDE 模型数据, 你必须通过代码显式地这些做).

> 写任务必须在写安全上下文中做些事情吗? 不. 然而, 这两个概念很容易混淆. 而且写安全上下文很容易被认为有写锁. 其实它并没有. 这两个事情是毫无关联的 -
> 1.  写安全上下文,
> 2.  写锁.
> 你可能处于写安全上下文但仍然需要获取写锁来修改 IDE 模型数据. 你也可以使用以下方式来检测执行上下文是否已经持有写锁:
> 1.  `ApplicationManager.getApplication().isWriteAccessAllowed()`.
> 2.  `ApplicationManager.getApplication().assertWriteAccessAllowed()`.

## 什么是写安全上下文?
来自 `TransactionGuard.java` [Java 文档](https://github.com/JetBrains/intellij-community/blob/master/platform/core-api/src/com/intellij/openapi/application/TransactionGuard.java#L25) 解释了什么写安全上下文:
* 是一种机制, 能够保证在对话框打开的时候, 没人能够通过 `SwingUtilities#invokeLater()` 或者类似 API 来执行意外的 IDE 模型数据修改.

写安全上下文的示例代码:

* `Application#invokeLater(Runnable, ModalityState)` calls with a modality state that’s either non-modal or was started inside a write-safe context. The use cases shown in the sections below are related to the non-modal scenario.
* Direct user activity processing (key/mouse presses, actions) in non-modal state.
* User activity processing in a modality state that was started (e.g. by showing a dialog or progress) in a write-safe context.

想了解更多信息关于如何处理在 EDT 上运行的代码, 请看下面:

* 运行在 EDT 上面的代码不必处于写安全上下文, 这就是为什么 `submitTransaction()` 提供了这种机制来执行给定的 `Runnable` (EDT 本身就是写安全的上下文).
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
