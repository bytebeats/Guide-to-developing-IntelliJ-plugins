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
