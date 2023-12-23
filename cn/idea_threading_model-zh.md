## IDEA 线程模型

> 本文中, **模态** 指的是 **Modality State**; **写安全上下文**指的是**Write-Safe Context**.

在大多数情况下, 你在插件中编写的代码都是在主线程上执行的. 某些操作, 例如更改 IDE 的`数据模型`(PSI, VFS, 项目根模型)中的任何内容, 都必须在主线程中完成, 以防止出现竞争条件. 以下是有关这方面的[官方文档](https://plugins.jetbrains.com/docs/intellij/general-threading-rules.html#modality-and-invokelater).

在很多情况下, 你的一些插件代码需要在后台线程中运行, 这样当发生长时间运行操作时, 用户界面就不会被冻结. 插件 SDK 有很多策略允许你这样做. 但是, 你不应该做的一件事是使用`SwingUtilities.invokeLater()`.
1.  运行后台任务的方法之一是使用`TransactionGuard.submitTransaction()`, 但该方法已被弃用.
2.  新方法是使用`ApplicationManager.getApplication().invokeLater()`.

> ### API 废弃
> * 你可以在[这里](https://plugins.jetbrains.com/docs/intellij/api-notable.html) 找到 IDEA 各个版本中主要 API 废弃和更改的详细列表.
> * 你可以通过查找`@ApiStatus.ScheduledForRemoval(inVersion = <version>)`搜索 JB 平台代码库中的废弃 API, 其中`<version>`可以是`2020.1`等.

不过, 在理解所有这一切的确切含义之前, 我们必须先理解一些重要的概念--`模态`, `写锁`和`写安全上下文`. 因此, 我们将从讨论`submitTransaction()`开始, 在使用`invokeLater()`的过程中逐一讨论这些概念.

## 什么是 `submitTransaction()` ?

现已废弃的`TransactionGuard.submitTransaction(Runnable)`基本上是在以下环境中运行传递的`Runnable`:

1. 写安全上下文(它只是确保在显示对话框时, 没有人能使用`SwingUtilities#invokeLater()`, 或类似的 API 执行意外的 IDE 模型数据更改).
2. 在写线程(如 EDT)中.

但是, `submitTransaction()`不会获取写锁或启动写操作(如果需要修改 IDE 模型数据, 必须由代码显式完成).

> 写操作与写安全上下文有什么关系? 没什么关系. 然而, 人们很容易混淆这两个概念, 认为写安全上下文有写锁; 其实不然. 这是两个不同的概念
> 1. 安全写上下文,
> 2. 写锁.
> 你可以处于写安全上下文中, 但仍需要获取写锁才能更改 IDE 模型数据. 你也可以使用以下方法来检查你的执行上下文是否已持有写锁:
> 1. `ApplicationManager.getApplication().isWriteAccessAllowed()`.
> 2. `ApplicationManager.getApplication().assertWriteAccessAllowed()`.

## 什么是写安全上下文?

来自`TransactionGuard.java`的[JavaDocs](https://github.com/JetBrains/intellij-community/blob/master/platform/core-api/src/com/intellij/openapi/application/TransactionGuard.java#L25) 解释了什么是写安全上下文:

* 一种机制, 可确保在显示对话框时, 没有人能够使用`SwingUtilities#invokeLater()`或类似方法执行意外的 IDE 模型数据更改.

下面是一些写安全上下文的示例:

* 使用非模态或在写安全上下文中启动的模态调用`Application#invokeLater(Runnable, ModalityState)`. 下面各节所示的用例与非模式情景有关.
* 非模态下的直接用户活动处理(按键/鼠标点击, 操作).
* 在模态下的用户活动处理, 该状态是在写安全上下文中启动的(例如, 通过显示对话框或进度).

以下是有关如何处理 EDT 上运行的代码的更多信息:

* 在 EDT 中运行的代码不一定是在写安全上下文中, 这就是为什么`submitTransaction()`提供了在写安全上下文(EDT 本身)中执行给定的`Runnable`的机制.
* 这也有一些例外情况, 例如, 在 EDT 上运行时, 在非模态下的操作中运行的代码也是在安全写上下文中运行的(如上所述).
* 来自[`@DirtyUI`注解](https://github.com/JetBrains/intellij-community/blob/master/platform/util/ui/src/com/intellij/ui/DirtyUI.java#L8) 的 JavaDocs 还提供了一些关于在 EDT 和写操作中运行的 UI 代码的更多信息.

```
/**
* <p>
* 此注解指定在 Swing 事件派发线程上运行并访问 IDE 模型(PSI 等)的代码. 
* 同时运行的代码. 
* <p>
* 默认情况下, 从 EDT 访问 IDE 模型是被禁止的, 但许多现有组件在设计时都没有这种限制. 
* 此类限制. 此类代码可使用此注解进行标记, 这将导致专用的工具程序
* 修改字节码, 以便在执行前获取写入意图锁, 并在执行后释放. 
* <p>
* 标记的方法将被修改, 以获取/释放 IW 锁. 标记的类将有一组预定义的方法
* 以同样的方式修改. 该方法列表可在 {@link com.intellij.ide.instrument.LockWrappingClassVisitor#METHODS_TO_WRAP} 中找到. 
*
* @see com.intellij.ide.instrument.WriteIntentLockInstrumenter
* @see com.intellij.openapi.application.Application
*/
```

以下是在 EDT 中运行代码的一些最佳实践:

1.  对 IDE 数据模型的任何写入都必须在写入线程(即 EDT)上进行.
2.  即使你可以安全地从 EDT 读取 IDE 模型数据(无需获取读锁), 你仍需要获取写锁才能修改 IDE 模型数据.
3.  在 EDT 中运行的代码必须显式地获取写锁(将代码封装在写操作中), 才能更改任何 IDE 模型数据. 这可以通过使用`ApplicationManager.getApplication().runWriteAction()`或`WriteAction run()/compute()`在写操作中封装代码来实现.
4.  如果代码在对话框中运行, 并且不需要对 IDE 数据模型执行任何写入操作, 则可以简单地使用`SwingUtilities.invokeLater()`或类似功能. 只有当你计划更改 IDE 数据模型时, 才需要在正确的模式下运行写安全上下文.
5.  IntelliJ Platform SDK 文档中有更多关于 IDEA 线程的详细信息, 请查看[这里](https://plugins.jetbrains.com/docs/intellij/general-threading-rules.html?from=jetbrains.org).

## 取代 `submitTransaction()` 的方法是什么?

替代`TransactionGuard.submitTransaction(Runnable, ...)`的方法是`ApplicationManager.getApplication().invokeLater(Runnable, ...)`. `invokeLater()`确保代码在写安全上下文中运行, 但如果要修改任何 IDE 模型数据, 仍需获取写锁.

## `invokeLater()` 和 `ModalityState`

`invokeLater()`除了可以接受一个`Runnable`参数外, 还可以接受一个`ModalityState`参数. 基本上, 你可以选择何时实际执行你的`Runnable`, 可以是尽快, 也可以是在显示对话框期间, 或者是在关闭所有对话框后.

> 要查看源代码示例, 了解如何在插件(使用 Kotlin UI DSL 编写)中的对话框中使用`ModailtyState`, 请查看此示例[ShowKotlinUIDSLSampleInDialogAction.kt](https://github.com/nazmulidris/idea-plugin-example/blob/main/src/main/kotlin/ui/ShowKotlinUIDSLSampleInDialogAction.kt).
> 要进一步了解官方文档对此的说明, 请查看`ModalityState.java`中的[JavaDocs](https://github.com/JetBrains/intellij-community/blob/master/platform/core-api/src/com/intellij/openapi/application/ModalityState.java#L23). 下面将深入解释这一切是如何工作的.


文档中描述的用户流程大致如下:
1.  在插件中运行某些操作, 然后使用`SwingUtilities.invokeLater()`将`Runnable A`在 EDT 上排队, 以便稍后调用.
2.  然后, 该操作会显示一个对话框, 等待用户输入.
3.  当用户正在考虑如何在对话框中做出选择时, `Runnable A`已经开始执行, 并对 IDE 数据模型做了一些大动作(如删除项目之类).
4.  用户最终下定决心并做出选择. 此时, 在操作中运行的代码并不知道`Runnable A`所做的更改. 这可能会给操作带来一些大问题.
5.  `ModalityState`允许你控制`Runnable A`在稍后时间的实际执行时间.

下面是关于这种情况的更多信息. 如果在动作开始执行时, 即在启动 `Runnable A` 之前设置一个断点, 那么在调试器中查看`ApplicationManager.getApplication().myTransactionGuard.myWriteSafeModalities`时, 你可能会看到以下内容. 这是在对话框显示**之前**的.

```
 myWriteSafeModalities = {ConcurrentWeakHashMap@39160}  size = 3
 {ModalityStateEx@39091} "ModalityState.NON_MODAL" -> {Boolean@39735} true
 {ModalityStateEx@41404} "ModalityState:{}" -> {Boolean@39735} true
 {ModalityStateEx@40112} "ModalityState:{}" -> {Boolean@39735} true
```

在对话框显示**之后**, 你可能会在调试器中看到类似下面的内容(如果你在对话框返回用户选择的值之后设置了断点).

```
 myWriteSafeModalities = {ConcurrentWeakHashMap@39160}  size = 4
 {ModalityStateEx@39091} "ModalityState.NON_MODAL" -> {Boolean@39735} true
 {ModalityStateEx@41404} "ModalityState:{}" -> {Boolean@39735} true
 {ModalityStateEx@43180} "ModalityState:{[...APPLICATION_MODAL,title=Sample..." -> {Boolean@39735} true
 {ModalityStateEx@40112} "ModalityState:{}" -> {Boolean@39735} true
```

注意已添加了一个新的`ModalityState`, 它基本上指向正在显示的对话框.
因此, `invokeLater()`的模态参数是这样工作的.

1.  如果不想立即执行传给它的`Runnable`, 则可以指定`ModalityState.NON_MODAL`, 这样它就会在对话框关闭后运行(活跃的模式对话框堆栈中不再有模式对话框).
2.  如果你想立即运行, 可以使用`ModalityState.any()`.
3.  但是, 如果你将对话框的`ModalityState`作为参数传递, 那么该`Runnable`只会在显示该对话框时执行.
4.  现在, 即使在对话框显示时, 如果你使用`ModalityState.NON_MODAL`指定了另一个`Runnable`来运行, 那么它将在对话框关闭后运行.

## 如何使用 `invokeLater` 执行异步和同步任务

以下子区域将演示如何在给定用例中放弃使用`submitTransaction()`, 转而使用`invokeLater()`, 甚至在可能的情况下完全取消使用它们.

### `Runnable` 的同步执行(从已经写安全上下文中使用 `submitTransaction()/invokeLater()`)

在代码已经在写安全上下文中运行的情况下, 无需使用`submitTransaction()`或`invokeLater()`来入队你的`Runnable`, 你可以直接执行其内容.
> 摘自`TransactionGuard.java#submitTransaction()`JavaDoc[第 76 行](https://github.com/JetBrains/intellij-community/blob/master/platform/core-api/src/com/intellij/openapi/application/TransactionGuard.java#L76): `在绝对写安全的上下文中, 只需用 `{@code transaction}` 内容替换此调用. 否则, 请用 `{@link Application#invokeLater}` 替换, 并注意默认或显式传递的模态是写安全的.`

假设在对话框或工具窗口中有一个按钮, 该按钮有一个监听器. 当按钮被点击时, 它应该在写安全上下文中执行一个`Runnable`. 下面是为按钮注册动作监听器的代码.

```
init {
  button.addActionListener {
    processButtonClick()
  }
}
```

下面是对`Runnable`进行入队的代码. 如果在以下函数中运行`TransactionGuard.getInstance().isWriteSafeModality(ModalityState.NON_MODAL)`会返回 true.

```
private fun processButtonClick() {
  TransactionGuard.submitTransaction(project, Runnable {
    // Do something to the IDE data model in a write action.
  })
}
```

然而, 这段代码是在 EDT 中运行的, 而且由于`processButtonClick()`使用写操作来执行其工作, 因此确实没有必要将其封装在`submitTransaction()`或`invokeLater()`调用中. 我们这里的代码是:

1.  运行 EDT,
2.  已在写安全上下文中运行,
3.  而且, `processButtonClick()` 本身将其工作封装在一个写操作中.

在这种情况下, 可以安全地移除`Runnable`并直接调用其内容. 因此, 我们可以用类似下面这样的代码来替换它.

```
ApplicationManager.getApplication().assertIsDispatchThread()
// Do something to the IDE data model in a write action.
```

### `Runnable` 的异步执行(在(不)写安全上下文中使用 `submitTransaction()`)

对于替换通常传递给`submitTransaction()`的`Runnable`和`Disposable`, 最简单的方法是用:

```
ApplicationManager.ApplicationManager.getApplication().invokeLater(
  Runnable, ComponentManager.getDisposed())
```

调用:

```
TransactionGuard.submitTransaction(Disposable, Runnable)
```

1. 请注意, IDE 数据模型对象(如`Project`等)都会通过调用`getDisposed()`暴露出一个`Condition`对象.
2. 如果你必须自己创建一个`Condition`对象, 那么你只需在调用`Disposer.isDisposed(Disposable)`时封装你的`Disposable`对象即可.

让我们来看一个替换使用`submitTransaction()`的旧代码的简单示例.

1.  使用`submitTransaction`的旧代码. <br>```TransactionGuard.submitTransaction(myProject, ()->{ /* Runnable */ }```
2.  新代码使用`invokeLater()`. <br>```ApplicationManager.getApplication().invokeLater(()->{ /* Runnable */ }, myProject.getDisposed());```

这是一个稍微复杂点的例子, 其中必须创建一个`Condition`对象.

1.  旧代码使用`submitTransaction`. <br>```TransactionGuard.submitTransaction(myComponent.getModel(), ()->{ /* Runnable */ })```
2.  新代码使用`invokeLater()`. <br>```ApplicationManager.getApplication().invokeLater(()->{ /* Runnable */ }, ignore -> Disposer.isDisposed(myComponent.getModel()));```
3.  你也可以在`Runnable`开启的地方检查条件`Disposer.isDisposed(myComponent.getModel())`. 甚至, 如果你喜欢的话, 可以在`invokeLater()`中不传递`Condition`.

### `invokeLater()` vs `submitTransaction()` 及其对测试代码的影响

`submitTransaction()`迁移到`invokeLater()`时, 由于这两个函数实际工作方式的一个主要区别, 可能需要更新一些测试.

在某些情况下, 使用`invokeLater()`而非`submitTransaction()`的后果之一是, 在测试中, 你可能必须调用`PlatformTestUtil.dispatchAllInvocationEventsInIdeEventQueue()`以确保在 EDT 上排队等待执行的`Runnable` 被实际执行(因为测试是在无头环境中运行的). 你还可以调用`UIUtil.dispatchAllInvocationEvents()`, 而不是`PlatformTestUtil.dispatchAllInvocationEventsInIdeEventQueue()`. 过去使用`submitTransaction()`时, 无需这样做.

下面是旧代码和测试代码:

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

下面是新代码和测试代码:

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
