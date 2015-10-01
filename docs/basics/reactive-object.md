# 使用 ReactiveObject 创建视图模型

每个 MVVM 框架的核心都是 **视图模型**，同时这个类也是 MVVM 模式的最有意思之处，同时也最容易误解。正确的理解视图模型是什么、不是什么，是正确应用 MVVM 模式的关键。

### 视图模型之道

大多数 UI 框架在设计时都没有考虑到单元测试，或者这些考虑被认为是多余的。结果就是，UI 对象很难测试，因为他们不是 **纯对象（plain object）**。他们可能依赖于运行回路的存在，或者常常需要以一种确定的方式初始化静态类或全局类。

因此，由于 UI 类不可测试，我们的新目标是，尽可能将我们的**感兴趣的代码**放在一个**代表**视图的类中，这类是我们可以创建的标准类。然后，我们希望视图中的代码尽可能的无聊、机械和简短，因为它本质上是不可测试的。

这个类被称为**视图模型**，它是**视图**的**模型**。这意味着，每个视图都有一个模型。这并不需要严格遵守，但是一般都这样做。

理解视图模型的另一个重点是它们是分离**技术**与**策略**的抽象。视图模型与特定的按钮、菜单和文本框无关，只是描述了这些元素的数据是如何关联的。比如，“复制” `ICommand` 不知道他会连接到菜单项或按钮，它仅仅将复制的**动作**模型化。映射复制命令与调用的控件是视图的责任。

### 视图可重用

因为视图模型不需要明确引用 UI 框架或控件，这意味着视图模型可以**跨平台**重用。对于迁移到新平台来说，该模式能够显著的减少需要的时间，特别是结合了为此任务设计的可移植类库的话，比如 [Splat](https://github.com/paulcbetts/splat) 和
[Akavache](https://github.com/akavache/Akavache)。大多数代码（模型、网络处理、缓存、图片加载、视图模型）都能在所有平台上使用，只有视图相关的类需要重写。

### 常见错误与误解

许多人认为 MVVM 模式意味着在视图的代码后置类中不应该有任何代码，或者全部都应该在 XAML 中。虽然某些模式，例如 Blend 触发器促进了代码重用，但这是反模式（Antipattern）。C# 是比 XAML 更具有表现力，更简洁的语音，尽管 XAML **可能** 创建一个完整的复杂视图，但将是不可维护，难以阅读的混乱。

于是，该如何决定哪些放在视图中呢？诸如类似滚动位置和控制焦点是视图特定代码的好例子。处理动画和窗体位置、最小化也是应该放在视图中的例子。

另一个常见误解是分离——尽管视图模型不应该引用视图或任何视图创建的控件，是非常重要的，**反过来就不是这样了**。视图可以紧紧绑定到视图模型，并且实际上，这对于视图通过 `WhenAny` 和 `WhenAnyObservable` “伸入” 视图模型，是非常有用的。

说完了理论，让我们来看看如何在 ReactiveUI 中创建视图模型吧。

### 可读写属性

有更改通知的属性（它们被改变时进行通知），写法如下：

```cs
string name;
public string Name {
    get { return name; }
    set { this.RaiseAndSetIfChanged(ref name, value); }
}
```

注意，不像其他框架，（可读写）属性**应该总是这样写**，使用完全相同的样板代码。如果视图把**任何东西**放到写访问器中，你肯定**做错了**，使用 `WhenAny` 和 `ToProperty` 替代。

### 只读属性

仅在构造函数中初始化的属性，从不更改，不需要通过 `RaiseAndSetIfChanged` 重写，可以被定义普通属性：

```cs
// 命令也几乎在构造函数中初始化并且从不更改
// 因此也适合这个模式
public ReactiveCommand<Object> PostTweet { get; protected set; }

public PostViewModel()
{
    PostTweet = ReactiveCommand.Create(/*...*/);
}
```

### 输出属性

到目前为止，没什么特别令人惊讶的东西，都是 MVVM 共有的功能。但是，在 ReactiveUI 中有其他类型的属性，不存在与其他框架中，对于有效使用 ReactiveUI **非常重要**，即 `输出属性`。

输出属性，是获取 *Observable* 并将其转换为 **视图模型属性** 的一种方式。我们常常使用相反的方法，`WhenAny`，用于将视图模型属性转换为 *Observable*。正如其名，输出属性通常是只读的（比如，源决定什么时候属性改变）。

首先，使用 `ObservableAsPropertyHelper<T>` 类，定义一个输出属性：

```cs
readonly ObservableAsPropertyHelper<string> firstName;
public string FirstName {
    get { return firstName.Value; }
}
```

与创建可读写属性类似，这些代码也是100%模板化的。

然后，在构造函数中，使用帮助方法 `ToProperty` 初始化 `firstName`：

```cs
this.WhenAnyValue(x => x.Name)
    .Select(x => x.Split(' ')[0])
    .ToProperty(this, x => x.FirstName, out firstName);
```

这里，`ToProperty` 创建一个 `ObservableAsPropertyHelper` 实例，将会在 `FirstName` 属性被改变时发出通知。`ToProperty` 是一个 `IObservable<T>` 的扩展方法，类似于 `Subscribe`。

### 最佳实践

功能响应编程（Functional Reactive Programming）的一个核心概念是，与其写 **线性代码** （如，现在做 A，然后做 B，最后做 C），我们希望写 **函数的、声明式代码** ；与其写事件处理程序和方法去改变属性，我们希望 **去描述属性之间是如何关联的**，使用 `WhenAny` 和 `ToProperty` 。

结果就是，在一个编码良好的 ReactiveUI 视图模型中，几乎所有我们感兴趣的代码，都在**构造函数**中；这些代码描述视图模型中的属性是如何相互关联的。你的目标是在编写视图模型代码时，陈述视图**应该**如何依据视图模型的命令和属性工作，并将其翻译为构造函数中的代码。例如：

* 登陆按钮在用户名和密码都不为空时才能点击
* 错误消息应该在10秒后消失
* DirectMessageToSend 对象由目标用户和要发送的消息组成

所有这些声明都是 UI 应该如何工作的简单描述，它们可以被直接翻译为 Rx 表达式，在视图模型的构造函数中。
