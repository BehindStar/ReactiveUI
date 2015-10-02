# 使用 UserError 向用户报告错误

处理和友好的向用户显示错误，是每个良好的桌面、移动应用的核心工作。没有优秀的错误呈现，用户就不能解决问题和完成工作，结果用户体验很糟糕。

有一些在线文档讲述了如何构建良好的用户体验，比如 [Apple
HIG](https://developer.apple.com/library/mac/documentation/userexperience/conceptual/applehiguidelines/Windows/Windows.html#//apple_ref/doc/uid/20000961-TP10) 和 [GNOME
HIG](https://developer.gnome.org/hig-book/3.0/windows-alert.html.en#alert-text)。但是以一种天真的方式编写错误流程，结果就是容易出错、意大利面式代码等等。开发者认为“太难实现”，因而创建了[标准体验](http://cl.ly/image/100X3E2C2o3M)。

ReactiveUI 的错误框架使得分离错误的显示和源变得容易了，以一种 MVVM 友好、可测试的方式显示错误。允许在错误发生的地方处理错误，使得错误处理代码更清晰，还不需要跨越模块边界。

### UserError 类

核心类是 [`UserError`
类](https://github.com/reactiveui/ReactiveUI/blob/master/ReactiveUI/Errors.cs)。UserError 在概念上和异常类似，它们在错误发生的时候创建，并“被抛出”给处理程序。但是，不像异常，UserError 表示可以被 **用户** 处理的错误，不是程序错误。UserError **不能替代**异常。

UserErrors 由几个重要方面组成：

* `ErrorMessage`，主要消息，显示给用户的主要文字。
* `ErrorCauseOrResolution`，次要消息，描述错误产生的原因或建议的解决方案。 
* `InnerException`，可选的，提供导致显示的错误的异常。
* `RecoveryOptions`，可以用于解决该问题的命令列表。稍后有更多内容。

一旦创建了一个 UserError，可以通过 `UserError.Throw` 抛出。将会以相反的顺序调用处理程序（类似于异常向上传播的堆栈），直到 UserError 被处理。如果 UserError 没有被任何注册的处理程序处理，将会抛出一个异常。

`UserError.Throw` 将会返回用户的决定，即三个选择之一：

* `CancelOperation`，指示应该放弃任务；
* `RetryOperation`，指示错误已被处理，应该重试；
* `FailOperation`，指示错误无法处理，应该抛出异常。

```cs
var exception = default(Exception);
try {
    TheTweets = await LoadTweets();
} catch (Exception ex) {
    exception = ex;
}

if (exception != null) {
    // 注意: 这不是一个很好的错误消息
    var errorMessage = "The Tweets could not be loaded";
    var errorResolution = "Check your Internet connection";
    var userError = new UserError(errorMessage, errorResolution);

    switch (await UserError.Throw(userError)) {
    case RecoveryOptionResult.RetryOperation:
        LoadTweets.Execute();
        break;
    case RecoveryOptionResult.FailOperation:
        throw exception;
    }
}
```

将这个与 ReactiveCommand 的 `ThrownExceptions` 结合使用可以获得非常清晰的代码：

```cs
//
// 注意: 这是在视图模型中
//

LoadTweetsCommand = ReactiveCommand.CreateAsyncTask(() => LoadTweets());
    .Subscribe(x => TheTweets = x);

var errorMessage = "The Tweets could not be loaded";
var errorResolution = "Check your Internet connection";

// 任何 LoadTweets 抛出的异常都会通过 ThrownExceptions 送出
LoadTweetsCommand.ThrownExceptions
    .Select(ex => 
        new UserError(errorMessage, errorResolution))
    .SelectMany(UserError.Throw);
    .Where(x => x == RecoveryOptionResult.RetryOperation)
    .InvokeCommand(LoadTweetsCommand);
```

### 处理程序链

使用 `Throw` 只是使用错误框架的一半工作。无论如何，也必须要编写将错误呈现给用户的代码。这部分代码通常都在视图层，因为这通常与打开对话框或在 UI 上呈现有关。为此，可以使用 `UserError.RegisterHandler`：

```cs
var disconnectHandler = UserError.RegisterHandler(async error => {
    // 我们不知道 UserError 会被哪个线程抛出，
    // 通常都需要将 UserError 转移到主线程。
    await RxApp.MainThreadScheduler.ScheduleAsync(() => {
        // 注意: 这个代码是不对的，
        // 因为它丢弃了恢复选项，仅返回取消，这是糟糕的。
        return MesssageBox.Show(error.ErrorMessage);
    });

    return RecoveryOptionResult.CancelOperation;
});
```

处理程序以他们注册时相反的顺序调用，这就是说视图能够第一时间处理 UserError。

**重要须知：** 如果在视图中注册 UserError，必须在视图呈现给用户之前注册，并在视图处于未激活状态时，解除注册。不这样做的结果是，弹出多个对话框和 Developer Confusion。

### 恢复选项

恢复选项是呈现给用户以解决问题的选项——通常为对话框上的按钮。良好的恢复选项是以一种确定方式解决用户问题的 **描述行为**。比如，一个“磁盘空间不足” 的 UserError 可能呈现一个“打开查找器”恢复选项，这样用户可以查找要删除的文件。糟糕的恢复选项仅仅提供了 “是” 或 “否” ，而不是帮助用户解决下面的问题。

恢复选项在 `UserError` 创建时进行注册 —— 它们是 `ICommand` 的子类，通常默认的 `RecoveryCommand` 类就足够了。当然你可能特别懒，因此也提供了默认恢复命令（`RecoveryCommand.Yes/No`，和
`RecoveryCommand.Ok/Cancel`）

通常以这种不是特别明显的代码段处理恢复选项：

```cs
var disconnectHandler = UserError.RegisterHandler(error => {
    // TODO: 显示对话框，并将命令关联到按钮
    ShowTheErrorDialog(error);

    // 返回点击的按钮代表的 RecoveryOptionResult
    return error.RecoveryCommands
        .Select(x => x.Select(_ => x.RecoveryOptionResult))
        .Merge()
        .ObserveOn(RxApp.MainThreadScheduler);
});
```

### 不太明显的处理程序链使用

最直接的呈现错误的方式是使用警告对话框，但是这经常不是最好的方式 —— UserError 可以以多种不同的用户体验呈现。比如， GitHub 的 Windows 版本也呈现 UserError，其将在某个预设的时间过去之后，隐式的单击“取消”恢复选项：

![](http://cl.ly/image/3s3W3Y0r1S2P/content#png)

处理程序也可以不实际解决错误，而是为 UserError 添加恢复命令。这对于复杂代码非常有用，因为在视图模型在解决错误可能更方便。该模式已经封装在 `UserError.AddRecoveryOption` 方法中了。

### 测试

测试 UserError 可以通过调用 `UserError.OverrideHandlersForTesting` 方法完成，类似框架中的其他全局方法。允许你测试用户选择不用的选项或以不同的方式响应，无需编写视图。

### 总结

UserError 封装了错误对话框 —— 提供了发生了什么错误、如何解决以及用户可以选择以解决错误的行为，允许开发者在最容易解决问题的视图模型中解决问题，无需调用视图或在平台特定的视图中编写错误处理代码。
