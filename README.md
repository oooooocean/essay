# React

## [Swift 结构化并发 _ OneV's Den](./Swift 结构化并发 _ OneV's Den.pdf)
1. 如何开启一个任务上下文?
2. 如何在任务上下文中构建子节点?
3. 任务上下文是如何结束的?
4. 非结构化并发任务如何创建?

### 1. 非结构化并发的问题
Swift 当前的并发手段, 最常见的是使用Dispatch库(GCD)将任务派发到指定队列, 并通过回调函数获取结果. 这些被派发的操作在运行时中, 并不知道自己从哪里来, 它们在自己的线程中拥有调用栈, 生命周期和开始的调用方作用域无关.
![](https://onevcat.com/assets/images/2021/unstructured-concurrency.png)
即使在一段时间后, 派发出去的操作通过回调函数回到闭包中, 但是它并没有关于原来调用者的信息 (比如调用栈等)，这只不过是一次孤独的跳转.
1. 由于和调用者属于不同的调用栈, 所以无法以抛出的方式向上传递错误, 只能将error作为回调函数参数.
2. goto的高级形式: 借由线程实现了任意跳转, 派发出去的操作执行会回调后, 后续无法预测.

### 2. 结构化并发目标
单一数据流, 结果可预测.
保证控制流路径的单一入口和单一出口. 程序可以产生多个控制流来实现并发, 但是所有的并发路径在出口时都应该处于完成(或取消)状态, 并合并到一起.
![](https://onevcat.com/assets/images/2021/structured-concurrency.png)

### 3. 基于Task的结构化并发
构建任务的从属关系(树). 特点:
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

### 4. 其他Api
- `withUnsafeCurrentTask`: 获取当前Task.
- `static Task.isCancelled`
- `static Task.currentPriority`
- `static Task.sleep`
- `Task.value`

### 4. 非结构化并发
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

## [Why React Re-Renders](./Why%20React%20Re-Renders.pdf)
`snapshot`

### 1. re-render
Every re-render in React starts with a state change.
1. Re-renders only affect the component that owns the state and its descendants(if any).
2. Re-render is to figure out what needs to change.

> **Re-render 不会真正创建Dom.**
> 
> Each render is a snapshot, that shows what the UI should look like, based on the current application state.
> Then, React find the differences between these tow snapshots. 
> 
> Usually, re-render is fast enough. But, in certain situations, these snapshots do take a while to create. This can lead to performance problems, like the UI not updating quickly enough after the user performs an action.

> **Misconception: A component will re-render because its props change.**
> 
> When a component re-renders, it tries to re-render all descendants, regradless of whether they're being passed a particular state variable throught props or not.
> 
> Why?
>
> Because, It's hard for React to know, with 100% certainty, whether children depends, directly or indirectly, on state variable.

### 2. ignore certain re-render requests
- `React.memo`: "Hey, I know that this component is pure. You dont't need to re-render it unless its props/dependences change".
- `React.useContext`: context is sorta like "invisible props", or maybe "internal props".

> Memoization
> 
> React will remember the previous snapshot. 
> If none of the props have changed, React will re-use that state snapshot rather than going through the trouble of generating a brand new one.

```js
const UserContext = React.createContext();

(
  <UserContext.Provider value={user}>
    ...
  </UserContext.Provider>
);

const GreetUser = React.memo(() => {
    const user = React.useContext(UserContext);
    
    return `Hello ${user.name}!`;
});
```

### 3. 警惕引用导致re-render.
组件的依赖是引用类型, 当引用变化时会导致re-render.


```js
function App() {
    const boxes = [
        { flex: boxWidth, background: 'hsl(345deg 100% 50%)' },
        { flex: 3, background: 'hsl(260deg 100% 40%)' },
        { flex: 1, background: 'hsl(50deg 100% 60%)' },
        ];
        
    return usememo(() => (  // 依赖引用, 即使使用memo, 当外部boxes引用变化时, 依然会刷新
        <Boxes boxes={boxes} />
     ));
}
```
解决方案:
1. 使用`usememo`装饰数据引用.
2. 使用`usecallback`装饰组件依赖的外部函数.

```js
React.useCallback(function foo(){}, []);

React.useMemo(() => function foo(){}, []);
```

```
function useToggle(initialValue) {
    const [value, setValue] = React.useState(initialValue);
    
    const toggle = React.useCallback(() => {
        setValue(v => !v);
    }, []);
    
    return [value, toggle];
}
```