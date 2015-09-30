# 简介

ReactiveUI 是 MVVM 和 Reactive Extensions (Rx) 的组合。组合这两者使得尽可能的以一种声明的、函数的方式，管理并发和表示对象之间的复杂交互。简单的说，如果你曾经不得不将事件、回调串联起来，以及定义状态整数/布尔值，以跟踪状态变化，那么 Reactive Extensions 是一个理想的选择。

## 库包含的内容

- **ReactiveObject** - 一个 ViewModel 对象，基于 Josh Smith 的实现，其实现了 IObservable 作为一种通知属性变化的方式。

- **ReactiveCommand** - 一个 ICommand 的实现，其在 Execute 执行完成后，触发 OnNext （事件）。他的 CanExecute 通过一个 IObservable 定义，这样 UI 就会立即更新，而无需依赖于 WPF 中 RequerySuggested 的实现。ReactiveCommand 封装了基本模式——“执行异步命令，然后返回结果到调度线程”。他还允许你控制是否允许并发。

- **ObservableAsPropertyHelper<T>** - 一个类，让你容易的将一个 IObservable 转换为一个存储其最新值的属性。这对于组合现有属性非常有用，替代 IValueConverters，因为你的 ViewModels 也是 IObservable。

- **ReactiveList<T>** - 一个 ObservableCollection 的自定义实现，允许你看见集合中的更改。

- **MessageBus** - 一个发布——订阅模式的响应实现，对于解耦对象十分有用，在对象之间仍然保持通讯。

- **MemoizingMRUCache** - 一个缓存，只保留指定数量的最近使用的对象。

- **ObservableAsyncMRUCache** - 一个线程安全的，异步的 MemoizingMRUCache。

- **ReactiveBinding** - 一个强大而灵活的跨平台绑定框架，作为 Xaml 绑定的替代。

## 组织

该库被组织成若干个高级程序集：

- **ReactiveUI** - 不依赖任何特定 UI 框架的核心库。`ReactiveObject`：ViewModel 对象的基类；以及 `ReactiveList<T>`：一个更好用的 ObservableCollection；`ReactiveCommand`：一个 ICommand 的实现。绑定框架也在这里。

- **ReactiveUI.Platforms** - 需要引用一个 Xaml 框架，比如 WPF 或 WinRT。这个程序集包含绑定框架的 Xaml 部分、屏幕、在基于 ViewModel 的视图之间导航时非常有用的导航框架。 

- **ReactiveUI.Blend** - 包含一些 Blend 行为和触发器，用于附加 ViewModel 变化到 Visual State Manager 状态。

- **ReactiveUI.Mobile** - 在为移动平台开发时十分有用，比如 Windows Phone 或 Windows Runtime。这些类处理类似保持状态和响应应用程序生命周期事件等事情。

## ReactiveObject 

和其他 MVVM 框架一样。ReactiveUI 有一个为 ViewModel 设计的类。该类基于 Josh Smith 的 ObservableObject 实现（实际上，许多类的灵感都来自于此）。这个 Reactive 版本就如你想象的那样， 实现了 INotifyPropertyChanged 和 IObservable，因此你可以订阅对象的属性改变。

ReactiveObject 也有一些其他的优秀特性：首先，如果你以 Debug 模式编译 ReactiveUI，他将在属性变化时，使用日志框架输出调试信息。其次，可以以一种十分简单的方式触发属性更改事件（有效利用了新的 CallerMemberName 特性）：

```cs
int _someProp; 
public int SomeProp { 
  get { return _someProp; } 
  set { this.RaiseAndSetIfChanged(ref _someProp, value); } 
}
```

## ReactiveCommand

ReactiveCommand 是一个 ICommand 实现，同时也是一个 RelayCommand 实现。可以提供一个 IObservable 对象给 CanExecute。比如，这里有一个命令，只能在鼠标弹起时运行：

