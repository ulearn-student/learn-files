# 第九课：useContext —— 跨组件传数据

---

## 一、为什么需要 useContext

学完 Props 后你已经能在父子组件之间传数据。但实际项目里，经常会遇到这样的结构：

```
App
 └── Layout
      └── Sidebar
           └── UserMenu
                └── Avatar  ← 需要用到登录用户的信息
```

App 里有登录用户的信息，Avatar 要用。按 Props 的写法，必须一层一层往下传：

App 把 user 传给 Layout，Layout 传给 Sidebar，Sidebar 传给 UserMenu，UserMenu 传给 Avatar。

中间的 Layout、Sidebar、UserMenu 自己根本不用 user，只是当了快递员。这种现象叫 props drilling（属性钻取），是 React 项目里最常见的痛点。

useContext 的作用就是：让任何一个深层组件都能直接拿到全局共享的数据，跳过中间所有层。

---

## 二、三个核心 API

useContext 体系由三个 API 组成：

createContext：创建一个 Context 对象（数据通道）

Context.Provider：在组件树某一层提供数据（数据源）

useContext：在任何子组件里取出数据（数据消费）

记住一个类比：Provider 像 WiFi 路由器，Context 是 WiFi 信号，useContext 是手机连接 WiFi。路由器覆盖范围内的设备都能直接连上，不需要拉网线。

---

## 三、基础用法

第一步：创建 Context 文件

```jsx
// src/contexts/UserContext.js

// createContext 是 React 提供的 API，用来创建一个 Context 对象
import { createContext } from 'react';

// createContext(初始值)
// 初始值在没有被 Provider 包裹时使用，正常情况下会被 Provider 的 value 覆盖
// 初始值的作用主要是给 TypeScript 提供类型提示，运行时基本不会用到
export const UserContext = createContext(null);
```

第二步：在父组件用 Provider 提供数据

```jsx
// App.jsx
import { useState } from 'react';
import { UserContext } from './contexts/UserContext';
import Layout from './Layout';

function App() {
  // 用户信息存在 App 这一层
  const [user, setUser] = useState({
    name: '李如杰',
    avatar: '/avatar.jpg',
    level: 'VIP'
  });

  return (
    // Provider 是 Context 自带的组件，必须用它包裹要共享数据的范围
    // value 属性是要共享的数据，可以是任何类型（对象、数组、函数等）
    // 包裹范围内的所有子组件（无论多深）都能通过 useContext 拿到 value
    <UserContext.Provider value={user}>
      <Layout />
    </UserContext.Provider>
  );
}

export default App;
```

第三步：在任意深层组件取出数据

```jsx
// src/components/Avatar.jsx
import { useContext } from 'react';
import { UserContext } from '../contexts/UserContext';

function Avatar() {
  // useContext(Context对象) 返回 Provider 提供的 value
  // 不需要从父组件接 props，直接拿
  const user = useContext(UserContext);

  return (
    <div>
      <img src={user.avatar} alt={user.name} />
      <span>{user.name}（{user.level}）</span>
    </div>
  );
}

export default Avatar;
```

中间的 Layout、Sidebar、UserMenu 不需要做任何事，user 数据直接穿透到 Avatar。

---

## 四、传函数：让子组件能修改全局数据

光读不够，子组件经常需要修改全局数据（比如登出、切换主题）。把修改函数也放进 value 里。

```jsx
// App.jsx
function App() {
  const [user, setUser] = useState({ name: '李如杰', level: 'VIP' });

  // 定义登出函数
  function logout() {
    setUser(null);
  }

  // 定义更新用户信息的函数
  function updateUser(newInfo) {
    // 用展开运算符合并新旧数据
    setUser(prev => ({ ...prev, ...newInfo }));
  }

  return (
    // value 改成对象，把 user 和操作函数一起传下去
    // 这样子组件既能读，又能改
    <UserContext.Provider value={{ user, logout, updateUser }}>
      <Layout />
    </UserContext.Provider>
  );
}
```

