---
title: useEffect(fn, []) 不等于 componentDidMount()
date: 2020-09-03 20:08:35
categories: 
- React
- Hooks
tags:
- useEffect
- capturing
- react-hooks
---

> 从 React 16.8 版本开始引入了新增特性 Hooks，可以在不编写 Class 的情况下使用 state 以及其他 React 特性。所有 React 使用者都得经历从 Class 组件到 Hooks 模式的过渡期，但是这一时期令我们很容易走进一个误区。

## 误区：哪个 Hook 的功能 等价于【某个生命周期函数】？

问出这个问题证明我们的思维模式还停留在 ”我需要一个 Hook 来代替 componentDidMount( )“ 的阶段。但是 Hooks 是一种范式转换，从“生命周期和时间”的思维模式转变为“状态和与DOM的同步”的思维模式。如果尝试采用旧的思维模式并找到与其对应的钩子，可能会阻碍我们正确的理解和使用 Hooks，甚至带来一些问题。

这种思维模式会带来下面几个方面的问题：

- 它们实际上在原理上是不同的，所以如果把它们看作相同的，有可能不会得到期望的结果。 
- 从时间的角度考虑问题，比如“一旦mount就就调用一次useEffect”的思维模式，会阻碍你学习钩子。 
- 从类到钩子的重构并不意味着简单地用useEffect(fn，[]) 替换组件 componentDidMount( )。

## 执行时机不同

componentDidMount在组件挂载之后运行。如果立即（同步）设置 state，那么React就会触发一次额外的render，并将第二个render的响应用作初始UI，这样用户就不会看到闪烁。假设需要使用componentDidMount读取一个DOM元素的宽度，并希望更新state来反映宽度。事件的执行顺序应该是下面这样的：

1. 首次执行render
2. 此次 render 的返回值 将用于更新到真正的 Dom 中
3. componentDidMount 执行而且执行setState
4. state 变更导致 再次执行 render，而且返回了新的 返回值
5. 浏览器只显示了第二次 render 的返回值，这样可以避免闪屏

可以理解为上面的过程都是同步执行的，会阻塞到浏览器将真实DOM最终绘制到浏览器上，当我们需要它的时候，这样的工作模式是合理的。但大多数情况下，我们可以在UI Paint 完毕之后，再执行一些异步拉取数据之后setState之类的副作用。

useEffect 也是在挂载后运行，但是它更往后，它不会阻塞真事Dom的渲染，因为 useEffect 在 Paint (绘制)之后延迟异步运行。这意味着如果需要从DOM读取数据，然后同步设置state以生成新的UI，有可能会有闪烁的问题发生。React 也提供了 同步执行模式的 useLayoutEffect，它更加接近 componentDidMount( )的表现。

如果想通过同步设置状态来避免闪烁，那么可以使用useLayoutEffect。但是大部分时间都需要使用useEffect比较好。


## Props 和 State 的捕获（Capturing）

在React应用程序中，会存在许多的异步操作。当多个异步操作执行时，props 和 state 的值可能会有点混乱。
假设我们有很多异步代操作流程，在执行时需要知道 count 的状态： 

```javascript
class App extends React.Component {
  state = {
    count: 0
  };

  componentDidMount() {
    longResolve().then(() => {
      alert(this.state.count);
    });
  }

  render() {
    return (
      <div>
        <button
          onClick={() => {
            this.setState(state => ({ count: state.count + 1 }));
          }}
        >
          Count: {this.state.count}
        </button>
      </div>
    );
  }
}
```

页面加载完成后，在 longResolve 执行完成之前， 假设大概有几秒钟的时间单击按钮几次。如过我在此期间点了5次按钮，那么最后alert最终显示的也是最新的值，也是5次。

同样的场景，我们一开始用 hooks 重构的代码如下：

```javascript
function App() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    longResolve().then(() => {
      alert(count);
    });
  }, []);

  return (
    <div>
      <button
        onClick={() => {
          setCount(count + 1);
        }}
      >
        Count: {count}
      </button>
    </div>
  );
}
```

但是运行后会发现，它的表现和 class 版本有所不同，无论你在 longResolve 执行完毕前点击多少次，最后 alert 的 count 都是 0。

造成这种差异的原因是 useEffect 在创建时就已经捕获了count的值。当我们把回调函数赋给useEffect时，它会存在于内存中，在内存中它只知道 count 在创建时是0（由于闭包）。不管经过了多少时间，以及 count 这个时间内改变了多少次，闭包的本质是只跟创建闭包时这个值的状态有关，我们称之为“捕获”。而在 class组件中，componentDidMount( ) 没有闭包，每次读取的都是当前 count 的值。

情况可以等同于下面的函数来理解，在内存中，useEffect 的回调函数中的 count 再创建时赋予了初始值0，此时 count 的值不会再因外界的变化而受到影响。

```javascript
() => {
  const count = 0
  longResolve().then(() => {
    alert(count);
  });
}
```

