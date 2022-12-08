## [Why React Re-Renders](https://www.joshwcomeau.com/react/why-react-re-renders/)
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