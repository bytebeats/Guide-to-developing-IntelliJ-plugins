## 动态插件

> 💡️  组件, 在本文档中表达的是 Component 的概念.

JetBrains 在 2020 年为平台 SDK 引入的最大变化之一就是[动态插件](https://plugins.jetbrains.com/docs/intellij/dynamic-plugins.html?from=jetbrains.org). 今后将禁止使用任何类型的组件.

原因如下:

1. 使用组件会导致插件无法加载(因为在 IDEA 本身启动时, 无法解除对已被类加载器加载的插件组件类的引用).
2. 而且, 它们还会影响启动性能, 因为代码不是在需要时被懒加载, 这将减慢 IDEA 的启动速度.
3. 由于将disposer附加到父类上, 父类的生命周期可能超过项目本身的生命周期, 因此即使在卸载后, 插件也可能会保留很长时间.

在新的动态世界中, 所有东西都是懒加载的, 并且可以被垃圾回收. 这里有更多关于[废弃组件](https://plugins.jetbrains.com/docs/intellij/plugin-components.html?from=jetbrains.org)的信息.

如果你习惯于使用组件, 就必须记住这样做的一些注意事项.

1. 这里有一份[来自 JetBrains 的非常简短的迁移指南](https://plugins.jetbrains.com/docs/intellij/plugin-components.html?from=jetbrains.org), 提供了将组件迁移为服务,[启动活动](https://www.plugin-dev.com/intellij/general/plugin-initial-load/), 监听器或扩展的一些要点.
2. 你必须为服务, 扩展或监听器(以前是组件)选择不同的父disposable. 你不能再将[Disposable](https://plugins.jetbrains.com/docs/intellij/disposers.html?from=jetbrains.org#choosing-a-disposable-parent)作用于项目, 因为插件可以在项目生命周期内卸载.
3. 不要缓存已注册扩展点的实现副本, 因为这些副本可能会因插件的动态性质而导致泄漏. 下面是关于[动态扩展点](https://plugins.jetbrains.com/docs/intellij/plugin-extension-points.html?from=jetbrains.org#dynamic-extension-points)的更多信息. 这些扩展点被标记为动态, 以便 IDEA 可以在需要时重新加载它们.
4. 请阅读[动态插件限制和故障排除指南](https://plugins.jetbrains.com/docs/intellij/dynamic-plugins.html?from=jetbrains.org), 它可能会在您将组件迁移为动态组件时派上用场.
5. 插件现在支持[自动重新加载](https://plugins.jetbrains.com/docs/intellij/ide-development-instance.html?from=jetbrains.org#enabling-auto-reload), 如果这会给你带来问题, 你可以禁用它.

### 扩展点 `postStartupActivity`, `backgroundPostStartupActivity` 可在项目加载时初始化插件

有 2 个扩展点可以实现这一功能: `com.intellij.postStartupActivity`和`com.intellij.backgroundPostStartupActivity`.

* [官方文档](https://plugins.jetbrains.com/docs/intellij/plugin-components.html?from=jetbrains.org)
* [使用示例](https://intellij-support.jetbrains.com/hc/en-us/community/posts/360002476840-How-to-auto-start-initialize-plugin-on-project-loaded-)

以下是使用`StartupActivity`的所有方法:

* 在项目打开时, 使用`postStartupActivity`在 EDT 上运行某些内容.
* 在项目打开期间, 使用实现了`DumbAware`的`postStartupActivity`在后台线程上运行某些内容, 并与其他的`DumbAware`的`postStartupActivity`并行. 当这些活动运行时, 索引处理尚未完成.
* 在项目打开约 5 秒后, 使用`postStartupActivity`在后台线程上运行.
* IntelliJ 平台代码库中关于这些[Startup Activity]的更多信息(https://github.com/JetBrains/intellij-community/blob/165e3b323c90e884972999e546f1e7085995ef7d/platform/service-container/overview.md).

> 💡 在迁移策略部分, 你将找到许多如何使用这些活动的示例.

### 轻服务

轻服务允许你使用注解将一个类声明为服务, 而无需在`plugin.xml`中创建相应的条目.

> 阅读有关轻服务的所有信息, 请点击[这里](https://plugins.jetbrains.com/docs/intellij/plugin-services.html?from=jetbrains.org#light-services).

下面是一些使用`@Service`注解的代码示例, 这些注解用于一个非常简单的服务, 该服务不面向项目或模块(但面向应用程序).

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

代码片段注意事项:
* 无需在`plugin.xml`中注册这些注解, 因此使用起来非常简单.
* 根据重载的构造函数, IDEA 会判断这是项目作用域, 模块作用域还是整个应用作用域的服务.
* 使用轻服务的唯一限制是它们必须是final(所有 Kotlin 类默认都是 final).

> ⚠️ 不鼓励使用模块作用域的轻服务, 也不支持使用这种服务.
> ⚠️ 你可能会发现自己正在寻找一个 `projectService` 声明, 该声明在`plugin.xml`中丢失了, 但仍可作为服务使用, 在这种情况下, 请确保在服务类`@Service`中使用了以下注解.

下面是针对项目作用域的轻服务的代码片段.

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

> 💡️ 你可以保留一个已打开项目的引用, 因为每个项目都会创建一个新的服务实例(对于项目作用域的服务而言).

### 迁移策略

移除组件并将其替换为服务, 启动活动, 监听器等的方法有很多. 以下是常见的重构策略列表, 你可以根据具体需要使用这些策略.

### 1. Component -> Service

在很多情况下, 你只需将组件替换为服务, 然后去掉项目的打开和关闭方法, 以及组件名称和处置组件的方法.

另一个需要注意的地方是, 确保所有的`getInstance()`方法都调用了`getService()`, 而不是`getComponent()`. 此外, 还要查看测试是否使用了`getComponent()`而不是`getService()`来获取已迁移组件的实例.

下面是一个 XML 片段, 显示了这种情况:

```<projectService serviceImplementation="MyServiceClass" />```

> 💡️ 如果使用的是轻服务, 则可以跳过在`plugin.xml`中注册服务类.

下面是服务的代码:

```
class MyServiceClass : Disposable {
  fun dispose() { /** Custom logic that runs when the project is closed. */ }

  companion object {
    @JvmStatic
    fun getInstance(project: Project) = project.getService(MyServiceClass::class.java)
  }
}
```

> 💡️ 如果在项目关闭时不需要在服务中执行任何自定义登录, 那么就没有必要实现`Disposable`方法, 只需删除`dispose()`方法即可.

> #### Dispose该服务并选择父Disposable
> 为了清理服务, 可以简单地实现`Disposable`接口, 并将清理逻辑放在`dispose()`方法中. 这应该足以应付大多数情况, 因为 IDEA 会[自动负责清理](https://plugins.jetbrains.com/docs/intellij/disposers.html?from=jetbrains.org#automatically-disposed-objects)服务实例.
> 1. 当 IDEA 关闭或提供服务的插件卸载时, 平台会自动dispose应用级服务.
> 2. 项目级服务会在项目关闭或插件卸载时自动dispose.
> 然而, 如果你还想对服务的Dispose时间进行更精细的控制, 可以使用`Disposer.register()`将`Project`或`Application`服务实例作为父参数传递.

> 💡️ 更多信息请查看[JetBrains关于选择父Disposable的官方文档](https://plugins.jetbrains.com/docs/intellij/disposers.html?from=jetbrains.org#choosing-a-disposable-parent).

> ### 总结一下
> 1. 对于插件整个生命周期所需的资源, 请使用应用级或项目级服务.
> 2. 对于在显示对话框时需要的资源, 请使用`DialogWrapper.getDisposable()`.
> 3. 对于显示工具窗口时所需的资源, 将实现`Disposable`的实例传递给`Context.setDisposer()`.
> 4. 对于生命周期较短的资源, 可使用`Disposer.newDisposable()`创建Disposable, 然后使用`Disposable.dispose()`手动处置.
> 5. 最后, 在传递我们自己的父对象时, 要小心 non-capturing-lambda.

### 2. Component -> postStartupActivity

这是一个非常简单的组件与`StartupActivity`的替换. `projectOpened()`中的逻辑只需进入`runActivity(project: Project)`方法即可. 组件 -> Service 中使用的方法仍然适用(删除不必要的方法并使用`getService()`调用).

### 3. Component -> postStartupActivity + Service

这是上述两种策略的组合. 这里有一种模式可以用来检测这种方法是否正确. 如果组件在`projectOpened()`中执行了一些逻辑, 而这些逻辑需要一个`Project`实例, 那么你可以执行以下操作:

1. 在`plugin.xml`文件中将组件设为服务. 同时, 添加一个`StartupActivity`.
2. 如果需要在组件被丢弃(项目关闭)时运行某些逻辑, 可以让组件实现`Disposable`, 而不是扩展`ProjectComponent`. 或者让它不实现任何接口或扩展任何类. 确保在构造函数中接受一个`Project`的参数.
3. 将`projectOpened()`方法重命名为`onProjectOpened()`. 将你在任何`init{}`块或任何其他构造函数中的任何逻辑添加到此方法中.
4. 创建一个`getInstance(project: Project)`函数, 从给定的项目中查找服务实例.
5. 创建一个启动活动内部类, 例如: `MyStartupActivity`, 该类只需调用`onProjectOpened()`.

```
<projectService serviceImplementation="MyServiceClass" />
<postStartupActivity implementation="MyServiceClass$MyStartupActivity"/>
```

以及 Kotlin 代码的变更:

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

许多组件只是在`projectOpened()`方法中订阅消息总线上的主题. 在这种情况下, 可以通过在模块的`plugin.xml`中(声明性地)注册一个[projectListener](https://plugins.jetbrains.com/docs/intellij/plugin-listeners.html?from=jetbrains.org)来完全取代组件.

下面是一个 XML 片段(放在`plugin.xml`中):

```
<listener class="MyListenerClass"
          topic="com.intellij.execution.testframework.sm.runner.SMTRunnerEventsListener"/>
<listener class="MyListenerClass"
          topic="com.intellij.execution.ExecutionListener"/>
```

以及监听器类本身:

```
class MyListenerClass(val project: Project) : SMTRunnerEventsAdapter(), ExecutionListener {}
```

### 5. Component -> projectListener + Service

有时, 组件可以用`Service`和`projectListener`来代替, 这只是将上述两种策略结合起来.

### 6. 删除 Component

在某些情况下, 组件可能已被弃用. 在这种情况下, 只需将其从相应模块的`plugin.xml`中删除, 就可以同时删除这些文件.

### 7. Component -> AppLifecycleListener

在某些情况下, 应用程序组件必须在 IDEA 启动时启动, 并在关闭时通知. 在这种情况下, 你可以使用[AppLifecycleListener](https://github.com/JetBrains/intellij-community/blob/master/platform/platform-impl/src/com/intellij/ide/AppLifecycleListener.java) 为 IDEA 附加一个[监听器](https://plugins.jetbrains.com/docs/intellij/plugin-listeners.html?from=jetbrains.org), 它只做[这样](https://plugins.jetbrains.com/docs/intellij/plugin-components.html?from=jetbrains.org)的事情.
