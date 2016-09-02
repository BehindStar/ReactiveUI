# 使用 ReactiveCommand 执行异步操作

ReactiveCommand 的一个最重要的功能是其内置的异步操作设施。以前的版本中，它是一个独立的命令类，但是从 5.0 开始，它是内置的了。

### Commands 和 CreateAsyncXYZ

要使用 ReactiveCommand 执行异步操作，通过 `CreateAsync` 方法家族创建命令

* **Create** - 创建标准 ReactiveCommand
* **CreateAsyncObservable** - 创建一个命令，它的异步方法返回 `IObservable<T>`
* **CreateAsyncTask** - 创建一个命令，它的异步方法返回 `Task` 或 `Task<T>`；在想使用 `async/await` 编写方法时使用这个方法。

这些方法将方法的结果作为创建的 ReactiveCommand 的参数化类型（比如，如果你的异步方法返回 `Task<String>`，那么命令将会是 `ReactiveCommand<String>`）。这意味着，订阅该命令自身返回异步方法的结果作为一个 Observable。

ReactiveCommand 自己保证他的结果**总是**在 UI 线程上传输，因此不需要额外的 `ObserveOn`。

重要的是，ReactiveCommand 自身作为一个 `IObservable`，将不会完成或触发错误（OnError）——异步方法中发生的错误将会通过 `ThrownExceptions` 属性出现。

如果你的异步方法可能抛出异常（通常都是这样），你**必须**订阅 `ThrownExceptions` ，否则异常将会在 UI 线程上再次抛出！

一个简单的例子：

```cs
LoadUsersAndAvatars = ReactiveCommand.CreateAsyncTask(async _ => {
    var users = await LoadUsers();

    foreach(var u in users) {
        u.Avatar = await LoadAvatar(u.Id);
    }

    return users;
});

LoadUsersAndAvatars.ToProperty(this, x => x.Users, out users);

LoadUsersAndAvatars.ThrownExceptions
    .Subscribe(ex => this.Log().WarnException("Failed to load users", ex));
```

### 如何执行命令

执行 ReactiveCommand 最好的方式是通过 `ExecuteAsync` 方法：

```cs
LoadUsersAndAvatars = ReactiveCommand.CreateAsyncTask(async _ => {
    var users = await LoadUsers();

    foreach(var u in users) {
        u.Avatar = await LoadAvatar(u.Id);
    }

    return users;
});

var results = await LoadUsersAndAvatars.ExecuteAsync();
Console.WriteLine("You've got {0} users!", results.Count());
```

你**必须 await ExecuteAsync**，这非常重要，否则什么事情也不会发生！`ExecuteAsync` 返回一个 *Cold Observable* ，意味着它只会在某人订阅它之后开始工作。

为了遗留代码和绑定到 UI 框架，Execute 方法依然保留。

### 为什么要 CreateAsyncTask

因为 ReactiveCommand 是一个 Observable，基于 ReactiveCommand 调用异步动作是非常容易的，例如：

```cs
searchButton
    .SelectMany(async x => await executeSearch(x))
    .ObserveOn(RxApp.MainThreadScheduler)
    .ToProperty(this, x => x.SearchResults, out searchResults);
```

无论如何，尽管这种模式很平易近人，他可能导致在搜索执行时禁用命令变得困难™（比如，阻止同时运行多个搜索命令）。CreateAsyncTask 帮你处理了这个问题。

另一个难点是不能处理异常——如果 `executeSearch` 曾经失败过一次，它将不会再次触发了——这是 Rx 约定。ReactiveCommand 将异常封送到 `ThrownExceptions` 属性，可以通过该属性处理异常。

### 常用模式

这个 UserError 的例子同时演示了 CreateAsyncTask 的标准用法：

```cs

// 在 LoadTweetsCommand 被调用时，
// 将会在后台运行 LoadTweets，其结果可以在主线程上观察，
// ToProperty 将会将结果保存在一个输出属性上 
LoadTweetsCommand = ReactiveCommand.CreateAsyncTask(() => LoadTweets())

LoadTweetsCommand.ToProperty(this, x => x.TheTweets, ref theTweets);

var errorMessage = "The Tweets could not be loaded";
var errorResolution = "Check your Internet connection";

// LoadTweets 抛出的所有异常都将通过 ThrownExceptions 输出
LoadTweetsCommand.ThrownExceptions
    .Select(ex => new UserError(errorMessage, errorResolution))
    .Subscribe(x => UserError.Throw(x));
```