子组件使用：

```jsx
// LogoutButton.jsx
import { useContext } from 'react';
import { UserContext } from '../contexts/UserContext';

function LogoutButton() {
  // 解构出需要用的部分
  const { user, logout } = useContext(UserContext);

  // 没登录就不显示按钮
  if (!user) return null;

  return <button onClick={logout}>退出登录</button>;
}
```

---

## 五、经典场景一：主题切换（明暗模式）

```jsx
// src/contexts/ThemeContext.js
import { createContext, useState } from 'react';

// 创建主题 Context
export const ThemeContext = createContext(null);

// 自定义 Provider 组件：把状态和逻辑封装在一起
// 这样 App.jsx 不会变得臃肿，主题相关代码全部在这一个文件里
export function ThemeProvider({ children }) {
  // 主题状态：'light' 或 'dark'
  const [theme, setTheme] = useState('light');

  // 切换主题的函数
  function toggleTheme() {
    setTheme(prev => prev === 'light' ? 'dark' : 'light');
  }

  // value 包含状态和操作函数
  // children 是 ThemeProvider 包裹的所有子组件
  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}
```

在 App.jsx 里使用：

```jsx
import { ThemeProvider } from './contexts/ThemeContext';
import Layout from './Layout';

function App() {
  return (
    // 用自定义 Provider 包裹，整个应用都能用主题
    <ThemeProvider>
      <Layout />
    </ThemeProvider>
  );
}
```

任何深层组件都能直接用：

```jsx
// 任意位置的按钮
import { useContext } from 'react';
import { ThemeContext } from '../contexts/ThemeContext';

function ThemeToggle() {
  const { theme, toggleTheme } = useContext(ThemeContext);

  return (
    <button
      onClick={toggleTheme}
      style={{
        // 根据主题动态改样式
        background: theme === 'light' ? '#fff' : '#333',
        color: theme === 'light' ? '#000' : '#fff'
      }}
    >
      当前：{theme === 'light' ? '☀️ 浅色' : '🌙 深色'}
    </button>
  );
}
```

---

## 六、经典场景二：购物车（电商必用）

跨境电商场景里，加购物车按钮、顶部购物车数量、结算页面、订单确认页，都需要访问购物车数据。如果用 props 传，会传爆。

```jsx
// src/contexts/CartContext.js
import { createContext, useState } from 'react';

export const CartContext = createContext(null);

export function CartProvider({ children }) {
  // 购物车数据，每项格式：{ id, name, price, quantity }
  const [cartItems, setCartItems] = useState([]);

  // 添加商品到购物车
  function addToCart(product) {
    setCartItems(prev => {
      // 查找购物车里是否已有这个商品
      const existing = prev.find(item => item.id === product.id);

      if (existing) {
        // 已存在：数量 +1
        return prev.map(item =>
          item.id === product.id
            ? { ...item, quantity: item.quantity + 1 }
            : item
        );
      }

      // 不存在：新增一条，数量为 1
      return [...prev, { ...product, quantity: 1 }];
    });
  }

  // 从购物车移除商品
  function removeFromCart(productId) {
    setCartItems(prev => prev.filter(item => item.id !== productId));
  }

  // 修改某个商品的数量
  function updateQuantity(productId, quantity) {
    // 数量小于等于 0 时直接移除
    if (quantity <= 0) {
      removeFromCart(productId);
      return;
    }

    setCartItems(prev => prev.map(item =>
      item.id === productId ? { ...item, quantity } : item
    ));
  }

  // 计算总价（派生数据，每次渲染重新计算）
  // 如果性能有问题可以用 useMemo 优化（第十课内容）
  const totalPrice = cartItems.reduce(
    (sum, item) => sum + item.price * item.quantity,
    0
  );

  // 计算总数量
  const totalCount = cartItems.reduce(
    (sum, item) => sum + item.quantity,
    0
  );

  // 把所有数据和方法暴露给子组件
  return (
    <CartContext.Provider value={{
      cartItems,
      addToCart,
      removeFromCart,
      updateQuantity,
      totalPrice,
      totalCount
    }}>
      {children}
    </CartContext.Provider>
  );
}
```

