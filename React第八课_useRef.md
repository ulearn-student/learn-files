# 第八课：useRef —— 引用 DOM 与保存可变值

---

## 一、useRef 是什么

React 的核心思想是：数据变化触发界面更新。useState 就是这条线上的工具——改 state，组件重新渲染。

但实际开发中有两类需求 useState 处理不了：

第一类：直接操作 DOM 元素。比如让输入框自动获取焦点、控制视频播放、获取元素的宽高、滚动到指定位置。React 不推荐直接操作 DOM，但有些场景绕不开。

第二类：保存一个"可变的值"，但不想触发组件重新渲染。比如保存定时器的 ID、保存上一次的某个值、保存第三方库的实例。

useRef 就是为这两类需求设计的。

---

## 二、基础语法

```jsx
// 从 react 引入 useRef
import { useRef } from 'react';

function MyComponent() {
  // useRef(初始值) 返回一个对象：{ current: 初始值 }
  // 这个对象在组件整个生命周期里始终是同一个引用，不会被重新创建
  const myRef = useRef(null);

  // 读取值：myRef.current
  // 修改值：myRef.current = 新值
  // 修改不会触发组件重新渲染（这是关键区别）

  return <div>...</div>;
}
```

记住一句话：useRef 返回的是一个"盒子"，盒子里有一个 current 属性，你可以随便往里面放东西。盒子本身永远是同一个，但里面装的东西可以随时换。

---

## 三、用法一：操作 DOM 元素

最常见的场景。把 ref 挂到 JSX 元素上，就能拿到对应的真实 DOM 节点。

```jsx
import { useRef, useEffect } from 'react';

function AutoFocusInput() {
  // 创建一个 ref，初始值为 null
  // null 是因为组件刚定义时，DOM 还没生成
  const inputRef = useRef(null);

  // useEffect 在组件渲染完成后执行
  // 此时 DOM 已经生成，inputRef.current 就指向了真实的 input 元素
  useEffect(() => {
    // inputRef.current 是真实的 DOM 节点，可以调用原生 DOM 方法
    // .focus() 是 HTML input 元素的原生方法，让输入框获取焦点
    inputRef.current.focus();
  }, []); // 空数组 = 只在首次渲染后执行一次

  return (
    <div>
      {/* ref 属性是 React 的特殊属性，会自动把 DOM 节点赋值给 ref.current */}
      <input ref={inputRef} type="text" placeholder="页面加载后我会自动聚焦" />
    </div>
  );
}
```

执行流程：

1. 组件首次执行，useRef 创建 inputRef 对象，current 是 null
2. JSX 返回，React 开始渲染真实 DOM
3. DOM 渲染完成，React 把 input 元素赋值给 inputRef.current
4. useEffect 执行（在 DOM 渲染之后），调用 focus 方法

---

## 四、用法一的真实场景

场景 1：表单聚焦

```jsx
function LoginForm() {
  // 给账号输入框和密码输入框各创建一个 ref
  const usernameRef = useRef(null);
  const passwordRef = useRef(null);

  function handleSubmit() {
    // 校验账号是否为空
    if (!usernameRef.current.value) {
      // 提示错误并让账号输入框聚焦，提升用户体验
      alert('请输入账号');
      usernameRef.current.focus();
      return;
    }

    // 校验密码是否为空
    if (!passwordRef.current.value) {
      alert('请输入密码');
      passwordRef.current.focus();
      return;
    }

    // 实际项目里这里调用登录接口
    console.log('账号:', usernameRef.current.value);
    console.log('密码:', passwordRef.current.value);
  }

  return (
    <div>
      <input ref={usernameRef} type="text" placeholder="账号" />
      <input ref={passwordRef} type="password" placeholder="密码" />
      <button onClick={handleSubmit}>登录</button>
    </div>
  );
}
```

场景 2：滚动到指定元素

```jsx
function LongPage() {
  // 创建一个指向底部元素的 ref
  const bottomRef = useRef(null);

  function scrollToBottom() {
    // scrollIntoView 是 DOM 原生方法，让元素滚动到可视区域
    // behavior: 'smooth' 是平滑滚动，不写就是瞬间跳转
    bottomRef.current.scrollIntoView({ behavior: 'smooth' });
  }

  return (
    <div>
      <button onClick={scrollToBottom}>跳到底部</button>

      {/* 模拟很长的内容 */}
      <div style={{ height: '2000px' }}>很长的内容...</div>

      {/* 把 ref 挂到目标位置 */}
      <div ref={bottomRef}>这里是底部</div>
    </div>
  );
}
```

