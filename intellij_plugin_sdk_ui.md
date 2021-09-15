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