商品列表页使用：

```jsx
function ProductCard({ product }) {
  // 只解构需要的方法，不需要的不解构
  const { addToCart } = useContext(CartContext);

  return (
    <div>
      <h3>{product.name}</h3>
      <p>¥{product.price}</p>
      <button onClick={() => addToCart(product)}>加入购物车</button>
    </div>
  );
}
```

顶部购物车角标使用：

```jsx
function CartBadge() {
  const { totalCount } = useContext(CartContext);

  return (
    <div>
      🛒 购物车
      {/* 数量大于 0 才显示角标 */}
      {totalCount > 0 && <span className="badge">{totalCount}</span>}
    </div>
  );
}
```

结算页使用：

```jsx
function Checkout() {
  const { cartItems, totalPrice, updateQuantity } = useContext(CartContext);

  return (
    <div>
      <h2>结算</h2>
      {cartItems.map(item => (
        <div key={item.id}>
          <span>{item.name}</span>
          <button onClick={() => updateQuantity(item.id, item.quantity - 1)}>-</button>
          <span>{item.quantity}</span>
          <button onClick={() => updateQuantity(item.id, item.quantity + 1)}>+</button>
        </div>
      ))}
      <p>总价：¥{totalPrice}</p>
    </div>
  );
}
```

---

## 七、多个 Context 嵌套

一个项目通常有多个全局数据：用户信息、主题、语言、购物车……Provider 可以多层嵌套。

```jsx
function App() {
  return (
    // 嵌套顺序无所谓，但建议把变化频率低的放外层、变化频率高的放内层
    // 因为外层 Provider 重渲染会带动所有内层
    <UserProvider>
      <ThemeProvider>
        <LanguageProvider>
          <CartProvider>
            <Layout />
          </CartProvider>
        </LanguageProvider>
      </ThemeProvider>
    </UserProvider>
  );
}
```

如果嵌套层次太多变成"金字塔"，可以封装成一个 AppProviders 组件：

```jsx
// src/contexts/index.jsx
function AppProviders({ children }) {
  return (
    <UserProvider>
      <ThemeProvider>
        <LanguageProvider>
          <CartProvider>
            {children}
          </CartProvider>
        </LanguageProvider>
      </ThemeProvider>
    </UserProvider>
  );
}

// App.jsx 变干净了
function App() {
  return (
    <AppProviders>
      <Layout />
    </AppProviders>
  );
}
```

这是企业项目里的标准写法。

---

## 八、Context + useReducer（复杂状态管理）

当全局状态变得复杂（操作多、逻辑复杂），用 useState 会让 Provider 变得很乱。这时候用 useReducer 更合适。

useReducer 是 useState 的进阶版，把所有状态修改逻辑集中到一个 reducer 函数里。

