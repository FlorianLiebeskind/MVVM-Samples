---
title: AsyncRelayCommand
author: Sergio0694
description: An asynchronous command whose sole purpose is to relay its functionality to other objects by invoking delegates
keywords: windows 10, uwp, windows community toolkit, uwp community toolkit, uwp toolkit, mvvm, componentmodel, property changed, notification, binding, command, delegate, net core, net standard
dev_langs:
  - csharp
---

# AsyncRelayCommand and AsyncRelayCommand&lt;T>

The [`AsyncRelayCommand`](https://docs.microsoft.com/dotnet/api/microsoft.toolkit.mvvm.input.AsyncRelayCommand) and [`AsyncRelayCommand<T>`](https://docs.microsoft.com/dotnet/api/microsoft.toolkit.mvvm.input.AsyncRelayCommand-1) are `ICommand` implementations that extend the functionalities offered by `RelayCommand`, with support for asynchronous operations.

## How they work

`AsyncRelayCommand` 和 `AsyncRelayCommand<T>` 有以下特点:

- 它们扩展了库中包含的同步命令的功能，支持返回`Task`的委托
- 它们可以用附加的`CancellationToken`参数包装异步函数来支持取消，并公开`canbecanccanceled`和`IsCancellationRequested`属性，以及`Cancel`方法
- 它们公开了一个`ExecutionTask`属性，可用于监视挂起操作的进度，以及一个`IsRunning`属性，可用于检查操作何时完成。 这在将命令绑定到UI元素(如加载指示器)时特别有用
- 它们实现了`IAsyncRelayCommand`和`IAsyncRelayCommand<T>`接口，这意味着视图模型可以轻松地使用这些来暴露命令，以减少类型之间的紧密耦合。 例如，如果需要，这使得用公开相同公共API表面的自定义实现替换命令变得更容易

## Working with asynchronous commands

让我们想象一个类似于`RelayCommand`示例中描述的场景，但是命令执行了一个异步操作

在单击`Button`时，命令被调用，`ExecutionTask`被更新。 当操作完成时，该属性将引发一个通知，该通知将反映在UI中。 在这种情况下，将同时显示任务状态和任务的当前结果。 请注意，为了显示任务的结果，需要使用`TaskExtensions.GetResultOrDefault`方法——该方法提供了对尚未完成任务的结果的访问，而不会阻塞线程(可能会导致死锁)

## Sample Code

There are more examples in the [unit tests](https://github.com/Microsoft/WindowsCommunityToolkit//blob/master/UnitTests/UnitTests.Shared/Mvvm).

## Requirements

| Device family | Universal, 10.0.16299.0 or higher |
| --- | --- |
| Namespace | Microsoft.Toolkit.Mvvm |
| NuGet package | [Microsoft.Toolkit.Mvvm](https://www.nuget.org/packages/Microsoft.Toolkit.Mvvm/) |

## API

* [AsyncRelayCommand source code](https://github.com/Microsoft/WindowsCommunityToolkit//blob/master/Microsoft.Toolkit.Mvvm/Input/AsyncRelayCommand.cs)
* [AsyncRelayCommand&lt;T> source code](https://github.com/Microsoft/WindowsCommunityToolkit//blob/master/Microsoft.Toolkit.Mvvm/Input/AsyncRelayCommand{T}.cs)