场景 3：控制视频/音频播放

```jsx
function VideoPlayer() {
  const videoRef = useRef(null);

  // play 和 pause 是 video 元素的原生方法
  function play() {
    videoRef.current.play();
  }

  function pause() {
    videoRef.current.pause();
  }

  // currentTime 是 video 的属性，单位是秒
  function jumpTo10s() {
    videoRef.current.currentTime = 10;
  }

  return (
    <div>
      <video ref={videoRef} src="/demo.mp4" width="400" />
      <button onClick={play}>播放</button>
      <button onClick={pause}>暂停</button>
      <button onClick={jumpTo10s}>跳到 10 秒</button>
    </div>
  );
}
```

场景 4：获取元素尺寸

```jsx
function MeasureBox() {
  const boxRef = useRef(null);

  function measure() {
    // getBoundingClientRect 返回元素的位置和尺寸信息
    // 包括 width、height、top、left、right、bottom 等
    const rect = boxRef.current.getBoundingClientRect();
    console.log('宽度:', rect.width);
    console.log('高度:', rect.height);
  }

  return (
    <div>
      <div ref={boxRef} style={{ width: '200px', height: '100px', background: 'red' }}>
        我是一个盒子
      </div>
      <button onClick={measure}>测量我</button>
    </div>
  );
}
```

---

## 五、用法二：保存可变值（不触发重新渲染）

这是 useRef 容易被忽视的能力，但企业项目里非常常用。

useRef 的 current 可以保存任何值——数字、对象、函数、定时器 ID、第三方实例。修改它不会触发组件重新渲染。

```jsx
import { useState, useRef, useEffect } from 'react';

function Timer() {
  const [count, setCount] = useState(0);

  // 保存定时器 ID 用 useRef，而不是 useState
  // 因为定时器 ID 只用来在清除时引用，不需要显示在界面上
  // 用 useState 会触发额外的重新渲染，浪费性能
  const timerRef = useRef(null);

  function start() {
    // 防止重复启动定时器
    if (timerRef.current) return;

    // setInterval 返回一个 ID，存到 ref 里
    // 函数式更新 setCount(c => c + 1) 避免闭包陷阱
    timerRef.current = setInterval(() => {
      setCount(c => c + 1);
    }, 1000);
  }

  function stop() {
    // 用保存的 ID 清除定时器
    clearInterval(timerRef.current);
    // 清空引用，标记定时器已停止
    timerRef.current = null;
  }

  // 组件卸载时清理定时器，避免内存泄漏（企业项目必做）
  useEffect(() => {
    // 返回的函数会在组件卸载时执行
    return () => {
      if (timerRef.current) {
        clearInterval(timerRef.current);
      }
    };
  }, []);

  return (
    <div>
      <p>计数：{count}</p>
      <button onClick={start}>开始</button>
      <button onClick={stop}>停止</button>
    </div>
  );
}
```

为什么不用 useState 存定时器 ID？

useState 改变会触发组件重新渲染。定时器 ID 只是内部使用，没必要因为它的变化触发渲染。useRef 存的值变化不会触发渲染，更高效。

---

## 六、useRef vs useState —— 关键对比

这是面试高频考点，必须吃透。

| 特性 | useState | useRef |
| --- | --- | --- |
| 值变化是否触发重新渲染 | 是 | 否 |
| 读取语法 | state | ref.current |
| 修改语法 | setState(新值) | ref.current = 新值 |
| 是否同步更新 | 异步（批量更新） | 同步（立即生效） |
| 适用场景 | 影响 UI 的数据 | 不影响 UI 的数据、DOM 引用 |

判断标准：这个值变化时，界面需要跟着变吗？

需要变 → useState（比如表单输入、用户列表、按钮文字）

不需要变 → useRef（比如定时器 ID、上一次的值、DOM 节点、第三方库实例）

---

