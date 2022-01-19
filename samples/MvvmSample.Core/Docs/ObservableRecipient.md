---
title: ObservableRecipient
author: Sergio0694
description: A base class for observable objects that also acts as recipients for messages
keywords: windows 10, uwp, windows community toolkit, uwp community toolkit, uwp toolkit, mvvm, componentmodel, property changed, notification, binding, messenger, messaging, net core, net standard
dev_langs:
  - csharp
---

# ObservableRecipient

[`ObservableRecipient`](https://docs.microsoft.com/dotnet/api/microsoft.toolkit.mvvm.componentmodel.ObservableRecipient) 类型是`observable`对象的基类，它也充当消息的接收者。这个类是`ObservableObject`的扩展，`ObservableObject`也提供了内置的`IMessenger`类型的支持  
## How it works

`Observablerreceiver`类型是用来作为视图模型的基础，也使用·IMessenger`功能，因为它提供了内置的支持。 特别是:

-它有一个无参数构造函数和一个接受`IMessenger`实例的构造函数，用于依赖注入。 它还公开了一个`Messenger`属性，可以用来在视图模型中发送和接收消息。 如果使用无参数构造函数，则`WeakReferenceMessenger`。 默认的'实例将被分配给`Messenger`属性。  
它暴露一个`IsActive`属性来激活/取消激活视图模型。 在这个上下文中，“激活”意味着一个给定的视图模型被标记为正在使用，例如。 它将开始侦听已注册的消息，执行其他设置操作，等等。 有两个相关的方法，`OnActivate`和`OnDeactivated`，当属性改变值时被调用。 默认情况下，`OnDeactivated`会自动从所有已注册的消息中注销当前实例。 为了获得最好的结果并避免内存泄漏，建议使用`OnActivated`来注册消息，并使用`OnDeactivated`来做清理操作。 该模式允许多次启用/禁用视图模型，同时可以安全地收集，而不会在每次禁用视图模型时出现内存泄漏的风险。 默认情况下，`OnActived`将自动注册所有通过`IRecipient<TMessage>`接口定义的消息处理程序。  
-它暴露了一个`Broadcast<T>(T, T, string)`方法，该方法通过`Messenger`属性中的`IMessenger`实例发送一个`PropertyChangedMessage<T>`消息。 这可以用来轻松地广播视图模型属性的更改，而不需要手动检索`Messenger`实例来使用。 此方法由各种`SetProperty`方法的重载使用，这些方法有一个额外的`bool broadcast`属性来指示是否也发送消息

下面是一个视图模型的例子，它在活动时接收`LoggedInUserRequestMessage`消息:

```csharp
public class MyViewModel : ObservableRecipient, IRecipient<LoggedInUserRequestMessage>
{
    public void Receive(LoggedInUserRequestMessage message)
    {
        // 处理这里的消息
    }
}
```

在上面的例子中，`OnActivated`自动将实例注册为`LoggedInUserRequestMessage`消息的接收者，使用该方法作为要调用的动作。 使用`IRecipient<TMessage>`接口不是强制的，注册也可以手动完成(甚至只使用内联lambda表达式):

```csharp
public class MyViewModel : ObservableRecipient
{
    protected override void OnActivated()
    {
        // 使用方法组…
        Messenger.Register<MyViewModel, LoggedInUserRequestMessage>(this, (r, m) => r.Receive(m));

        // ...或者lambda表达式
        Messenger.Register<MyViewModel, LoggedInUserRequestMessage>(this, (r, m) =>
        {
            // 处理这里的消息
        });
    }

    private void Receive(LoggedInUserRequestMessage message)
    {
        // 处理这里的消息
    }
}
```

## Sample Code

There are more examples in the [unit tests](https://github.com/Microsoft/WindowsCommunityToolkit//blob/master/UnitTests/UnitTests.Shared/Mvvm).

## Requirements

| Device family | Universal, 10.0.16299.0 or higher |
| --- | --- |
| Namespace | Microsoft.Toolkit.Mvvm |
| NuGet package | [Microsoft.Toolkit.Mvvm](https://www.nuget.org/packages/Microsoft.Toolkit.Mvvm/) |

## API

* [ObservableRecipient source code](https://github.com/Microsoft/WindowsCommunityToolkit//blob/master/Microsoft.Toolkit.Mvvm/ComponentModel/ObservableRecipient.cs)
