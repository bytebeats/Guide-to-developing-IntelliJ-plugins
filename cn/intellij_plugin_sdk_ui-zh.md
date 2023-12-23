## UI (JetBrains UI 组件, Swing 和 Kotlin UI DSL)

插件与终端用户交互的方式有很多种 -- 与操作绑定的键盘快捷方式, 以及插件提供的 UI 组件, 这些组件可以完全自定义, 也可以集成到现有集成开发环境的 UI 组件中. JetBrains 还提供了许多适用于不同场景的 UI 控件.

从简单的发送通知和忽略通知, 到可以与用户进行密切交互的更复杂的 UI , 这些组件应有尽有.  UI 组件甚至还有一个 Kotlin DSL, 这使得在 IDEA 中创建表格变得相对容易.

甚至还有一种使用 Swing 创建表格的方法, 但这种方法正逐渐被淘汰, 取而代之的是 Kotlin UI DSL. 除此之外, 你甚至还可以为 IDEA 创建自定义主题.

为插件编写 UI 代码的一个有效方法是看看`intellij-community`仓库中的 UI 代码是如何编写的, 看看 JetBrains 是如何做的. 文档很少, 甚至根本不存在, 有时最好的办法是查看 IDEA 本身的源代码, 了解某些 UI 是如何创建的.

在考虑为插件编写 UI 代码时, 可以参考以下资源.