[A Complete Guide to useEffect](https://overreacted.io/a-complete-guide-to-useeffect/)也提供了一个例子，演示了使用 hooks 后 setInterval 的实际表现和你的预期可能所有不同。

```javascript
// The class version:
class App extends React.Component {
  state = { count: 0 }

  componentDidMount() {
    setInterval(() => {
      this.setState({ count: this.state.count + 1 })
    }, 1000);
  }

  render() {
    return <div>{this.state.count}</div>
  }
}

// What we think is the same logic but rewritten as hooks:
function App() { 
  const [count, setCount] = useState(0)

  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1)
    }, 1000);
    return () => clearInterval(id)
  }, [])

  return <div>{count}</div>
}
```

Class 版本的代码，每隔一秒，显示的 count 会加1。然而用 hooks 实现的版本，显示count只会从 0 变为1，而且其实此时 setinerval 并没有停止，只是在不断的重复 setCount( 0 + 1 )， 因为对于 useEffect 回调函数内来说得到的 count 一直是 0。

这么看起来 Hooks 貌似造成一些之前不会有的麻烦，但是如果接受了这种模式，它反而能让你避免错误。讲到这里，需要再强调下，我们不是在讨论应该怎么使用 setInterval ，而是如何调整心智模型从 类组件 转变为 Hooks。

接下来一个重要的概念就是 **依赖数组（depends array）**

*解释起来很简单，如果某个逻辑依赖某个变量，那么它就应该出现在对应的依赖数组里。*

那么上面的 Hooks 实现就需要修改为，这样的话表现就跟我们的预期一致了。

```javascript
  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1)
    }, 1000)
    return () => clearInterval(id)
  }, [count])
```

这样的写法可以用下面的函数来理解，一旦依赖项发生变化，每次都销毁上一次的，新建一个新的。

```js
// Hey memory, we need you to store a function...
() => {
  const count = 0
  const id = setInterval(() => {
    setCount(count + 1)
  }, 1000)
  return () => clearInterval(id)}

// Later on when count changed...
// Hey memory, call the cleanup of that first function, then
// we need you to store another function...
() => {
  const count = 1
  const id = setInterval(() => {
    setCount(count + 1)
  }, 1000)
  return () => clearInterval(id)
}
```



## 捕获(capturing) 模式好还是不好

当使用捕获而不是当前值时，其实是可以可以避免一些错误的。以 [dan abramov的例子](https://codesandbox.io/s/pjqnl16lm7) 为例，展示了捕获是如何到达预期行为的，而不是之前类组件使用的每次都用当前值的模式。在这个例子中，我们可以查看不同的人物信息，并且点击follow 关注对应的人。如果我们点击 follow，在接口返回响应前修改查看人物信息，这时 class 组件返回的关注成功的消息其实是当前最新的，显然这是一个bug，因为我们当前最新切换的资料不是我们点击 follow 时对应的资料。所以有了时效性以及增加对应的依赖，反而能让我们把复杂的情况更容易理清，而不是一味的只用最新值。

[Dan 对此相关的一片文章](https://overreacted.io/how-are-function-components-different-from-classes/)

 

## 到底应该怎么通过 Hooks 重构已经用 class 实现的组件 

下面是一段很常见的 class组件的 代码

```javascript
class UserProfile extends React.Component {
  state = { user: null }

  componentDidMount() {
    getUser(this.props.uid).then(user => {
      this.setState({ user })
    })
  }

  render() {
    // ...
  }
}
```

很容易就发现一个需要改进的地方，如果 uid 发生变化我们应该怎么办，这种情况我们通常需要再写一个 componentDidUpdate 来配合处理，其实很容易忘记，而且内部处理的逻辑是一样的，而且都是副作用，代码看起来很冗余。

如果像文章开头所说，我们提前假定  useEffect(fn, [])  === componentDidMount（），我们就会直接得到如下的代码：

```javascript
function UserProfile({ uid }) {
  const [user, setUser] = useState(null)

  useEffect(() => {
    getUser(uid).then(user => {
      setUser(user)
    })
  }, []) //without `uid` in this array

  // ...
}
```

然后如果你记忆力比较好，会接着还一样的模式向自己提问：怎么样的 hook === didupdate，然后再实现一下。如果你忘记了，那代码就是遗漏了很大一部分功能。显然这不是正确的思维模式，如果我们提前了解 useEffect 的执行时机以及 对于props的捕获（capturing）特性，之后的思考是更连续，更符合 Hooks 模式的心智模型，就会有下面的重构后的代码：

```javascript
useEffect(() => {
  let isCurrent = true
  getUser(uid).then(user => {
    if (isCurrent) {
      setUser(user)
    }
  })
  return () => {
    isCurrent = false
  }
}, [uid])
```

## 总结 

使用 Hooks 模式进行编程时，我们需要忘记 生命周期和时间线 的概念，使用 **以状态为中心，以及对应状态发生变化时，哪些副作用需要重新执行** 的思想来进行编码。