```jsx
// src/contexts/CartContext.js
import { createContext, useReducer } from 'react';

// reducer 函数：根据 action 类型决定如何修改 state
// 这是 Redux 的核心思想，企业项目用得非常多
function cartReducer(state, action) {
  // action 是一个对象，至少包含 type 字段，说明要做什么
  switch (action.type) {
    case 'ADD_ITEM': {
      // 添加商品
      const existing = state.find(item => item.id === action.payload.id);
      if (existing) {
        return state.map(item =>
          item.id === action.payload.id
            ? { ...item, quantity: item.quantity + 1 }
            : item
        );
      }
      return [...state, { ...action.payload, quantity: 1 }];
    }

    case 'REMOVE_ITEM':
      // 删除商品
      return state.filter(item => item.id !== action.payload);

    case 'UPDATE_QUANTITY':
      // 修改数量
      return state.map(item =>
        item.id === action.payload.id
          ? { ...item, quantity: action.payload.quantity }
          : item
      );

    case 'CLEAR':
      // 清空购物车
      return [];

    default:
      // 未知 action 类型，直接返回原状态
      return state;
  }
}

export const CartContext = createContext(null);

export function CartProvider({ children }) {
  // useReducer(reducer 函数, 初始状态)
  // 返回 [当前状态, dispatch 函数]
  // dispatch 用来触发状态修改，类似 setState
  const [cartItems, dispatch] = useReducer(cartReducer, []);

  // 封装成简单的方法，子组件调用时不需要知道 action 的细节
  const addToCart = (product) => dispatch({ type: 'ADD_ITEM', payload: product });
  const removeFromCart = (id) => dispatch({ type: 'REMOVE_ITEM', payload: id });
  const updateQuantity = (id, quantity) => dispatch({
    type: 'UPDATE_QUANTITY',
    payload: { id, quantity }
  });
  const clearCart = () => dispatch({ type: 'CLEAR' });

  return (
    <CartContext.Provider value={{
      cartItems,
      addToCart,
      removeFromCart,
      updateQuantity,
      clearCart
    }}>
      {children}
    </CartContext.Provider>
  );
}
```

useState vs useReducer 选择标准：

状态简单（一两个字段）→ useState

状态复杂（对象 / 数组 / 多种操作）→ useReducer

需要可追溯的状态变化历史 → useReducer

未来可能要迁移到 Redux → useReducer（语法几乎一样）

---

## 九、性能陷阱（必须知道）

Context 有一个出名的性能问题：Provider 的 value 一变，所有用到这个 Context 的组件全部重新渲染，无论它们用到的字段变没变。

```jsx
// 反面教材
function App() {
  const [user, setUser] = useState({...});
  const [theme, setTheme] = useState('light');

  // 每次渲染都会创建一个新对象，value 引用变了
  // 即使 user 和 theme 都没变，所有消费者都会重新渲染
  return (
    <AppContext.Provider value={{ user, theme, setUser, setTheme }}>
      ...
    </AppContext.Provider>
  );
}
```

解决方案 1：拆分 Context

把不相关的数据拆到不同的 Context：

```jsx
// 用户信息一个 Context
<UserContext.Provider value={user}>
  {/* 主题一个 Context */}
  <ThemeContext.Provider value={theme}>
    <App />
  </ThemeContext.Provider>
</UserContext.Provider>
```

这样改 user 不会触发主题消费者重渲染。

解决方案 2：用 useMemo 稳定 value 引用（第十课会详细讲）

```jsx
import { useMemo } from 'react';

function App() {
  const [user, setUser] = useState({...});

  // 用 useMemo 包裹，只在 user 变化时才重新创建对象
  const value = useMemo(() => ({ user, setUser }), [user]);

  return (
    <UserContext.Provider value={value}>...</UserContext.Provider>
  );
}
```

解决方案 3：把读和写拆成两个 Context（高级技巧）

```jsx
const UserStateContext = createContext(null);    // 只放 state
const UserActionsContext = createContext(null);  // 只放修改函数

// 只读用户信息的组件订阅 UserStateContext
// 只调用修改函数的组件订阅 UserActionsContext
// 修改函数引用稳定，永远不变 → 调用方永远不会因为它重渲染
```

---

## 十、Context 的边界（什么时候不该用）

Context 不是万能药，下面这些场景不建议用：

频繁变化的数据：比如鼠标位置、输入框文字。每次变化都会触发整个订阅树重渲染，性能差。这种数据应该用 useState 局部管理，或者用专门的状态管理库。

只在父子组件间传的数据：直接用 props 更清晰。Context 适合"跨多层"的传递。