## 七、易错点：useRef 不能用来"避免渲染"

新手常犯的错误：以为只要把所有数据放 useRef 就能优化性能。

```jsx
// 错误示范：把要显示的数据放 useRef
function BadCounter() {
  const countRef = useRef(0);

  function increment() {
    // 修改了 ref，但组件不会重新渲染
    countRef.current += 1;
    console.log(countRef.current); // 控制台能看到数字变了
  }

  // 但界面上的数字永远是 0，因为组件没重新渲染
  return (
    <div>
      <p>计数：{countRef.current}</p>
      <button onClick={increment}>+1</button>
    </div>
  );
}
```

记住：界面上要显示的数据必须用 useState。useRef 适合存"幕后"的数据。

---

## 八、保存上一次的值（经典场景）

```jsx
import { useState, useRef, useEffect } from 'react';

function PrevValueDemo() {
  // 当前的 count
  const [count, setCount] = useState(0);

  // 用 ref 保存上一次的 count
  const prevCountRef = useRef(0);

  // useEffect 在每次渲染之后执行
  // 把当前的 count 存到 ref 里，下次渲染时这个值就是"上一次的值"
  useEffect(() => {
    prevCountRef.current = count;
  }, [count]); // 依赖 count，count 变化时执行

  return (
    <div>
      <p>当前值：{count}</p>
      <p>上一次的值：{prevCountRef.current}</p>
      <button onClick={() => setCount(count + 1)}>+1</button>
    </div>
  );
}
```

执行流程：

1. 首次渲染，count=0，prevCountRef.current=0，显示"当前值 0，上一次 0"
2. 渲染后 useEffect 执行，prevCountRef.current 被设为 0
3. 点击按钮，count 变成 1，组件重新渲染
4. 渲染时读取的还是上一次的 prevCountRef.current（即 0），显示"当前值 1，上一次 0"
5. 渲染后 useEffect 再次执行，prevCountRef.current 变成 1
6. 再次点击，count 变成 2，显示"当前值 2，上一次 1"

---

## 九、forwardRef —— 给子组件传 ref

正常情况下，ref 只能挂到原生 DOM 元素上。如果想把 ref 挂到自定义组件上，需要用 forwardRef。

```jsx
import { useRef, forwardRef } from 'react';

// 普通组件不能接收 ref
// 必须用 forwardRef 包裹，才能让父组件的 ref 穿透到内部的 DOM
// forwardRef 的回调函数接收两个参数：props 和 ref
const MyInput = forwardRef((props, ref) => {
  return (
    <div>
      <label>{props.label}</label>
      {/* 把父组件传来的 ref 挂到真实的 input 上 */}
      <input ref={ref} type="text" />
    </div>
  );
});

// 父组件使用
function Parent() {
  const inputRef = useRef(null);

  function handleClick() {
    // 通过 ref 直接操作子组件内部的 input
    inputRef.current.focus();
  }

  return (
    <div>
      {/* 给子组件传 ref，会被 forwardRef 接收 */}
      <MyInput label="用户名" ref={inputRef} />
      <button onClick={handleClick}>聚焦输入框</button>
    </div>
  );
}
```

注意：React 19 之后 forwardRef 已经不再需要，可以直接通过 props 传递 ref。但 React 18 及以下的企业项目仍然广泛使用 forwardRef，必须掌握。

---

## 十、useImperativeHandle —— 暴露指定方法给父组件

forwardRef 让父组件能拿到子组件的 DOM。但有时候我们不想暴露整个 DOM，只想暴露几个特定的方法。

```jsx
import { useRef, forwardRef, useImperativeHandle } from 'react';

// 子组件：自定义输入框
const FancyInput = forwardRef((props, ref) => {
  // 子组件内部使用的 ref
  const inputRef = useRef(null);

  // useImperativeHandle：自定义暴露给父组件的内容
  // 第一个参数：父组件传来的 ref
  // 第二个参数：返回一个对象，对象里的属性就是父组件能调用的方法
  useImperativeHandle(ref, () => ({
    // 自定义聚焦方法
    focus: () => {
      inputRef.current.focus();
    },
    // 自定义清空方法
    clear: () => {
      inputRef.current.value = '';
    },
    // 自定义获取值的方法
    getValue: () => {
      return inputRef.current.value;
    }
  }));

  return <input ref={inputRef} type="text" />;
});

// 父组件
function Parent() {
  const fancyRef = useRef(null);

  function handleClick() {
    // 父组件只能调用 useImperativeHandle 暴露的方法
    // 不能直接调用 fancyRef.current.value（因为没暴露 DOM）
    fancyRef.current.focus();

    setTimeout(() => {
      console.log('当前值：', fancyRef.current.getValue());
      fancyRef.current.clear();
    }, 2000);
  }

  return (
    <div>
      <FancyInput ref={fancyRef} />
      <button onClick={handleClick}>操作输入框</button>
    </div>
  );
}
```

