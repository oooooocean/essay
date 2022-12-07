# React

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

> **Misconception: A component will re-render because its props change.**
> 
> When a component re-renders, it tries to re-render all descendants, regradless of whether they're being passed a particular state variable throught props or not.
> 
> Why?
>
> Because, It's hard for React to know, with 100% certainty, whether children depends, directly or indirectly, on state variable.

### 2. ignore certain re-render requests
- `React.memo`: "Hey, I know that this component is pure. You dont't need to re-render it unless its props change".
- `React.useContext`: context is sorta like "invisible props", or maybe "internal props".

> Memoization
> 
> React will remember the previous snapshot. If none of the props have changed, React will re-use that state snapshot rather than going through the trouble of generating a brand new one.

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