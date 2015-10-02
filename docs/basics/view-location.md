# 视图定位和 IViewFor

视图定位是 ReactiveUI 的一个功能，允许你关联视图和视图模型，并自动的设置它们。

### ViewModelViewHost

使用视图定位最容易的方式是通过 `ViewModelViewHost` 控件，它是一个视图（在 Cocoa 是一个 UIView/NSView，在基于 XAML 的平台中，是一个控件），有一个 `ViewModel` 属性。当
`ViewModel` 被设置时，视图定位查找相关视图并将其加载到容器。`ViewModelViewHost` 是一个强大的列表，以至于你如果你绑定到基于 XAML 的平台的 `ItemsSource` 的话，不需要设置一个数据模板（DataTemplate），只需要使用 `ViewModelViewHost` 就能完成配置。

```xml
<ListBox x:Name="ToasterList" />
```

```cs
// 现在 ListBox 自动获得 DataTemplate
this.OneWayBind(ViewModel, vm => vm.ToasterList, v => v.ToasterList.ItemsSource);
```

### 注册新视图

要使用视图定位，必须首先注册类型，通过 Splat 的服务定位功能。

```cs
Locator.CurrentMutable.Register(() => new ToasterView(), typeof(IViewFor<ToasterViewModel>)); 
```

视图定位在内部使用了一个叫做 `ViewLocator` 的类，该类可以被替换，或者作为默认类。`ResolveView` 方法将返回与指定视图模型关联的视图。
