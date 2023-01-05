## [Swift 结构化并发 _ OneV's Den](https://onevcat.com/2021/09/structured-concurrency/)
1. 如何开启一个任务上下文?
	1. Task.init
	2. Task.detached
2. 如何在任务上下文中构建子节点?
	1. TaskGroup
	2. async let
3. 任务上下文是如何结束的?
4. 非结构化并发任务如何创建?
5. 结构化并发要解决的问题? 
    - 逻辑正确
    - 内存安全
6. async关键字的作用? 它能提供异步能力吗? Task呢?

### 协程
**非抢占式或者说协作式**的计算机程序**并发调度**的实现, 程序可以主动挂起或者恢复.
角色: `Task`类似是异步函数的运行环境. `Async Function`是具体的异步任务.

### Swift 如何实现协程
`Continuation`

```swift
func foo() async -> Int {
    // 1. 获取 continuation
    await withCheckedContinuation { continuation in
        DispatchQueue.global().async {
            // 2. 恢复
            continuation.resume(returning: Int(arc4random()))
        }
    }
}
```

### 非结构化并发的问题
Swift 当前的并发手段, 最常见的是使用Dispatch库(GCD)将任务派发到指定队列, 并通过回调函数获取结果. 这些被派发的操作在运行时中, 并不知道自己从哪里来, 它们在自己的线程中拥有调用栈, 生命周期和开始的调用方作用域无关.
![](https://onevcat.com/assets/images/2021/unstructured-concurrency.png)
即使在一段时间后, 派发出去的操作通过回调函数回到闭包中, 但是它并没有关于原来调用者的信息 (比如调用栈等)，这只不过是一次孤独的跳转.
1. 由于和调用者属于不同的调用栈, 所以无法以抛出的方式向上传递错误, 只能将error作为回调函数参数.
2. goto的高级形式: 借由线程实现了任意跳转, 派发出去的操作执行会回调后, 后续无法预测.

### 结构化并发目标
单一数据流, 结果可预测.
保证控制流路径的单一入口和单一出口. 程序可以产生多个控制流来实现并发, 但是所有的并发路径在出口时都应该处于完成(或取消)状态, 并合并到一起.
![](https://onevcat.com/assets/images/2021/structured-concurrency.png)

### 基于Task的结构化并发
构建任务的从属关系(树), 即并发的入口. 特点:
1. 有优先级和取消标识, 在子任务(叶子节点)中执行异步函数.
2. 父任务取消会传递到子任务.
3. 子任务会将结果上报到父任务, 在抛出结果/错误之前, 父任务处于pending.

创建结构化任务的流程:
1. 开启任务上下文: `Task.init`/`Task.detached`/`withTaskGroup`构建根节点.
2. 创建叶子节点, 执行异步函数: `TaskGroup.addTask`或者`async let`.
3. 使用`for await`收集结果或隐式等待.

```swift
await withTaskGroup(of: Int.self) { group in
    for i in 0...3 {
        group.addTask {
            await work(i)
        }
    }
    
    for await result in group { // 如果不写, 编译器会自动添加(隐式等待)
    
    }
}
```

```swift
async let v1 = async work(0)
async let v2 = async work(1)
let result = await v1 + v2
```

> AsyncSequence:
> 
> 可以使用`for await`的语法来获取子任务的执行结果. group 中的某个任务完成时, 它的结果将被放到异步序列的缓冲区中.
> 每当 group 的 next 会被调用时, 如果缓冲区里有值, 异步序列就将它作为下一个值给出; 如果缓冲区为空, 那么就等待下一个任务完成.

> group 隐式等待:
> 
> 如果不明确等待group完成, 编译器在检测到结构化并发作用域结束时, 会自动添加await/waitForAll, 并在所有任务结束后再继续控制流.

> async let:
> 
> 简化结构化并发的使用. 右侧是异步函数的调用.
> Swift 会创建一个并发执行的子任务并立即开始执行.

> async let 隐式取消:
> 
> 如果没有 await async let 的结果, 当离开作用域时, 会隐式将子任务取消掉, 然后await.
> `v1.task.cancel();  _ = await v1`

> Task.init: 继承当前任务环境.
> Task.detached: 不继承当前任务环境.

### 其他Api
- `withUnsafeCurrentTask`: 获取当前Task.
- `static Task.isCancelled`
- `static Task.currentPriority`
- `static Task.sleep`
- `Task.value`

### 非结构化并发
通过`Task.init`或`Task.detached`创建的Task.
1. 超过作用域不会有隐式取消或等待.
2. 外层Task的取消, 并不会传递到内层的Task, 即没有树形关系.

```swift
Task { // 1
    Task { // 2
    
    }
}
```

```swift
func foo() async {
    async let t1 = Task {
        await work(1)
        print(Task.isCancelled)
    }.value
    
    // 此时 async let 创建的Task取消
    // 右侧的 Task 不会取消
}
```

### Actor: Sendable
作为模型当中的基本计算单元, 由`state` `mailbox` `executor` 三部分组成, 提供数据的安全访问.
1. 属性隔离(actor-isolated): 在内部提供一个隔离域, 在内部访问时, 可以不用使用队列和锁; 在外部对Actor的成员进行访问时, 需要明确切换到Actor的隔离域, 此时编译器自动将访问函数转换为异步函数.
2. 原理: 发送消息到mailbox, 然后executor串行执行mailbox中的消息以确保state是线程安全的(封装了私有队列的class).

> 要解决的问题?
> 为了解决数据竞争的问题.
> 
> 以前的解决方案?
> 1. 使用私有的`DispatchQueue`将对数据的操作保护起来.
> 2. 使用锁.

#### 1 外部函数修改Actor
即需要明确声明`actor-isolated`.

```swift
actor Account {
	var balance: Int
}

func deposit(to account: isolated Account) { // 编译器自动转换为 async
	account.balance += 100
}

await deposit(to: account)
```

#### 2 不需要隔离的属性或函数
声明为`nonisolated`.

### MainActor: GlobalActor
将对属性或者函数的访问隔离到主线程中执行.

### Task 与 Actor

### QA

#### 1. Task执行在什么线程?
结构化并发:
1. 入口是main
2. 执行在任意线程(由调度器决定, 这里是默认调度器)
3. 出口是main

```swift
Task {
	print(Thread.current)
	MainActor.run {
		print(Thread.current)
	}
}
```