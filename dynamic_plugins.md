## Dynamic plugins

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

### Extension points postStartupActivity, backgroundPostStartupActivity to initialize a plugin on project load

There are 2 extension points to do just this `com.intellij.postStartupActivity` and `com.intellij.backgroundPostStartupActivity`.

* [Official doc](https://plugins.jetbrains.com/docs/intellij/plugin-components.html?from=jetbrains.org)
* [Usage samples](https://intellij-support.jetbrains.com/hc/en-us/community/posts/360002476840-How-to-auto-start-initialize-plugin-on-project-loaded-)

Here are all the ways in which to use a `StartupActivity`:

* Use a `postStartupActivity` to run something on the EDT during project open.
* Use a `postStartupActivity` implementing `DumbAware` to run something on a background thread during project open in parallel with other dumb-aware post-startup activities. Indexing is not complete when these are running.
* Use a `backgroundPostStartupActivity` to run something on a background thread approx 5 seconds after project open.
* More information from the IntelliJ platform codebase about these [startup activities](https://github.com/JetBrains/intellij-community/blob/165e3b323c90e884972999e546f1e7085995ef7d/platform/service-container/overview.md).

> 💡 You wil find many examples of how these can be used in the migration strategies section.

### Light services

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

### Migration strategies

There are a handful of ways to go about removing components and replacing them w/ services, startup activities, listeners, etc. The following is a list of common refactoring strategies that you can use depending on your specific needs.

### 1. Component -> Service

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

> ### Summary
> 1. For resources required for the entire lifetime of a plugin use an application-level or project-level service.
> 2. For resources required while a dialog is displayed, use a `DialogWrapper.getDisposable()`.
> 3. For resources required while a tool window is displayed, pass your instance implementing `Disposable` to `Context.setDisposer()`.
> 4. For resources w/ a shorter lifetime, create a disposable using a `Disposer.newDisposable()` and dispose it manually using `Disposable.dispose()`.
> 5. Finally, when passing our own parent object be careful about non-capturing-lambdas.

### 2. Component -> postStartupActivity

This is a very straightforward replacement of a component w/ a startup activity. The logic that is in `projectOpened()` simply goes into the `runActivity(project: Project)` method. The same approach used in Component -> Service still applies (w/ removing needless methods and using `getService()` calls).

### 3. Component -> postStartupActivity + Service

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

### 4. Component -> projectListener

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

### 5. Component -> projectListener + Service

Sometimes a component can be replaced w/ a service and a `projectListener`, which is simply combining two of the strategies shown above.

### 6. Delete Component

There are some situations where the component might have been deprecated already. In this case simply remove it from the appropriate module’s `plugin.xml` and you can delete those files as well.

### 7. Component -> AppLifecycleListener

There are some situations where an application component has to be launched when IDEA starts up and it has to be notified when it shuts down. In this case you can use [AppLifecycleListener](https://github.com/JetBrains/intellij-community/blob/master/platform/platform-impl/src/com/intellij/ide/AppLifecycleListener.java) to attach a [listener](https://plugins.jetbrains.com/docs/intellij/plugin-listeners.html?from=jetbrains.org) to IDEA that does just [this](https://plugins.jetbrains.com/docs/intellij/plugin-components.html?from=jetbrains.org).