服务端数据缓存：比如接口返回的数据列表。这种用 React Query / SWR 更合适，它们有自动缓存、失效、刷新机制。

---

## 十一、企业级方案对比

| 方案 | 适用场景 | 优点 | 缺点 |
| --- | --- | --- | --- |
| useState（局部） | 单个组件内的状态 | 简单 | 不能跨组件 |
| Context + useState | 小型项目全局状态 | React 原生，零依赖 | 性能问题、写法繁琐 |
| Context + useReducer | 中型项目复杂状态 | 集中管理逻辑 | 模板代码多 |
| Redux / Redux Toolkit | 大型企业项目 | 生态成熟、调试工具强 | 学习成本高、代码量大 |
| Zustand | 中小型现代项目 | API 简洁、无 Provider | 生态相对小 |
| Jotai / Recoil | 原子化状态 | 细粒度更新 | 思维方式不同 |
| React Query / SWR | 服务端数据 | 自动缓存、请求去重 | 不适合纯客户端状态 |

实际经验：

小项目（只用 React 自带）→ Context + useState 够用

中型项目（个人 / 小团队）→ Zustand 是性价比最高的选择

大型企业项目 → Redux Toolkit（业界标准，简历加分）

服务端数据 → 一律用 React Query / SWR，不要用 Context 包接口数据

第十三课会讲 Zustand，那是目前国内外新项目的主流。

---

## 十二、企业开发最佳实践

不要把所有数据塞进一个 Context。按业务模块拆：UserContext、ThemeContext、CartContext、NotificationContext，各自独立。

每个 Context 配一个自定义 Hook。用起来更顺手，也方便后期重构：

```jsx
// 在 ThemeContext.js 里导出自定义 hook
export function useTheme() {
  const context = useContext(ThemeContext);
  // 防御性编程：检测是否忘记包 Provider
  if (!context) {
    throw new Error('useTheme 必须在 ThemeProvider 内部使用');
  }
  return context;
}

// 子组件使用时更清爽
function MyButton() {
  const { theme, toggleTheme } = useTheme();
  // ...
}
```

Provider 文件结构推荐：

```
src/
  contexts/
    UserContext.jsx       ← Context + Provider + 自定义 Hook
    ThemeContext.jsx
    CartContext.jsx
    index.jsx             ← 统一导出 AppProviders
```

把 Provider 和 Context 写在同一个文件，对外只导出 Provider 组件和自定义 Hook，不导出原始 Context 对象。这样调用方永远走自定义 Hook，便于统一管理。

---

## 十三、知识体系总结

useContext 解决 props drilling 问题，实现跨组件数据共享。

三个核心 API：

createContext 创建数据通道

Context.Provider 提供数据

useContext 消费数据

进阶组合：

Context + useState 简单全局状态

Context + useReducer 复杂全局状态

Context + 自定义 Hook 封装最佳实践

性能要点：

Provider 的 value 一变，所有消费者全部重渲染

解决办法：拆分 Context、useMemo 稳定引用、读写分离

企业级选择：

小项目用 Context 就够

中大型项目用 Zustand / Redux Toolkit

服务端数据用 React Query

---

## 十四、作业

实现一个简单的用户登录系统：

1. 创建 AuthContext，包含 user（用户信息）、login（登录函数）、logout（登出函数）
2. 用自定义 Hook useAuth 封装 useContext 调用
3. 在 App 顶部包 AuthProvider
4. 创建一个 LoginButton 组件，没登录时显示"登录"按钮，点击后调用 login 设置一个假用户
5. 创建一个 UserInfo 组件，显示当前用户名 + 登出按钮
6. 把 LoginButton 和 UserInfo 放在不同的位置，验证它们都能正常工作

进阶要求：

把 useState 改成 useReducer，定义 LOGIN、LOGOUT、UPDATE_PROFILE 三种 action。

写完发给我看，或者遇到卡点随时问。