useImperativeHandle 的价值：封装组件，避免父组件直接操作子组件内部的 DOM，保持组件的封装性。这是企业级组件库（Ant Design、Element 等）的常用模式。

---

## 十一、企业级真实场景

场景 1：集成第三方库（如 ECharts、Three.js）

```jsx
import { useRef, useEffect } from 'react';
import * as echarts from 'echarts';

function ChartComponent() {
  // 容器 DOM 引用
  const containerRef = useRef(null);

  // 保存 echarts 实例，避免每次渲染都重新初始化
  // 实例放 useRef 是因为它不影响 UI，但需要在多个函数里共用
  const chartInstanceRef = useRef(null);

  useEffect(() => {
    // 初始化图表，绑定到 DOM 容器
    chartInstanceRef.current = echarts.init(containerRef.current);

    // 设置图表配置
    chartInstanceRef.current.setOption({
      title: { text: '销售数据' },
      xAxis: { data: ['周一', '周二', '周三'] },
      yAxis: {},
      series: [{ type: 'bar', data: [10, 20, 30] }]
    });

    // 组件卸载时销毁实例，防止内存泄漏
    return () => {
      chartInstanceRef.current.dispose();
    };
  }, []);

  return <div ref={containerRef} style={{ width: '600px', height: '400px' }} />;
}
```

场景 2：防止重复点击（节流）

```jsx
function SubmitButton() {
  // 用 ref 标记是否正在提交，不需要触发渲染
  const isSubmittingRef = useRef(false);

  async function handleSubmit() {
    // 如果正在提交，直接返回
    if (isSubmittingRef.current) {
      console.log('正在提交，请勿重复点击');
      return;
    }

    // 标记开始提交
    isSubmittingRef.current = true;

    try {
      // 模拟接口请求
      await fetch('/api/submit');
      console.log('提交成功');
    } finally {
      // 无论成功失败，3 秒后才允许再次提交
      setTimeout(() => {
        isSubmittingRef.current = false;
      }, 3000);
    }
  }

  return <button onClick={handleSubmit}>提交</button>;
}
```

为什么不用 useState？因为这个值不需要显示在界面上，用 useState 会引发不必要的重新渲染。

场景 3：表格滚动到指定行

```jsx
function DataTable({ data, highlightId }) {
  // 用对象保存所有行的 ref，key 是行的 id
  const rowRefsMap = useRef({});

  useEffect(() => {
    // 当 highlightId 变化时，滚动到那一行
    if (highlightId && rowRefsMap.current[highlightId]) {
      rowRefsMap.current[highlightId].scrollIntoView({
        behavior: 'smooth',
        block: 'center'
      });
    }
  }, [highlightId]);

  return (
    <table>
      <tbody>
        {data.map(row => (
          <tr
            key={row.id}
            // 把每一行的 DOM 存到 ref 对象里，用 id 作为 key
            // 这种写法叫"ref 回调"，是 ref 的高级用法
            ref={el => {
              if (el) {
                rowRefsMap.current[row.id] = el;
              } else {
                // 元素卸载时清理引用
                delete rowRefsMap.current[row.id];
              }
            }}
            style={{ background: row.id === highlightId ? 'yellow' : 'white' }}
          >
            <td>{row.name}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

---

## 十二、ref 回调（高级用法）

除了把 useRef 创建的对象赋给 ref 属性，还可以传一个函数。

```jsx
function CallbackRef() {
  function measureNode(node) {
    // node 是 DOM 元素（元素挂载时）或 null（元素卸载时）
    if (node) {
      console.log('元素挂载了，宽度：', node.getBoundingClientRect().width);
    } else {
      console.log('元素卸载了');
    }
  }

  return <div ref={measureNode}>测量我</div>;
}
```

应用场景：需要在元素挂载/卸载时立即执行操作，比 useEffect 更早。useEffect 在所有渲染完成后才执行，ref 回调在元素挂载那一刻就执行。

---

## 十三、常见陷阱

陷阱 1：在渲染过程中读写 ref.current

```jsx
function Bad() {
  const ref = useRef(0);

  // 不要在渲染期间修改 ref（破坏 React 的纯函数原则）
  ref.current = ref.current + 1;

  return <div>{ref.current}</div>;
}
```

应该在事件处理函数、useEffect、回调里修改 ref。渲染过程中保持纯净。

陷阱 2：DOM 还没生成就访问

```jsx
function Bad() {
  const ref = useRef(null);

  // 错误：组件首次渲染时，JSX 还没变成真实 DOM
  // ref.current 此时是 null，调用 focus 会报错
  ref.current.focus();

  return <input ref={ref} />;
}

