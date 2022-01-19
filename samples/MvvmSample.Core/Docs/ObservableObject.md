---
title: ObservableObject
author: Sergio0694
description: A base class for objects of which the properties must be observable
keywords: windows 10, uwp, windows community toolkit, uwp community toolkit, uwp toolkit, mvvm, componentmodel, property changed, notification, binding, net core, net standard
dev_langs:
  - csharp
---

# ObservableObject

[`ObservableObject`](https://docs.microsoft.com/dotnet/api/microsoft.toolkit.mvvm.componentmodel.ObservableObject) 是可通过实现 [`INotifyPropertyChanged`](https://docs.microsoft.com/dotnet/api/system.componentmodel.inotifypropertychanged) 和 [`INotifyPropertyChanging`](https://docs.microsoft.com/dotnet/api/system.componentmodel.inotifypropertychanging) 接口。它可以作为所有需要支持属性更改通知的对象的起点。

## How it works

`ObservableObject` 具有以下主要特点:

-它提供了`INotifyPropertyChanged`和`INotifyPropertyChanging`的基本实现，暴露`PropertyChanged`和`PropertyChanging`事件。  
-它提供了一系列的`SetProperty`方法，可以用来轻松地设置从`ObservableObject`继承的类型的属性值，并自动引发适当的事件。  
-它提供了`SetPropertyAndNotifyOnCompletion`方法，类似于`SetProperty`，但有能力设置`Task`属性，并在分配的任务完成时自动引发通知事件。  
-它公开了`OnPropertyChanged`和`OnPropertyChanging`方法，它们可以在派生类型中被重写，以自定义通知事件如何被引发。

## Simple property

下面是如何实现对自定义属性的通知支持的示例:

```csharp
public class User : ObservableObject
{
    private string name;

    public string Name
    {
        get => name;
        set => SetProperty(ref name, value);
    }
}
```

提供的`SetProperty<T>(ref T, T, string)`方法检查属性的当前值，如果不同就更新它，然后也会自动引发相关事件。 属性名是通过使用`[CallerMemberName]`属性自动捕获的，因此不需要手动指定要更新的属性。

## Wrapping a non-observable model

这也是需要的，当想要注入通知支持模型，不实现`INotifyPropertyChanged`接口。`ObservableObject`提供了一个专门的方法来简化这个过程。 下面的例子中，`User`是一个直接映射数据库表的模型，而不继承`ObservableObject`:

```csharp
public class ObservableUser : ObservableObject
{
    private readonly User user;

    public ObservableUser(User user) => this.user = user;

    public string Name
    {
        get => user.Name;
        set => SetProperty(user.Name, value, user, (u, n) => u.Name = n);
    }
}
```

在本例中，我们使用了`SetProperty<TModel, T>(T, T, TModel, Action<TModel, T>， string)`重载。 这个签名比前一个稍微复杂一些——这是必要的，即使我们不能像前一个场景那样访问后台字段，也要让代码仍然非常高效。 我们可以详细了解这个方法签名的每个部分，以了解不同组件的角色:  

- ` TModel `是一个类型参数，表示我们包装的模型的类型。 在本例中，它将是我们的` User `类。 请注意，我们不需要显式地指定这一点——c#编译器会通过我们如何调用` SetProperty `方法自动地推断出这一点。  
- `T`是我们想要设置的属性的类型。 与` TModel `类似，这是自动推断的。  
- ` T oldValue `是第一个参数，在这种情况下，我们使用` user. Name `来传递我们正在包装的属性的当前值。  
- ` T newValue `是要设置到属性的新值，这里我们传递的是` value `，它是属性setter中的输入值。  
- `TModel model`是我们包装的目标模型，在这种情况下，我们传递的实例存储在`user`字段。  
- ` Action<TModel, T> callback `是一个函数，如果属性的新值与当前值不一致，需要对属性进行设置。 这将由这个回调函数完成，它将接收目标模型和要设置的新属性值作为输入。 在本例中，我们只是将输入值(我们称之为`n`)赋值给` Name `属性(通过执行`u.Name = n`)。 在这里，重要的是要避免从当前范围中获取值，并且只与作为回调函数输入的值交互，因为这允许c#编译器缓存回调函数并执行一些性能改进。 正因为如此，我们不只是直接访问这里的` user `字段或setter中的`value`参数，而是只使用lambda表达式的输入参数。  

`SetProperty<TModel, T>(T, T, TModel, Action<TModel, T>， string)`方法使得创建这些包装属性非常简单，因为它在提供一个非常紧凑的API的同时，处理了检索和设置目标属性。  

> [!NOTE]
> 与使用LINQ表达式实现这个方法相比，特别是通过一个类型为` Expression<Func<T>> `的参数，而不是状态和回调参数，通过这种方式可以实现的性能改进是非常显著的。 特别是，这个版本比使用LINQ表达式的版本快了大约200倍，而且根本不分配任何内存。  

## Handling `Task<T>` properties

如果一个属性是一个`Task`，那么在任务完成后也有必要触发通知事件，以便在正确的时间更新绑定。 如。 显示由任务所代表的操作的加载指示器或其他状态信息。 `ObservableObject`有一个用于这个场景的API:

```csharp
public class MyModel : ObservableObject
{
    private TaskNotifier<int>? requestTask;

    public Task<int>? RequestTask
    {
        get => requestTask;
        set => SetPropertyAndNotifyOnCompletion(ref requestTask, value);
    }

    public void RequestValue()
    {
        RequestTask = WebService.LoadMyValueAsync();
    }
}
```

这里的`SetPropertyAndNotifyOnCompletion<T>(ref TaskNotifier<T>， Task<T>， string)`方法将负责更新目标字段，监视新任务，如果存在，并在任务完成时引发通知事件。 通过这种方式，可以只绑定到任务属性，并在其状态改变时收到通知。 `TaskNotifier<T>`是`ObservableObject`公开的一种特殊类型，它包装了一个目标`Task<T>`实例，并为该方法启用了必要的通知逻辑。 如果你只有一个通用的`Task`，`Tasktifier`类型也可以直接使用。

> [!NOTE]
> `SetPropertyAndNotifyOnCompletion`方法是用来替换`Microsoft.Toolkit`包中的`NotifyTaskCompletion<T>`类型的使用。如果这个类型正在被使用，它可以被内部的`Task`(或`Task<TResult>`)属性代替，然后`SetPropertyAndNotifyOnCompletion`方法可以用来设置它的值并引发通知更改。 所有由`NotifyTaskCompletion<T>`类型公开的属性都可以直接在`Task`实例上使用。

## Sample Code

There are more examples in the [unit tests](https://github.com/Microsoft/WindowsCommunityToolkit//blob/master/UnitTests/UnitTests.Shared/Mvvm).

## Requirements

| Device family | Universal, 10.0.16299.0 or higher |
| --- | --- |
| Namespace | Microsoft.Toolkit.Mvvm |
| NuGet package | [Microsoft.Toolkit.Mvvm](https://www.nuget.org/packages/Microsoft.Toolkit.Mvvm/) |

## API

* [ObservableObject source code](https://github.com/Microsoft/WindowsCommunityToolkit//blob/master/Microsoft.Toolkit.Mvvm/ComponentModel/ObservableObject.cs)

## Related

* [MVVM Toolkit Preview 3: the journey of an API](https://devblogs.microsoft.com/pax-windows/mvvm-toolkit-preview-3-the-journey-of-an-api/)