* [Swing 布局管理器](https://docs.oracle.com/javase/tutorial/uiswing/layout/visual.html)
* [Swing 简介](https://docs.oracle.com/javase/tutorial/uiswing/index.html)
* [在 Kotlin UI DSL 中使用的MigLayout](https://github.com/mikaelgrev/miglayout)

### 简单的 UI 组件 - 通知, 弹出窗口, 对话框

本节将介绍一些简单的 UI 组件, 接下来的章节将介绍与用户之间更复杂的交互(通过表单, 对话框和设置 UI)

#### 通知

通知是向用户提供某些信息的一种非常简单的方式. 如果希望用户在收到通知时执行操作, 您甚至可以为通知附加操作.

* 通知会显示在集成开发环境的`Event Log`工具窗口中.
* 你可以选择将任何显示的通知也记录在案.
* 通知可以是针对特定项目的(并显示在包含项目的集成开发环境窗口中), 也可以是针对整个集成开发环境的一般通知.
* 以下是关于通知的[官方文档](https://plugins.jetbrains.com/docs/intellij/notifications.html?from=jetbrains.org).

这是一个简单操作的示例, 使用略有不同的方式显示两个通知. 下面是该操作在`plugin.xml`中的代码段.

```
<actions>
  <!-- Add notification action. -->
  <action id="MyPlugin.actions.ShowNotificationSample" class="ui.ShowNotificationSampleAction"
      description="Show sample notifications" text="Sample Notification" icon="/icons/ic_extension.svg" />
</actions>
```

下面是实现此操作的类的开头, 以便进行设置.

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

方法 1 - 下面是使用`Notifications.Bus`上的静态方法显示通知的示例.

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

方法 2 - 下面是另一种通过创建通知组来显示通知的方法.

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

#### 弹出窗口

弹框是一种获取用户输入信息的方式, 而不会打断用户正在进行的操作. 弹出式窗口显示时不使用依赖任何 UI 以关闭弹出式窗口, 而是在失去键盘焦点时会自动消失. IDE在键入超前补全, 自动补全等功能中广泛使用弹出窗口.

* 弹出窗口允许用户从显示的选项列表中做出单项选择. 一旦用户做出选择, 您可以提供一些代码(`itemChosenCallback`)来运行该选择.
* 弹出窗口可以显示标题.
* 可以移动和调整大小(并支持记忆大小).
* 它们甚至可以嵌套(当一个项目被选中时显示另一个弹出窗口).

下面是一个显示两个嵌套弹出窗口的操作示例. 下面是`plugin.xml`中用于注册该操作的代码段.

```
<!-- Add popup action. -->
<action id="MyPlugin.actions.ShowPopupSample" class="ui.ShowPopupSampleAction" text="Sample Popup"
    description="Shows sample popups" icon="/icons/ic_extension.svg" />
```

下面是实现该操作(显示一个弹出窗口, 当用户在第一个弹出窗口中选择一个选项时, 第二个弹出窗口就会显示)的类. 它展示了创建弹出窗口以显示项目列表的多种方法. 一种方法是使用`createPopupChooserBuilder()`, 另一种方法是使用`createListPopup()`, 这两种方法都是[JBPopupFactory.getInstance()](https://github.com/JetBrains/intellij-community/blob/master/platform/platform-api/src/com/intellij/openapi/ui/popup/JBPopupFactory.java)方法返回的类的方法.

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

#### 对话框

当你的插件需要用户输入时, 对话框可能是合适的组件. 与通知和弹出窗口不同, 对话框是模态的. IDEA 允许从对话框返回简单的布尔值响应.

为了在对话框中创建复杂的 UI , 最好使用 Kotlin UI DSL.

下面是一个非常简单的操作示例, 对话框显示"按确定或取消", 允许用户按确定和取消按钮. 下面是`plugin.xml`的代码段.

```
<!-- Add dialog action. -->
<action id="MyPlugin.actions.ShowDialogSample" class="ui.ShowDialogSampleAction" text="Sample Dialog"
    description="Show sample dialog" icon="/icons/ic_extension.svg" />
```

下面是使用[DialogWrapper](https://github.com/JetBrains/intellij-community/blob/master/platform/platform-api/src/com/intellij/openapi/ui/DialogWrapper.java)类实际显示对话框的操作.

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

请注意, 在本例中, `showAndGet()`既用于显示对话框, 也用于获取响应. 你也可以使用`show()`和`isOK()`将其分为两步.

> 有关对话框的更多信息, 请参阅[官方文档](https://github.com/JetBrains/intellij-community/blob/master/platform/platform-api/src/com/intellij/openapi/ui/DialogWrapper.java).
> 有关 Swing 布局管理器的更多信息, 请参阅此[Java辅导文章](https://docs.oracle.com/javase/tutorial/uiswing/layout/visual.html).

### 使用 Kotlin UI DSL 创建复杂的 UI (表单, 对话框, 设置屏幕)

在下面的章节中, 我们将使用`DialogWrapper`来获取使用 Kotlin UI DSL 创建的 UI 并将其展示出来.

为了创建更复杂的 UI , 你可以使用 Kotlin UI DSL. 你可以在 IDEA 设置 UI 中显示这些表单, 也可以直接在对话框中显示. 你还可以将表单绑定到数据对象, 这样你就可以在 IDEA 重启时持久保存这些数据对象.

#### 理解 DSL 的结构(布局, 行和单元格)

Kotlin UI DSL 允许以包含行和单元格的布局来表达 UI. 每一行中的内容都是从左到右对齐的, 但你也可以指定某个项目是否需要右对齐. IDE中的超前类型完成也为如何使用该 DSL 提供了指导.

下面是一个使用此 DSL 的简单 UI 示例, 其中包含以下 UI 组件.

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

#### 如何将数据绑定到表单 UI

将 UI 与你创建的数据对象绑定的一个非常简单的方法是, 为每个应由 UI 组件呈现的属性传递对`KProperty`的引用. 这样可以确保:

1. 在显示 UI 组件之前, 数据对象会将其填充到正确的初始状态.
2. 当用户操作 UI 组件并使其状态发生变化时, 数据对象的属性也会随之发生变化.

在上面的示例中, 需要注意的是, 某些保存状态信息的 UI 组件需要绑定一个 Kotlin 属性(来自你提供的数据对象). 这是 DSL 在表单中为你处理数据绑定的一种简单方法.

你还可以提供显式的setter和getter, 而不是仅仅使用[KProperty](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/-k-property/). 这使得创建表格变得非常简单, 因为状态会自动加载和保存.

#### 表单中可使用哪些 UI 元素

1. `panel`函数会创建一个[LayoutBuilder](https://github.com/JetBrains/intellij-community/blob/master/platform/platform-impl/src/com/intellij/ui/layout/LayoutBuilder.kt), 你可以在其中添加行(或`noteRow`或`titledRow`等). 请查看[RowBuilder](https://github.com/JetBrains/intellij-community/blob/master/platform/platform-impl/src/com/intellij/ui/layout/Row.kt), 了解你可以在此添加的所有基于行的组件.
2. 要查看每个`row`内可以添加哪些对象, 请查看[Cell](https://github.com/JetBrains/intellij-community/blob/master/platform/platform-impl/src/com/intellij/ui/layout/Cell.kt). 你可以添加诸如`browserLink`,`comboBox`,`checkBox`等对象.

> 请务必阅读有关 DSL 的[官方文档](https://plugins.jetbrains.com/docs/intellij/kotlin-ui-dsl.html?from=jetbrains.org).

#### 如何在对话框中显示表单

使用此 DSL 创建的表单可以显示在任何可以使用`DisplayPanel`的地方(它只是 `JPanel` 的子类). 这包括设置 UI (如下节所示)和对话框. 下面是一个示例操作, 它使用上述`createDialogPanel()`返回的对象, 并使用`DialogWrapper`将其显示在对话框中.

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

#### 如何在 IDE 重启之间持久化用户创建/选择的数据(`PersistentStateComponent`)

如果你希望在表单中创建的数据能在IDE的其他部分中使用, 那么持久化这些数据并将其提供给插件的其他部分就很有意义. 这就是[PersistentStateComponent](https://github.com/JetBrains/intellij-community/blob/master/platform/projectModel-api/src/com/intellij/openapi/components/PersistentStateComponent.java) 的作用所在. 它允许你将数据对象的状态序列化到磁盘, 并从磁盘反序列化. 要让 IDEA 持久化数据对象, 只需做两件事:

1. 让你的数据类扩展`PersistentStateComponent`.
2. 将其封装在一个`Service`中, 这样你就可以在插件中的任何代码中访问它.

下面是一个示例. `KotlinDSLUISampleService`服务包含一个名为`myState`的`State`对象, 它实际上保存着绑定到表单的数据. `State` 包含 4 个属性(可绑定到表单中的各种 UI 组件): `myFlag: Boolean`,`myString: String`,`myInt: Int`,`myStringChoice: String`.

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

在上面的表单 UI 示例中, 我们只是将 UI 组件的数据绑定到一个由简单数据类创建的对象上. 下面是一个示例.

```
row {
  label("a string")
  textField(myDataObject::myString)
}
```

为了用`PersistentStateComponent`实例来代替它, 我们可以这样做.

```
row {
  label("a string")
  textField(KotlinDSLUISampleService.instance.myState::myString)
}
```

换句话说, `myDataObject`将被替换为`KotlinDSLUISampleService.instance.myState`.

上面的代码中有一点非常重要. 你的代码不应调用`getState()`和`loadState(...)`方法. 这些是 IDEA 调用服务的钩子, 以便管理加载和保存状态到持久化中. 你应该为你的数据属性提供不涉及使用`getState()`的访问器. 并确保这些访问器可以处理提供"初始"状态的`loadState(...)`(如果在持久化中存储了任何内容). 而且必须确保默认情况下必须定义初始状态(如果没有从持久化中加载自定义数据, 则状态将包含默认值). IDEA 使用默认状态来判断用户是否更改了表单 UI 中的任何内容(差异将偏离默认状态).

此外, 你还可以向 IDEA 指定许多选项, 以确定在何处存储持久化数据. 在本例中, 我们告诉 IDEA 将用户修改的数据存储到 IDEA 设置所在的`config/options/`文件夹中名为`kotlinDSLUISampleData.xml`的 XML 文件中. 你还可以为`storages`指定选项, 以确定是否应禁用这些数据的漫游, 甚至是否只应在特定平台上存储数据.

#### 表单 UI 示例

下面是一个使用 Kotlin DSL 和`State`对象(由`PersistentStateComponent`服务提供)的表单.

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

#### MigLayout, panel, 和 grow

`panel`函数(在 Kotlin UI DSL 中)实际上创建了一个[MigLayout](https://github.com/jetbrains/intellij-community/blob/master/platform/platform-impl/src/com/intellij/ui/layout/migLayout/patched/MigLayout.kt#L43). `MigLayout`可以生成流动式, 网格式, 绝对式(带链接), 分组式和停靠式布局.

* `面板`接受`row`函数, 而`row`又接受`cell`函数.
* 可以向`panel`和`cell`函数传递`grow`和`fill`属性.
* 还可以使用`component`函数包装`JComponents`, 该函数也可以接收`grow`和`fill`属性.
* `panel`会自动调整对话框的大小, 不过, 你可以使用自己预定义的`width`和`height`来覆盖它.

下面是一个示例(`myJComponent`只是一个在其他地方创建的`JComponent`).

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
> 请注意,`JComponent`对象可以在`panel`函数中调用, 而且它们接受限制其增长的`CCFLags`.

> 请注意, 有时直接使用[Swing 布局管理器](https://docs.oracle.com/javase/tutorial/uiswing/layout/visual.html) 而不使用此 DSL 可能会更简单.

### 为插件创建IDE设置 UI

比方说, 我们想显示上面创建的表单(通过`createDialogPanel()`), 而且除了在`Dialog`中可用外, 我们还想在设置 UI 中显示它. 请注意, 你可以在这两个地方显示表单, 并让它在同一个`PersistentStateComponent`服务中更新数据, 这真的很方便.

为了告诉 IDEA 你希望在 设置 UI 中显示表单, 你可以使用两个扩展点之一. 在`plugin.xml`中, 你可以声明两种类型的`configurables`(可配置的), 以便自定义 IDEA 设置 UI, 其中`configurables`是 JetBrains 平台提供的一个基类, 它允许你显示表单 UI.

1. `projectConfigurable` -- 这些是特定项目的设置.
2. `applicationConfigurable` -- 这些设置适用于整个IDE.

参考资料:
* [旧 JB 该校](https://plugins.jetbrains.com/docs/intellij/settings.html).
* [新 JB 文档](https://plugins.jetbrains.com/docs/intellij/kotlin-ui-dsl.html?from=jetbrains.org#configurables).
* [MacOS 暗黑模式同步插件源代码](https://github.com/gilday/dark-mode-sync-plugin)

当使用 Kotlin UI DSL 而不是实现`Configurable`接口时, 只需扩展`BoundConfigurable`. 这样做时, 你可以:

1. 向`BoundConfigurable`的构造函数传递一个`displayName`名称, 该名称将实际显示在设置 UI 中(你可以提前键入搜索).
2. 在构造函数中传递任意数量的 JB 平台对象, 例如:
```
class MyConfigurable(private val lafManager: LafManager) : BoundConfigurable("Display Name")
```

下面是一个例子(`plugin.xml`条目和一个扩展`BoundConfigurable`的类).

```
<!-- Add Settings Dialog that is similar to what ShowKotlinUIDSLSampleAction does. -->
<extensions defaultExtensionNs="com.intellij">
  <applicationConfigurable instance="ui.KotlinDSLUISampleConfigurable" />
</extensions>
```

这里使用上一节(Kotlin UI DSL)中的代码来生成表单本身.

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

### 在对话框中创建复杂的 UI

在某些情况下, 需要在`DialogWrapper`中创建复杂的 UI 组件. 在这种情况下, 可以使用 Kotlin UI DSL 或直接使用 Swing 布局管理器创建 Swing 组件.

> 请查看[idea-plugin-example2](https://github.com/nazmulidris/idea-plugin-example2) 仓库, 其中的插件具有下图所示的部分功能.

下面是一个示例, 用于创建一个对话框, 允许用户将剪贴板中的文本粘贴到`Editor`组件中. 粘贴操作实际上是作为一个可撤销的命令来实现的. 当对话框显示时, 它会自动将剪贴板中的任何文本粘贴到编辑器中.

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

### 在Tool窗口中添加插件 UI

在 IDEA 中, Tool窗口提供了对有用的开发任务的访问, 如查看项目结构, 运行和调试代码, Git 集成等. 你的插件可以创建适合 IDEA 工具窗口的 UI , 而不是在对话框中显示.

> 在[官方文档](https://www.jetbrains.com/help/idea/tool-windows.html) 中阅读更多关于工具窗口的内容.

对于这两种类型的工具窗口, 以下内容都适用:

1. 每个Tool窗口可以有多个标签页(又称"content).
2. IDE的每一侧在任何时候都只能显示 2 个工具窗口, 分别为主窗口或辅助窗口. 例如: 可以将"Project"工具窗口移到"左上角", 将"Structure"工具窗口移到"左下角". 这样就可以同时打开这两个工具窗口. 请注意, 将这些工具窗口移动到"左上"或"左下"时, 它们实际上是移动到IDE的顶部或底部.

工具窗口主要有两种类型 1) **声明式**, 和 2) **编程式**.

> 请查看[idea-plugin-example2](https://github.com/nazmulidris/idea-plugin-example2) 仓库, 其中包含一个插件, 具有下图所示的部分功能.

#### 1. 声明式Tool窗口

始终可见, 用户可随时与之交互(例如: Gradle 插件工具窗口).

1. 这类工具窗口必须使用`com.intellij.toolWindow`扩展点在`plugin.xml`中注册. 你可以在 XML 中指定一些内容来注册:
   * `id`: 工具窗口按钮中显示的文本.
   * `anchor`: 显示工具窗口的屏幕侧面("左","右"或"底").
   * `secondary`: 指定显示在主组还是次组中.
   * `icon`: 工具窗口按钮中显示的图标(`13px` x `13px`).
   * `factoryClass`: 实现`ToolWindowFactory`接口的类, 当用户点击工具窗口按钮时(通过调用`createToolWindowContent()`), 该类用于实例化工具窗口. 请注意, 如果用户没有与按钮交互, 则不会创建工具窗口.
   * 对于`2020.1`及更高版本, 如果不需要为所有项目显示工具窗口, 还需执行`isApplicable(Project)`方法. 请注意, 只有在首次加载项目时才会评估该条件.

下面是一个示例.

工厂类如下.

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

`plugin.xml`代码段.

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

#### 2. 编程式Tool窗口

只有当插件创建它以显示操作结果(如: 分析依赖关系操作)时才可见. 这类工具窗口必须通过调用`ToolWindowManager.getInstance().registerToolWindow(RegisterToolWindowTask)`以编程方式添加.

请记住以下几点:

1. 在使用工具窗口之前, 你必须注册该工具窗口(通过Tool Window Manager). 这是一次性操作. 如果Tool窗口已经注册, 则无需再注册. 注册只是在 IDEA UI 中显示工具窗口. 取消注册则会将其从 UI 中删除.
2. 你可以告诉Tool窗口在没有内容时自动隐藏.
3. 你可以创建任意数量的"Content"并将其添加到Tool窗口中. 每个内容基本上都是一个选项卡. 你还可以指定内容是否可以关闭.
4. 你还可以为内容附加一个处理程序, 以便在内容或标签关闭时采取一些措施. 例如, 当Tool窗口中没有内容时, 就可以取消工具窗口的注册.

下面是一个包含上述所有内容的示例.

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

用于注册操作的`plugin.xml`代码段.

```
<actions>
  <action id="MyPlugin.AnotherToolWindow" class="actions.AnotherToolWindow" text="Open Tool Window"
      description="Opens tool window programmatically" icon="/icons/ic_extension.svg">
    <add-to-group group-id="EditorPopupMenu" anchor="first" />
  </action>
</actions>
```

#### 索引 和 DumbAware

显示许多工具窗口的内容需要访问索引. 因此, 除非将`canWorkInDumbMode`的值作为 `true` 传递给`registerToolWindow()`函数(用于编程工具窗口), 否则在建立索引时, 工具窗口通常是禁用的. 你还可以在工厂类中实现`DumbAware`, 让 IDEA 知道在构建索引时可以显示工具窗口.

#### 为任何类型的工具窗口创建内容

无论工具窗口的类型如何(声明式或编程式), 以下是添加内容时必须执行的操作顺序:

1. 创建内容所需的组件/ UI , 即 Swing 组件(如上文的`createDialogPanel()`).
2. 将组件/ UI 添加到内容的`ToolWindowManager`(编程式)或工具窗口的`ContentManager`(声明式)中.

#### 内容可关闭性

插件可控制是否允许用户关闭标签:1)**全局的**或 2)**基于内容基础**.

1. **全局的**: 通过向`registerToolWindow()`函数传递`canCloseContents`参数, 或在`plugin.xml`中指定`canCloseContents=`true``来实现. 默认值为`false'. 请注意, 除非明确设置了`canCloseContents`, 否则在`ContentManager`内容上调用`setClosable(true)`将被忽略.
2. **基于内容基础**: 这是通过在每个内容对象上调用`setCloseable(Boolean)`来实现的.

如果在一般情况下启用了关闭标签, 插件可以通过调用`Content.setCloseable(false)`来禁用特定标签的关闭.

### 在插件中添加行标记提供器

行标记提供器允许插件在编辑器窗口的沟槽中显示图标. 你还可以提供在用户与沟槽图标交互时运行的操作, 以及在用户将鼠标悬停在沟槽图标上时生成的工具提示.

> 请查看[idea-plugin-example2](https://github.com/nazmulidris/idea-plugin-example2) 仓库, 其中的插件具有下图所示的部分功能.

要使用行标记提供器, 你必须做两件事:
1. 创建一个实现`LineMarkerProvider`的类, 该类将为你希望 IDEA 在 IDE 中突出显示的正确`PsiElement`生成`LineMarkerInfo`.
2. 在`plugin.xml`中注册此`provider`并将其关联到特定语言中运行.

加载插件后, 当编辑器加载该语言类型的文件时, IDEA 将运行行标记提供器. 出于性能考虑, 这将分两次进行.

1. IDEA 将首先使用当前可见的`PsiElements`调用你的提供器实现.
2. 然后, IDEA 将使用当前隐藏的`PsiElements`调用你的提供器实现.

非常重要的一点是, 你只能为希望 IDEA 高亮显示的更具体的`PsiElement`返回一个`LineMarkerInfo`, 因为如果范围太广, 就会出现沟槽图标闪烁的情况! 下面是对原因的详细解释([LineMarkerProvider.java](https://github.com/JetBrains/intellij-community/blob/master/platform/lang-api/src/com/intellij/codeInsight/daemon/LineMarkerProvider.java) 源文件中的评论).

> 请仅为叶元素创建行标记信息, 即尽可能小的元素. 例如, 与其为`PsiMethod`返回方法标记, 不如为该方法的名称`PsiIdentifier`创建标记.
> 高亮显示(特别是`LineMarkersPass`)分两次查询所有`LineMarkerProviders`(出于性能考虑):
> 1. 第一次查询可见区域内的所有元素
> 2. 第二次查询其余所有元素 如果提供器在两个区域都没有返回任何信息, 则会清除其行标记.
     > 因此, 试想一下,`LineMarkerProvider`(不正确地)是这样编写的:
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

> 请注意, 它为整个方法体创建了`LineMarkerInfo`. 当该方法处于半可见状态时(例如, 其名称可见, 但其主体的一部分不可见), 会发生以下情况:
> 1. 第一次会删除行标记信息, 因为整个`PsiMethod`不可见;
     > 2.第二遍会尝试添加行标记信息, 因为最后调用了`PsiMethod`的`LineMarkerProvider`.
     > 因此, 行标记图标会烦人地闪烁. 不如这样写:

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

### Markdown 语言提供器示例

比方说, 对于在IDE中打开的 Markdown 文件, 我们希望高亮显示其中包含链接的任何行. 我们希望在沟槽区域显示一个图标, 用户可以看到并点击该图标进行一些操作. 例如, 他们可以打开链接.

### 1. 声明依赖

此外, 由于我们依赖于`Markdown`插件, 因此必须在插件中添加以下依赖项.

在`plugin.xml`中, 我们必须添加:

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

我们必须添加`build.gradle.kts`文件里:

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

### 2. 在 XML 中注册提供器

我们需要做的第一件事是在`plugin.xml`中注册行标记提供器.

```
<extensions defaultExtensionNs="com.intellij">
  <codeInsight.lineMarkerProvider language="Markdown" implementationClass="ui.MarkdownLineMarkerProvider" />
</extensions>
```

### 3. 提供 `LineMarkerProvider` 的实现

然后, 我们必须提供一个`LineMarkerProvider`的实现, 它能为成功匹配的最细粒度的`PsiElement`返回一个`LineMarkerInfo`. 换句话说, 我们既可以匹配`LINK_DESTINATION`元素, 也可以匹配`LINK_TEXT`元素.

下面是一个例子. 对于包含内联`Markdown`链接的字符串:

```
[`LineMarkerProvider.java`](https://github.com/JetBrains/intellij-community/blob/master/platform/lang-api/src/com/intellij/codeInsight/daemon/LineMarkerProvider.java)
```

其 PSI 看起来是这样的:

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

下面是与`INLINE_LINK`匹配的行标记提供器的实现.

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
你可以在此处添加`ic_linemarkerprovider.svg`图标(在`$PROJECT_DIR/src/main/resources/icons/`文件夹中创建此文件).

```
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" height="13px" width="13px">
  <path d="M0 0h24v24H0z" fill="none" />
  <path
      d="M3.9 12c0-1.71 1.39-3.1 3.1-3.1h4V7H7c-2.76 0-5 2.24-5 5s2.24 5 5 5h4v-1.9H7c-1.71 0-3.1-1.39-3.1-3.1zM8 13h8v-2H8v2zm9-6h-4v1.9h4c1.71 0 3.1 1.39 3.1 3.1s-1.39 3.1-3.1 3.1h-4V17h4c2.76 0 5-2.24 5-5s-2.24-5-5-5z" />
</svg>
```

### 4. 提供更复杂的 `LineMarkerProvider` 实现

目前的示例只是在编辑器窗口中符合匹配条件的行旁显示一个沟槽图标. 比方说, 我们想显示一些相关操作, 这些操作可以在与沟槽图标匹配并关联的`PsiElement(s)`上执行. 在这种情况下, 我们必须深入研究一下`LineMarkerInfo`类.

如果查看[LineMarkerInfo.java](https://github.com/jetbrains/intellij-community/blob/master/platform/lang-api/src/com/intellij/codeInsight/daemon/LineMarkerInfo.java#L137), 你会发现一个`createGutterRenderer()`方法. 实际上, 我们可以重载该方法, 创建自己的`GutterIconRenderer`对象, 该对象内部有一个操作组, 其中包含所有相关操作.

下面的类[RunLineMarkerProvider.java](https://github.com/jetbrains/intellij-community/blob/master/platform/execution-impl/src/com/intellij/execution/lineMarker/RunLineMarkerProvider.java#L115) 实际上为我们提供了如何使用所有这些的一些线索. 在 IDEA 中, 当有可以运行的目标时, 就会出现一个沟槽图标(播放按钮), 允许你执行运行目标. 该类实际上提供了该功能的实现. 以此为灵感, 我们可以创建更复杂版本的行标记提供器.

我们将对`MarkdownLineMarkerProvider`的初始实现进行大幅修改. 首先, 我们要添加一个新的`LineMarkerInfo`实现类, 名为`RunLineMarkerInfo`. 该类允许我们返回一个`ActionGroup`, 现在我们必须提供该`ActionGroup`.

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

接下来是新版本的`MarkdownLineMarkerProvider`类本身.

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

`createActionGroup(...)`方法实际上是创建一个`ActionGroup`, 并添加一系列用户点击该插件的沟槽图标时可用的操作. 请注意, 你也可以使用类似下面的方法添加在`plugin.xml`中注册的操作.

```group.add(ActionManager.getInstance().getAction("ID of your plugin action"))```

最后, 这里是打开与沟槽中突出显示的`INLINE_LINK`相关联的 URL 的操作.

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
