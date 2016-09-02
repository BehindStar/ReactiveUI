# 基础属性绑定

对于使用 MVVM 模式，一个核心的部分是视图模型和视图之间非常特殊的关系，即，视图通过 *绑定* ，以一种单向依赖手段连接到视图模型。

ReactiveUI 提供了这个概念的实现，相较于平台特定的实现（比如基于 XMAL 的绑定），有许多优点。

* 绑定能够在 **全平台**  上工作，操作方式一样。
* 绑定是基于表达式的。这意味着重命名 UI 布局的某个控件，但是不更新绑定，编译不通过。
* 控制类型如何绑定到属性是灵活的，并且可以自定义。

### 入门

为了在视图中使用绑定，你必须首先在视图中实现 `IViewFor<TViewModel>`。根据平台的不同，实现将会不同：

* **iOS** - 将你的基类改为 Reactive UIKit 类型（如 ReactiveUIViewController）中的一个，并且使用 RaiseAndSetIfChanged 实现 `视图模型`，*或者* 在视图上实现 `INotifyPropertyChanged`，并确保视图模型为改变送出信号。

* **Android:** - 将基类改为 Reactive Activity / Fragment 类中的一个（比如 ReactiveActivity<T>），*或者*在视图上实现 `INotifyPropertyChanged`，并确保视图模型为改变送出信号。

* **基于 Xaml 的** - 手动实现 `IViewFor<T>` ，并确保 ViewModel 是一个依赖属性。

### 绑定的类型

一旦实现了 `IViewFor<T>`，就可以在类中使用扩展的绑定方法。就像 ReactiveUI 中的其他东西一样，在视图创建时，应该仅在构造函数或设置方法中设置绑定。

* **OneWayBind:** - 将视图模型上的一个属性单向绑定到视图的一个属性。

```cs
var disp = this.OneWayBind(ViewModel, x => x.Name, x => x.Name.Text);
disp.Dispose();   // 尽快断开绑定
```

* **Bind:** - 在视图模型上的属性与视图之间设置一个双向绑定。

```cs
this.Bind(ViewModel, x => x.Name, x => x.Name.Text);
```

* **BindCommand:** - 将 `ICommand` 绑定到控件，或者到该控件的指定事件。（实现方式取决于 UI 框架）：

```cs
// 绑定 OK 命令到按钮
this.BindCommand(ViewModel, x => x.OkCommand, x => x.OkButton);

// 绑定 OK 命令到用户按键
this.BindCommand(ViewModel, x => x.OkCommand, x => x.RootView, "KeyUp");
```

### 类型转换

直接在属性之间进行绑定很方便，但是经常两个类型之间不能相互赋值。比如，绑定一个 “Age” 属性到一个文本框通常都会失败，因为文本框需要一个字符串值。因此，ReactiveUI 有一个可扩展的强制转换系统。

关于绑定类型转换器的更多内容，查看 “自定义” 章节，了解更多关于如何扩展属性类型装换。

### 选择什么时候更新源

在默认情况下，绑定的源将会在目标更改时被更新，等同于在 WPF 绑定中设置了 `UpdateSourceTrigger = PropertyChanged`。有时需要更细粒度的控制什么时候更新源（比如，在每次敲击上进行绑定更新会触发某些昂贵的工作，这并无必要）。

一个常见需求是在用户控件失去键盘（或 逻辑）焦点时更新源，WPF 为此提供了 `UpdateSourceTrigger = 
LostFocus`。

ReactiveUI 绑定通过向绑定提供一个 IObservable 作为参数来支持这个，以禁用默认行为，并在 IObservable 触发时更新源。IObservable (`SignalViewUpdate`) 可以是任意类型，意味着用户控件的任意事件都可以导致绑定更新。

比如，在控件失去键盘焦点时，进行绑定更新，如下所示：

```cs
this.Bind(ViewModel, vm => vm.SomeProperty, v => v.SomeTextBox, SomeTextBox.Events().LostKeyboardFocus);
```

### "Hack" bindings and BindTo

你可能发现单向绑定或双向绑定不足以完成工作（或者你需要 视图 => 视图模型 绑定），可以使用一个灵活的，基于 Rx 手段：通过组合 `WhenAny` 与 `BindTo` 操作符，允许你绑定任意一个 `IObservable` 到某对象的一个属性。

比如，有一个关于绑定 ListBox 的 SelectedItem 到视图模型的例子：

```cs
public MainView()
{
    // 绑定视图的 SelectedItem 到视图模型
    this.WhenAny(x => x.SomeList.SelectedItem)
        .BindTo(this, x => x.ViewModel.SelectedItem);

    // 通过 SelectedItem 绑定视图模型的 IsSelected 。
    // 注意这仅仅是演示目的，在视图模型上绑定更加合适
    // 比如：WhenAny + ToProperty
    this.WhenAny(x => x.SomeList.SelectedItem)
        .Select(x => x != null)
        .BindTo(this, x => x.ViewModel.IsSelected);
}
```

BindTo 使用与其他属性绑定方法相同的绑定钩子和类型转换，因此源和目标的类型不需要必须匹配。

尽管你可以构建复杂绑定（甚至是在两个视图模型之间！），需要注意的是视图中的绑定逻辑无法测试，因此在绑定里面不包含有意义的逻辑是非常不错的主意。

### Hack Command Bindings

与属性绑定类似，你可以向命令添加自定义绑定。有两个方法可以使用，`InvokeCommand` 和
`WhenAnyObservable`。前一个允许你在 Observable 触发时调用命令，后者让你以一种安全的方式从视图模型安全地取得一个 Observable。如下所示：

```cs
//
// 这些代码在视图的构造函数中
//

// 在 ESC 按下时调用命令
this.Events().KeyUpObs
    .Where(x => x.EventArgs.Key == Key.Escape)
    .InvokeCommand(this, x => x.ViewModel.Cancel);

// 订阅 Cancel 事件，在触发时关闭窗体
this.WhenAnyObservable(x => x.ViewModel.Cancel)
    .Subscribe(_ => this.Close());
```
