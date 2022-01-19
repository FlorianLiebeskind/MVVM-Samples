---
title: Messenger
author: Sergio0694
description: A type that can be used to exchange messages between different objects
keywords: windows 10, uwp, windows community toolkit, uwp community toolkit, uwp toolkit, mvvm, service, messenger, messaging, net core, net standard
dev_langs:
  - csharp
---

# Messenger

[`IMessenger`](https://docs.microsoft.com/dotnet/api/microsoft.toolkit.mvvm.Messaging.IMessenger)接口是一种类型契约，可用于在不同对象之间交换消息。这对于解耦应用程序的不同模块很有用，而不必保持对被引用类型的强引用。 还可以将消息发送到由令牌唯一标识的特定通道，并在应用程序的不同部分中使用不同的信使。 MVVM工具包提供了开箱即用的两个实现:[`WeakReferenceMessenger`](https://docs.microsoft.com/dotnet/api/microsoft.toolkit.mvvm.Messaging.WeakReferenceMessenger)和[`StrongReferenceMessenger`](https://docs.microsoft.com/dotnet/api/microsoft.toolkit.mvvm.Messaging.StrongReferenceMessenger): 前者内部使用弱引用，为接收方提供自动内存管理，而后者使用强引用，并要求开发人员在不再需要接收方时手动取消订阅(关于如何取消消息处理程序的详细信息可以在下面找到)， 但作为交换，它提供了更好的性能和更少的内存使用

## How it works

实现`IMessenger`的类型负责维护接收者(消息的接收者)与其注册的消息类型之间的链接，并使用相关的消息处理程序。 任何对象都可以使用消息处理程序注册为给定消息类型的接收者，每当使用`IMessenger`实例发送该类型的消息时，都会调用消息处理程序。 还可以通过特定的通信通道(每个通道由唯一的令牌标识)发送消息，这样多个模块可以交换相同类型的消息，而不会引起冲突。 不使用令牌发送的消息使用默认的共享通道。

有两种方法来执行消息注册:要么通过 `IRecipient<TMessage>` 接口，要么使用 `MessageHandler<TRecipient, TMessage>` 委托作为消息处理程序。 第一个让你注册的所有处理程序与单个调用 `RegisterAll` 扩展,自动注册的所有声明的消息处理程序,而后者是有用的,当你需要更大的灵活性或当你想要使用一个简单的lambda表达式作为消息处理程序。  

`WeakReferenceMessenger` 和 `StrongReferenceMessenger` 都暴露了一个 `Default` 属性，该属性提供了一个内置在包中的线程安全实现。 如果需要，也可以创建多个messenger实例，例如，如果一个不同的实例通过DI服务提供商注入到应用程序的不同模块中(例如，多个窗口运行在同一个进程中)。  

> [!NOTE]
> 因为 `WeakReferenceMessenger` 类型使用起来更简单，并且与 `MvvmLight` 库中的信使类型的行为相匹配，所以它是MVVM工具箱中的[`observablerreceiver`](observablerreceiver.md)类型所使用的默认类型。
 `StrongReferenceType` 仍然可以被使用，通过将一个实例传递给该类的构造函数。

## Sending and receiving messages

考虑以下:

```csharp
// 创建一个消息
public class LoggedInUserChangedMessage : ValueChangedMessage<User>
{
    public LoggedInUserChangedMessage(User user) : base(user)
    {        
    }
}

// 在某个模块中注册消息
WeakReferenceMessenger.Default.Register<LoggedInUserChangedMessage>(this, (r, m) =>
{
    //在这里处理消息，r是接收方，m是输入消息传递者。 
    //使用接收方作为输入传递，
    //这样lambda表达式就不会捕获“this”，从而提高了性能。  
});

// 从其他模块发送消息
WeakReferenceMessenger.Default.Send(new LoggedInUserChangedMessage(user));
```

让我们想象一下，在一个简单的消息传递应用程序中使用这种消息类型，该应用程序显示一个头(包含当前登录用户的用户名和个人资料图像)、一个包含对话列表的面板，以及一个包含来自当前对话的消息(如果选中了其中一个)的面板。 我们假设这三个部分分别由`HeaderViewModel `， `ConversationsListViewModel`和`ConversationViewModel`类型支持。 在这个场景中，`LoggedInUserChangedMessage`消息可能会在一个登录操作完成后由`HeaderViewModel`发送，并且这两个其他的视图模型可能会为它注册处理程序。 例如，`ConversationsListViewModel`将为新用户加载对话列表，而`ConversationViewModel`将关闭当前的对话，如果有的话。  

`IMessenger`实例负责将消息传递给所有注册的收件人。 请注意，收件人可以订阅特定类型的消息。 请注意，MVVM工具包提供的默认`IMessenger`实现中不会注册继承的消息类型。  

当不再需要收件人时，应注销该收件人，以便其停止接收消息。 您可以通过消息类型、注册令牌或接收方取消注册 :

```csharp
// 从消息类型中注销接收方
WeakReferenceMessenger.Default.Unregister<LoggedInUserChangedMessage>(this);

// 从指定通道中的消息类型注销接收方
WeakReferenceMessenger.Default.Unregister<LoggedInUserChangedMessage, int>(this, 42);

// 在所有通道的所有消息中注销接收方  
WeakReferenceMessenger.Default.UnregisterAll(this);
```

> [!WARNING]
> 不过，取消订阅仍然是提高性能的好方法。 另一方面，`StrongReferenceMessenger`实现使用强引用来跟踪已注册的收件人。 这样做是出于性能考虑，这意味着每个注册的接收方都应该手动注销，以避免内存泄漏。 也就是说，只要一个接收方已经注册，正在使用的`StrongReferenceMessenger`实例就会保持对它的活动引用，这将阻止垃圾收集器收集该实例。 你可以手动处理这个，或者你可以继承`observablerreceiver`，默认情况下，当它被禁用时，它会自动为收件人删除所有的消息注册(有关此的更多信息，请参阅`observablerreceiver`的文档)。

也可以使用`IRecipient<TMessage>`接口来注册消息处理程序。 在这种情况下，每个接收方都需要为给定的消息类型实现接口，并提供接收消息时将调用的`Receive(TMessage)`方法，如下所示:

```csharp
// 创建一个消息
public class MyRecipient : IRecipient<LoggedInUserChangedMessage>
{
    public void Receive(LoggedInUserChangedMessage message)
    {
        // 处理这里的消息…
    }
}

// 注册特定的信息…
WeakReferenceMessenger.Default.Register<LoggedInUserChangedMessage>(this);

// ...或者，注册所有声明的处理程序 
WeakReferenceMessenger.Default.RegisterAll(this);

// 从其他模块发送消息
WeakReferenceMessenger.Default.Send(new LoggedInUserChangedMessage(user));
```

## Using request messages

messenger实例的另一个有用特性是，它们还可以用于从一个模块向另一个模块请求值。 为了做到这一点，包包含一个基类`RequestMessage<T>`，可以这样使用:

```csharp
// 创建一个消息
public class LoggedInUserRequestMessage : RequestMessage<User>
{
}

// 在模块中注册接收器
WeakReferenceMessenger.Default.Register<MyViewModel, LoggedInUserRequestMessage>(this, (r, m) =>
{
    //假设“CurrentUser”是视图模型中的一个私有成员。 
    //和前面一样，我们通过作为输入传递给处理程序的接收者来访问它，
    //以避免在委托中捕获“this”。    
    m.Reply(r.CurrentUser);
});

// 从另一个模块请求该值
User user = WeakReferenceMessenger.Default.Send<LoggedInUserRequestMessage>();
```

`RequestMessage<T>`类包含了一个隐式转换器，它可以实现从`LoggedInUserRequestMessage`到包含的`User`对象的转换。 这还将检查是否收到了消息的响应，如果没有收到，则抛出异常。 也可以在没有强制响应保证的情况下发送请求消息:只将返回的消息存储在一个本地变量中，然后手动检查响应值是否可用。 如果`Send`方法返回时没有收到响应，那么这样做不会触发自动异常。  

相同的命名空间还包括用于其他场景的基本请求消息:`AsyncRequestMessage<T>`，`CollectionRequestMessage<T>`和`AsyncCollectionRequestMessage<T>`。  
下面介绍如何使用异步请求消息:

```csharp
// 创建一个消息
public class LoggedInUserRequestMessage : AsyncRequestMessage<User>
{
}

// 在模块中注册接收器
WeakReferenceMessenger.Default.Register<MyViewModel, LoggedInUserRequestMessage>(this, (r, m) =>
{
    m.Reply(r.GetCurrentUserAsync()); // 我们用一个Task<User>来回复
});

// 从另一个模块请求值(我们可以直接等待请求)  
User user = await WeakReferenceMessenger.Default.Send<LoggedInUserRequestMessage>();
```

## Sample Code

There are more examples in the [unit tests](https://github.com/Microsoft/WindowsCommunityToolkit//blob/master/UnitTests/UnitTests.Shared/Mvvm).

## Requirements

| Device family | Universal, 10.0.16299.0 or higher |
| --- | --- |
| Namespace | Microsoft.Toolkit.Mvvm |
| NuGet package | [Microsoft.Toolkit.Mvvm](https://www.nuget.org/packages/Microsoft.Toolkit.Mvvm/) |

## API

* [IMessenger source code](https://github.com/Microsoft/WindowsCommunityToolkit//blob/master/Microsoft.Toolkit.Mvvm/Messaging/IMessenger.cs)
* [StrongReferenceMessenger source code](https://github.com/Microsoft/WindowsCommunityToolkit//blob/master/Microsoft.Toolkit.Mvvm/Messaging/StrongReferenceMessenger.cs)
* [WeakReferenceMessenger source code](https://github.com/Microsoft/WindowsCommunityToolkit//blob/master/Microsoft.Toolkit.Mvvm/Messaging/WeakReferenceMessenger.cs)
