---
categories:
  - blog
date: 2018-05-13 14:44:08
tags:
- .Net Core
- XUnit
title: 在 .Net Core 中使用 Xunit 进行单元测试的奇葩问题
---

假如在单元测试的代码（包括测试代码及测试代码调用的任何代码）中 `Debug.Assert` 的结果为 false，Xunit 会直接崩溃，并打印出如下的错误信息：

``` text
活动的测试运行已中止。原因: .3.1.3858, Culture=neutral, PublicKeyToken=8d05b1bb7a6fdb6c]](<RunTestMethodsAsync>d__38<System.__Canon> ByRef)
   at Xunit.Sdk.TestClassRunner`1[[System.__Canon, System.Private.CoreLib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=7cec85d7bea7798e]].RunTestMethodsAsync()
   at Xunit.Sdk.TestClassRunner`1+<RunAsync>d__37[[System.__Canon, System.Private.CoreLib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=7cec85d7bea7798e]].MoveNext()
   at System.Runtime.CompilerServices.AsyncTaskMethodBuilder`1[[System.__Canon, System.Private.CoreLib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=7cec85d7bea7798e]].Start[[Xunit.Sdk.TestClassRunner`1+<RunAsync>d__37[[System.__Canon, System.Private.CoreLib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=7cec85d7bea7798e]], xunit.execution.dotnet, Version=2.3.1.3858, Culture=neutral, PublicKeyToken=8d05b1bb7a6fdb6c]](<RunAsync>d__37<System.__Canon> ByRef)
   at Xunit.Sdk.TestClassRunner`1[[System.__Canon, System.Private.CoreLib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=7cec85d7bea7798e]].RunAsync()
   at Xunit.Sdk.TestCollectionRunner`1+<RunTestClassesAsync>d__28[[System.__Canon, System.Private.CoreLib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=7cec85d7bea7798e]].MoveNext()
   at System.Runtime.CompilerServices.AsyncTaskMethodBuilder`1[[System.__Canon, System.Private.CoreLib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=7cec85d7bea7798e]].Start[[Xunit.Sdk.TestCollectionRunner`1+<RunTestClassesAsync>d__28[[System.__Canon, System.Private.CoreLib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=7cec85d7bea7798e]], xunit.execution.dotnet, Version=2.3.1.3858, Culture=neutral, PublicKeyToken=8d05b1bb7a6fdb6c]](<RunTestClassesAsync>d__28<System.__Canon> ByRef)
   at Xunit.Sdk.TestCollectionRunner`1[[System.__Canon, System.Private.CoreLib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=7cec85d7bea7798e]].RunTestClassesAsync()
   at Xunit.Sdk.TestCollectionRunner`1+<RunAsync>d__27[[System.__Canon, System.Private.CoreLib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=7cec85d7bea7798e]].MoveNext()
   at System.Runtime.CompilerServices.AsyncTaskMethodBuilder`1[[System.__Canon, System.Private.CoreLib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=7cec85d7bea7798e]].Start[[Xunit.Sdk.TestCollectionRunner`1+<RunAsync>d__27[[System.__Canon, System.Private.CoreLib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=7cec85d7bea7798e]], xunit.execution.dotnet, Version=2.3.1.3858, Culture=neutral, PublicKeyToken=8d05b1bb7a6fdb6c]](<RunAsync>d__27<System.__Canon> ByRef)
   at Xunit.Sdk.TestCollectionRunner`1[[System.__Canon, System.Private.CoreLib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=7cec85d7bea7798e]].RunAsync()
   at Xunit.Sdk.XunitTestAssemblyRunner+<>c__DisplayClass14_2.<RunTestCollectionsAsync>b__2()
   at System.Threading.Tasks.Task`1[[System.__Canon, System.Private.CoreLib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=7cec85d7bea7798e]].InnerInvoke()
   at System.Threading.ExecutionContext.Run(System.Threading.ExecutionContext, System.Threading.ContextCallback, System.Object)
   at System.Threading.Tasks.Task.ExecuteWithThreadLocal(System.Threading.Tasks.Task ByRef)
   at System.Threading.Tasks.Task.ExecuteEntry()
   at System.Threading.Tasks.SynchronizationContextTaskScheduler+<>c.<.cctor>b__8_0(System.Object)
   at Xunit.Sdk.MaxConcurrencySyncContext.RunOnSyncContext(System.Threading.SendOrPostCallback, System.Object)
   at System.Threading.ExecutionContext.Run(System.Threading.ExecutionContext, System.Threading.ContextCallback, System.Object)
   at Xunit.Sdk.MaxConcurrencySyncContext.WorkerThreadProc()
   at Xunit.Sdk.XunitWorkerThread+<>c.<QueueUserWorkItem>b__5_0(System.Object)
   at System.Threading.ExecutionContext.Run(System.Threading.ExecutionContext, System.Threading.ContextCallback, System.Object)
   at System.Threading.Tasks.Task.ExecuteWithThreadLocal(System.Threading.Tasks.Task ByRef)
   at System.Threading.ExecutionContext.Run(System.Threading.ExecutionContext, System.Threading.ContextCallback, System.Object)


测试运行已中止。
```

这里没有任何关于哪里出了问题的提示。

这应该是 Xunit 的一个 bug，讨论在这里： https://github.com/xunit/xunit/issues/1482

目前还没找到什么好的解决办法。