```cs
var mouseIsUp = Observable.Merge(
   Observable.FromEvent<MouseButtonEventArgs>(window, ”MouseDown”).Select(_ => false), 
   Observable.FromEvent<MouseButtonEventArgs>(window, ”MouseUp”).Select(_ => true),
).StartWith(true);

var cmd = new ReactiveCommand(mouseIsUp); 
cmd.Subscribe(x => Console.WriteLine(x));
```

或者，一个只能在其他两个命令被禁用时运行的命令：

```cs
// 假设已经初始化了
var cmd1 = new ReactiveCommand(); 
var cmd2 = new ReactiveCommand();

var can_exec = cmd1.CanExecuteObservable.CombineLatest(cmd2.CanExecuteObservable, (lhs, rhs) => !(lhs && rhs));
var new_cmd = new ReactiveCommand(can_exec);
new_cmd.Subscribe(Console.WriteLine);
```

需要注意的一件更重要的事情是，命令的 CanExecute 是即时更新的，而不是依赖于 CommandManager.RequerySuggested。如果你曾经在 WPF 或 Silverlight 中遇到 ——直到失去焦点或单击他们之前，按钮不能自动启用——就已经看到过这个BUG了。使用 IObservable 意味着 Commanding 框架准确的知道什么时候状态改变了，而不需要重新查询每个命令对象。

### 关于 Execute

这是 ReactiveCommand 的 IObservable 实现带来的。ReactiveCommand 可以被观察，同时他在 Execute 被调用时，传递新对象（就是传递给调用 Execute 的参数）给 Subscribe。这意味着，Subscribe 与 Execute 动作的行为是一样的，比如：

```cs
var cmd = new ReactiveCommand();
cmd.Where(x => ((int)x) % 2 == 0).Subscribe(x => Console.WriteLine(”Even numbers like {0} are cool!”, x));
cmd.Where(x => ((int)x) % 2 != 0).Timestamps().Subscribe(x => Console.WriteLine(”Odd numbers like {0} are even cooler, especially at {1}!”, x.Value, x.Timestamp));

cmd.Execute(2); 
>>> ”Even numbers like 2 are cool!”

cmd.Execute(5); 
>>> ”Odd numbers like 5 are even cooler, especially at (the current time)!”
```

### 异步运行命令

在事件处理器中执行耗时操作，比如读取大文件、从网络下载东西，会阻塞 UI；或者干脆就不能执行阻塞操作，必须在另一个线程上执行。然后你发现 WPF 和 Silverlight 的线程问题，即，你只能在创建它们的线程上访问它们。因此，在计算的最后执行 runtextBox.Text = results 时，你突然获得一个异常。Dispatcher.BeginInvoke 能够解决这个问题，这样，你发现了使用 Dispatcher 解决这个问题的模式：

```cs
void SomeUIEvent(object o, EventArgs e) { 
  var some_data = this.SomePropertyICanOnlyGetOnTheUIThread;
  var t = new Task(() => { 
    var result = doSomethingInTheBackground(some_data);
    Dispatcher.BeginInvoke(new Action(() => { this.UIPropertyThatWantsTheCalculation = result; }));
  }

  t.Start();
}
```

我们常常使用这个模式，因此在执行一个命令时，我们会这样：

1. 命令执行，开启了一个新的线程

2. 执行一个长时间运算

3. 获得结果，使用 Dispatcher 在 UI 上设置属性

ReactiveCommand 封装了这个模式，允许你注册一个执行的 Task 或 IObservable。同时还有其他好处。比如，在异步实例执行期间，禁用命令。或者，在执行期间显示执行状态或进度条。

这里是一个使用命令的简单例子，其将会在后台执行一个任务，并且同一时间内只能执行一个任务。（比如，它的 CanExecute 将会返回 false，直到动作完成）

```cs
var cmd = new ReactiveCommand(null, false, /* 不允许并发 */ null);
cmd.RegisterAsyncAction(i => {
    Thread.Sleep((int)i * 1000); // 假定执行一项工作
};

cmd.Execute(5 /*秒*/); 
cmd.CanExecute(5); // False! 正在执行第一个任务。
```

## 了解更多

关于更多如何使用 ReactiveUI 的信息，前往
[ReactiveUI](http://www.reactiveui.net)。
