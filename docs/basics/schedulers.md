# 默认调度器（Scheduler）

调度器是任何使用 Reactive Extension 编写的程序中的核心部分，因为所有操作都是延迟的（如在其他线程或在主线程上运行）。调度器允许应用控制代码的运行上下文，重要的是，在其他线程上运行的库是 scheduler-aware。

ReactiveUI 提供了两个应用级别的调度器，用于替代其他调度器，比如 Rx 内置的调度器：

* **RxApp.MainThreadScheduler** - 该调度器在 UI 线程上执行。在基于 XAML 的平台上，它等同于 `Dispatcher.BeginInvoke`。

* **RxApp.TaskpoolScheduler** - 该调度器通过 TPL 线程池执行代码。等同于 `Task.Run`。

这些全局变量是**全局的**，因为他们可以在单元测试时被全局的替换。（见后面的**测试调度器**章节）。这意味着，在单元测试中，可以使得**并发代码**（比如，同时在许多线程上执行的代码）以同步方式运行，并且**每次都发生相同的结果**。

### 与调度相关的对象

* **ReactiveCommand** - CreateAsyncXYZ 在 RxApp.MainThreadScheduler 上返回结果。

* **MessageBus** - 消息总线可以选择调度消息到任意调度器。

### 我应该关注调度的哪方面

你应该尝试删除所有并发源，除了通过 RxApp 调度的以外。这并不一定可能，但是通过 `new
Thread()` 和 `Task.Run` 创建的线程不能被单元测试控制。最简单的修复方式是使用 `Observable.Start` 来替代它们：

旧:

```cs
var result = await Task.Run(() => {
    int number = ThisCalculationTakesALongTime();
    return number;
});

Dispatcher.BeginInvoke(new Action(() => DoAThing()));
```

新:

```cs
var result = await Observable.Start(() => {
    int number = ThisCalculationTakesALongTime();
    return number;
}, RxApp.TaskpoolScheduler);

RxApp.MainThreadScheduler.Schedule(() => DoAThing());
```

如果你创建了一个共享组件，你应该考虑到允许通过一个可选的构造函数参数来指定调度器。

### 测试调度器

在单元测试中，默认情况下，`MainThreadScheduler` 直接运行代码，而不是在 UI 线程（不存在）上。默认情况下，不会修改 `TaskpoolScheduler`。

在候选调度器上运行的最佳方式是使用 `With` 方法，常常和 TestScheduler 一起使用。它使用指定调度器将**两个调度器**都替换了：

```cs
(new TestScheduler()).With(sched =>{
    // 在这块中运行的代码，其 RxApp.MainThreadScheduler 和 RxApp.TaskpoolScheduler 都被赋值为新的调度器
});
```