function Good() {
  const ref = useRef(null);

  // 正确：在 useEffect 里访问 DOM，此时 DOM 已渲染完成
  useEffect(() => {
    ref.current.focus();
  }, []);

  return <input ref={ref} />;
}
```

陷阱 3：把不该用 ref 的数据用 ref

```jsx
// 错误：商品列表是要显示的数据，必须用 useState
const productsRef = useRef([]);

// 正确
const [products, setProducts] = useState([]);
```

---

## 十四、与其他框架的对比

Vue 3 的 ref：和 React 的 useRef 名字一样但概念不同。Vue 的 ref 是响应式的，改值会触发界面更新。React 的 useRef 改值不会触发更新。Vue 的"不响应式"用法对应的是 React 的 useRef，Vue 的 ref 对应的是 React 的 useState。

Angular 的 ViewChild：和 useRef 的 DOM 引用用法类似，用装饰器获取模板里的元素。

原生 JS 的 document.querySelector：useRef 的本质就是封装了"获取 DOM 引用"这个操作，但它是声明式的、跟着组件生命周期走，比手动 querySelector 更安全。

---

## 十五、企业项目中的实际选择

什么时候用 useRef？

操作 DOM：聚焦、滚动、测量、播放视频、集成第三方库。

保存不需要触发渲染的值：定时器 ID、节流锁、上一次的值、第三方实例。

性能优化：避免因为某些后台数据变化导致整个组件重新渲染。

什么时候不用？

数据需要显示在界面上 → 用 useState。

数据需要在组件间共享 → 用 useContext 或状态管理库（Redux、Zustand）。

复杂派生状态 → 用 useMemo、useCallback。

---

## 十六、知识体系总结

useRef 的两大用途：

第一，引用 DOM 元素：创建 ref 对象，挂到 JSX 的 ref 属性上，在 useEffect 里访问 ref.current 操作 DOM。

第二，保存可变值：保存任何不需要触发渲染的值，比如定时器 ID、第三方实例、节流锁、上一次的值。

相关 API：

forwardRef：让自定义组件能接收父组件传来的 ref。React 19 之后可以直接通过 props 传递。

useImperativeHandle：自定义子组件暴露给父组件的方法，封装组件细节。

ref 回调：传函数给 ref 属性，元素挂载/卸载时立即执行。

核心区别：

useRef 改值不触发渲染，useState 改值触发渲染。这是判断用哪个的唯一标准。

---

## 十七、作业

实现一个"防止重复提交的登录表单"：

1. 用户名输入框页面加载后自动聚焦（useRef + useEffect）
2. 用户名为空时提交，自动聚焦到用户名输入框（useRef）
3. 密码为空时提交，自动聚焦到密码输入框（useRef）
4. 点击提交后 3 秒内不能再次提交（useRef 存节流锁）
5. 提交成功后清空输入框（通过 ref 操作 DOM）

进阶要求：

把用户名输入框封装成一个 FancyInput 组件，用 forwardRef + useImperativeHandle 暴露 focus 和 clear 方法给父组件。

写完发给我看，或者遇到卡点随时问。
