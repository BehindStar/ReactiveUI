# CreateCollection 和测试

Rx 的好处之一是，它让以前不可测试的代码变得具有确定性、容易验证测试。应用中的大多数竞用条件实际上是**顺序问题**，而不是**时序问题**，这个问题在——这个发生、然后这个发生、再然后那个发生——的时候发生。

Rx 让你很容易的控制事件发生的顺序，以一种任意的组合方式。这意味着 `async` 测试每次以一种相同的方式运行。

### Rx 容易测试

在 Rx 中控制并发的关键，是任何延迟的（比如在**另一个线程**上执行）是通过 `IScheduler` 接口发送的。与在**调度器**中提到的一样，ReactiveUI 允许将全局调度器 `MainThreadScheduler` 和 `TaskpoolScheduler` 替代你自己的。

### 使用 CreateCollection 验证更改

一个测试返回 `IObservable<T>` 的方法的简单方式是，将其转换为一个集合，然后检查其**最终**内容。`CreateCollection` 与 `ToList` 不同， `CreateCollection` **立即** 返回，而 `ToList` 在 Observable 完成后才返回。

在这个 ReactiveList 的测试中，将所有的 Reset 事件，都放在了一个集合中：

```cs
public void GetAResetWhenWeAddALotOfItems()
{
    var fixture = new ReactiveList<int> { 1, };
    var reset = fixture.ShouldReset.CreateCollection();
    Assert.Equal(0, reset.Count);

    fixture.AddRange(new[] { 2,3,4,5,6,7,8,9,10,11,12,13, });
    Assert.Equal(1, reset.Count);
}
```

### Subject 在模拟输入时非常有用

Rx，使用 Subject，让模拟异步方法为同步方式变得轻而易举。当你使用 Subject 替换并发方法（比如那些在后台线程上运行的方法）的时候，它们在一个单独的线程上立即运行 —— 也就是说测试以一种相同的方式执行。

这是 ReactiveUI 测试组件中的另一个例子：

```cs
public void DerivedCollectionSignalledToResetShouldFireExactlyOnce()
{
    var input = new List<string> { "Foo" };
    var resetSubject = new Subject<Unit>();
    var derived = input.CreateDerivedCollection(x => x, signalReset: resetSubject);
    
    var changeNotifications = derived.Changed.CreateCollection();

    Assert.Equal(0, changeNotifications.Count);
    Assert.Equal(1, derived.Count);

    input.Add("Bar");

    // 不应该获得任何东西，因为输入还没有响应
    Assert.Equal(0, changeNotifications.Count);
    Assert.Equal(1, derived.Count);

    resetSubject.OnNext(Unit.Default);

    Assert.Equal(1, changeNotifications.Count);
    Assert.Equal(2, derived.Count);
}
```


We set up things to asynchronously wait on Observables, then use
Subject.OnNext to move the order of events forward to the next part of the
operation.

### 使用 TestScheduler 的时机
