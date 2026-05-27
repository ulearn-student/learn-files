# 第十二课：axios —— 请求真实接口

---

## 一、为什么需要请求接口

前面所有课程的数据都是写死在代码里的（mock 数据）。真实项目里，数据来自后端服务器：用户列表、商品信息、订单数据、登录验证……

前端通过 HTTP 请求向后端要数据，后端返回 JSON，前端拿到后渲染到界面。这个过程叫"接口调用"或"API 请求"。

浏览器原生提供 fetch，但企业项目几乎都用 axios，原因下面会讲。

---

## 二、axios vs fetch 对比

| 对比项 | fetch（原生） | axios（第三方库） |
| --- | --- | --- |
| 自动转 JSON | 否，要手动 `.json()` | 是，自动 parse |
| 错误处理 | 4xx/5xx 不会被 catch | 4xx/5xx 自动抛错 |
| 请求/响应拦截器 | 没有 | 有，企业项目必用 |
| 请求超时 | 不支持 | 直接配 `timeout` |
| 取消请求 | 用 AbortController | 内置 `CancelToken` |
| 兼容老浏览器 | IE 不支持 | 支持 |
| 上传进度监听 | 不支持 | 支持 |
| 体积 | 0（内置） | 约 13KB |

企业级项目几乎清一色用 axios。fetch 用得少，但面试可能会问区别，知道上面这张表就够了。

---

## 三、安装

```bash
npm install axios
```

在组件里使用：

```jsx
// 直接 import 即可
import axios from 'axios';
```

---

## 四、最简单的 GET 请求

```jsx
import { useState, useEffect } from 'react';
import axios from 'axios';

function UserList() {
  // 用户列表状态，初始空数组
  const [users, setUsers] = useState([]);

  // useEffect 在组件挂载后执行
  // 空依赖数组保证只请求一次，不会无限循环
  useEffect(() => {
    // axios.get 返回一个 Promise
    // .then 拿到响应后执行
    axios.get('https://jsonplaceholder.typicode.com/users')
      .then(response => {
        // axios 返回的 response 对象包含：
        // response.data       → 后端返回的数据（已自动 parse 成对象）
        // response.status     → HTTP 状态码，如 200
        // response.headers    → 响应头
        // response.config     → 请求配置
        setUsers(response.data);
      })
      .catch(error => {
        // 网络错误、4xx、5xx 都会进 catch
        console.error('请求失败:', error);
      });
  }, []);

  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>{user.name} - {user.email}</li>
      ))}
    </ul>
  );
}
```

注意 useEffect 里不能直接写 async（语法限制）。要在内部定义一个 async 函数再调用，下面用 async/await 重写。

---

## 五、用 async/await 重写（更常用）

```jsx
function UserList() {
  const [users, setUsers] = useState([]);

  useEffect(() => {
    // 在 useEffect 内部定义一个 async 函数
    async function fetchUsers() {
      try {
        // await 等待请求完成，response 是 axios 返回的对象
        const response = await axios.get('https://jsonplaceholder.typicode.com/users');
        // 拿到数据更新 state
        setUsers(response.data);
      } catch (error) {
        // 用 try/catch 捕获错误，比 .catch 更清晰
        console.error('请求失败:', error);
      }
    }

    // 调用 async 函数
    fetchUsers();
  }, []);

  return (
    <ul>
      {users.map(user => <li key={user.id}>{user.name}</li>)}
    </ul>
  );
}
```

为什么不能直接 `useEffect(async () => {...})`？因为 async 函数返回 Promise，而 useEffect 的回调返回值应该是清理函数或 undefined，类型不匹配会引发问题。

---

## 六、完整的 loading / error 状态处理（必须这样写）

真实项目里，光请求数据是不够的。用户体验需要三种状态：加载中、成功、失败。

```jsx
function UserList() {
  const [users, setUsers] = useState([]);
  // 加载中状态，初始 true（一进来就在加载）
  const [loading, setLoading] = useState(true);
  // 错误信息，null 表示没错误
  const [error, setError] = useState(null);

  useEffect(() => {
    async function fetchUsers() {
      try {
        // 开始请求前重置错误
        setError(null);
        // 设置 loading 为 true
        setLoading(true);

        const response = await axios.get('https://jsonplaceholder.typicode.com/users');
        setUsers(response.data);
      } catch (err) {
        // 把错误信息存到 state，界面上可以显示
        setError(err.message || '请求失败');
      } finally {
        // finally 块：无论成功失败都会执行
        // 在这里关掉 loading，避免成功/失败两个分支都写一次
        setLoading(false);
      }
    }

    fetchUsers();
  }, []);

  // 加载中：显示加载提示
  if (loading) return <div>加载中...</div>;

  // 错误：显示错误信息 + 重试按钮
  if (error) return (
    <div>
      <p>出错了：{error}</p>
      {/* 实际项目里这里可以有重试按钮 */}
    </div>
  );

  // 成功：渲染数据
  return (
    <ul>
      {users.map(user => <li key={user.id}>{user.name}</li>)}
    </ul>
  );
}
```

这是企业项目的标准写法。每个接口调用都要考虑这三种状态。

---

## 七、四种 HTTP 方法

GET：获取数据（查）

```jsx
// 简单 GET
const response = await axios.get('/api/users');

// 带查询参数的 GET：URL 变成 /api/users?page=2&size=10
const response = await axios.get('/api/users', {
  params: {
    page: 2,
    size: 10,
    keyword: '李'
  }
});
```

POST：新增数据（增）

```jsx
// 第二个参数是要发送的数据，axios 自动转成 JSON
const response = await axios.post('/api/users', {
  name: '李如杰',
  email: 'lrj@example.com',
  level: 'VIP'
});
```

PUT / PATCH：修改数据（改）

```jsx
// PUT：完整替换资源（传所有字段）
await axios.put('/api/users/123', {
  id: 123,
  name: '李如杰',
  email: 'new@example.com',
  level: 'VIP'
});

// PATCH：部分更新（只传要改的字段）
await axios.patch('/api/users/123', {
  email: 'new@example.com'
});
```

DELETE：删除数据（删）

```jsx
// DELETE 通常不带 body，参数放在 URL 里
await axios.delete('/api/users/123');
```

---

## 八、提交表单（POST 实战）

```jsx
function CreateUser() {
  const [form, setForm] = useState({ name: '', email: '' });
  // 提交中状态，防止重复点击提交按钮
  const [submitting, setSubmitting] = useState(false);

  async function handleSubmit() {
    // 简单校验
    if (!form.name || !form.email) {
      alert('请填写完整');
      return;
    }

    try {
      setSubmitting(true);

      // 发送 POST 请求，把表单数据传给后端
      const response = await axios.post('/api/users', form);

      // 后端通常会返回新创建的用户信息（含 id）
      console.log('创建成功，新用户 ID:', response.data.id);
      alert('创建成功');

      // 重置表单
      setForm({ name: '', email: '' });
    } catch (err) {
      // 区分不同类型的错误
      if (err.response) {
        // 后端返回了错误响应（4xx、5xx）
        // err.response.status 是状态码
        // err.response.data 是后端返回的错误详情
        alert(`提交失败：${err.response.data.message}`);
      } else if (err.request) {
        // 请求发出去了但没收到响应（网络问题）
        alert('网络异常，请检查网络');
      } else {
        // 其他错误（代码问题等）
        alert('未知错误');
      }
    } finally {
      setSubmitting(false);
    }
  }

  return (
    <div>
      <input
        value={form.name}
        onChange={e => setForm({ ...form, name: e.target.value })}
        placeholder="姓名"
      />
      <input
        value={form.email}
        onChange={e => setForm({ ...form, email: e.target.value })}
        placeholder="邮箱"
      />
      {/* 提交中时按钮禁用，防止重复提交 */}
      <button onClick={handleSubmit} disabled={submitting}>
        {submitting ? '提交中...' : '创建'}
      </button>
    </div>
  );
}
```

---

## 九、axios 实例（企业级标配）

直接用 `axios.get` 在小项目可以，但企业项目要求所有请求统一配置：baseURL、超时时间、统一携带 token、统一处理错误。这就要用 axios 实例。

```jsx
// src/utils/request.js

import axios from 'axios';

// 创建一个 axios 实例
// 所有用这个实例发的请求都共享下面的配置
const request = axios.create({
  // 后端接口的根地址，请求时只需写相对路径
  // 比如 request.get('/users') 实际请求 https://api.example.com/users
  baseURL: 'https://api.example.com',

  // 请求超时时间（毫秒），10 秒没响应自动失败
  timeout: 10000,

  // 默认请求头，所有请求都会带上
  headers: {
    'Content-Type': 'application/json'
  }
});

// 导出供其他文件使用
export default request;
```

业务文件里使用：

```jsx
import request from '../utils/request';

// 不需要写完整 URL
const response = await request.get('/users');
const response = await request.post('/users', { name: '李如杰' });
```

---

## 十、请求拦截器（企业必用）

拦截器能在请求发出前 / 响应回来时自动做一些事，避免每个接口重复写。

最常见的用途：自动给每个请求带上登录 token。

```jsx
// src/utils/request.js

import axios from 'axios';

const request = axios.create({
  baseURL: 'https://api.example.com',
  timeout: 10000
});

// =========================
// 请求拦截器：在请求发出前执行
// =========================
request.interceptors.request.use(
  // 第一个函数：请求成功配置时执行
  (config) => {
    // 从 localStorage 读取登录后保存的 token
    const token = localStorage.getItem('token');

    // 如果有 token，自动加到请求头里
    // 后端通过 Authorization 头验证用户身份
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }

    // 必须返回 config，否则请求不会发出去
    return config;
  },
  // 第二个函数：请求配置出错时执行（很少进这里）
  (error) => {
    return Promise.reject(error);
  }
);

// =========================
// 响应拦截器：在响应回来时执行
// =========================
request.interceptors.response.use(
  // 响应成功时执行（HTTP 状态 2xx）
  (response) => {
    // 通常后端会返回固定格式，比如 { code: 200, data: ..., message: '...' }
    // 在这里可以统一处理业务错误（HTTP 成功但业务失败）
    if (response.data.code !== 200) {
      // 弹个错误提示
      alert(response.data.message);
      // 抛出错误，让业务代码的 catch 捕获
      return Promise.reject(new Error(response.data.message));
    }

    // 只返回 data 字段，业务代码不用再写 .data.data
    return response.data.data;
  },
  // 响应失败时执行（HTTP 状态 4xx、5xx）
  (error) => {
    // 根据状态码做不同处理
    if (error.response) {
      switch (error.response.status) {
        case 401:
          // 未登录或 token 过期：清除登录信息，跳到登录页
          localStorage.removeItem('token');
          window.location.href = '/login';
          break;
        case 403:
          alert('没有权限访问');
          break;
        case 404:
          alert('请求的资源不存在');
          break;
        case 500:
          alert('服务器内部错误');
          break;
        default:
          alert(`请求失败：${error.response.status}`);
      }
    } else if (error.code === 'ECONNABORTED') {
      // 请求超时
      alert('请求超时，请重试');
    } else {
      // 网络断开等其他情况
      alert('网络异常');
    }

    return Promise.reject(error);
  }
);

export default request;
```

配好拦截器之后，业务代码极简：

```jsx
// 不需要处理 token、不需要处理 401 跳转、不需要处理 500 提示
async function getUsers() {
  // 直接拿到 data 数据，因为拦截器已经处理过响应了
  const users = await request.get('/users');
  return users;
}
```

---

## 十一、API 接口集中管理（企业级）

不要把请求散落在各个组件里。把所有接口集中定义在一个文件，组件只调函数。

```jsx
// src/api/user.js

import request from '../utils/request';

// 获取用户列表
export function getUserList(params) {
  return request.get('/users', { params });
}

// 获取单个用户
export function getUserById(id) {
  return request.get(`/users/${id}`);
}

// 创建用户
export function createUser(data) {
  return request.post('/users', data);
}

// 更新用户
export function updateUser(id, data) {
  return request.put(`/users/${id}`, data);
}

// 删除用户
export function deleteUser(id) {
  return request.delete(`/users/${id}`);
}

// 登录
export function login(username, password) {
  return request.post('/auth/login', { username, password });
}
```

组件里使用：

```jsx
import { useState, useEffect } from 'react';
import { getUserList, deleteUser } from '../api/user';

function UserPage() {
  const [users, setUsers] = useState([]);

  useEffect(() => {
    async function load() {
      // 接口调用变得像调用普通函数，看不到 HTTP 细节
      const data = await getUserList({ page: 1, size: 20 });
      setUsers(data);
    }
    load();
  }, []);

  async function handleDelete(id) {
    await deleteUser(id);
    // 删除后重新加载列表
    setUsers(prev => prev.filter(u => u.id !== id));
  }

  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>
          {user.name}
          <button onClick={() => handleDelete(user.id)}>删除</button>
        </li>
      ))}
    </ul>
  );
}
```

好处：

接口路径变了只改一处

接口函数可以被多个组件复用

便于后期统一改造（比如换成 GraphQL）

便于 TypeScript 加类型

便于做 Mock 数据

---

## 十二、取消请求（避免内存泄漏）

经典场景：用户在搜索框打字，每个字母都发请求。结果用户搜了"李如杰"，但因为网络延迟，"李"的请求最后才返回，覆盖了正确结果。

或者：组件还没拿到响应，用户已经切走了页面，组件卸载。这时设置 state 会报警告"在已卸载组件上设置 state"。

解决办法：取消之前的请求。

```jsx
import { useEffect, useState } from 'react';
import axios from 'axios';

function SearchBox() {
  const [keyword, setKeyword] = useState('');
  const [results, setResults] = useState([]);

  useEffect(() => {
    // AbortController 是浏览器原生 API
    // 用来取消异步操作
    const controller = new AbortController();

    async function search() {
      // 关键字为空时不搜索
      if (!keyword) {
        setResults([]);
        return;
      }

      try {
        const response = await axios.get('/api/search', {
          params: { q: keyword },
          // 把 signal 传给 axios，外部可以通过 controller.abort() 取消请求
          signal: controller.signal
        });
        setResults(response.data);
      } catch (err) {
        // 被取消的请求会抛 CanceledError
        // 这种错误不需要处理（不是真的错误）
        if (axios.isCancel(err)) {
          console.log('请求被取消');
          return;
        }
        console.error(err);
      }
    }

    search();

    // useEffect 的清理函数：
    // 1. 组件卸载时执行
    // 2. 依赖变化前执行
    // 这里在 keyword 变化前，取消上一次还在进行的请求
    return () => {
      controller.abort();
    };
  }, [keyword]); // keyword 变化时重新搜索

  return (
    <div>
      <input
        value={keyword}
        onChange={e => setKeyword(e.target.value)}
        placeholder="搜索..."
      />
      <ul>
        {results.map(item => <li key={item.id}>{item.name}</li>)}
      </ul>
    </div>
  );
}
```

实际项目里搜索还要加 debounce（防抖），用户停止输入 300ms 后才发请求，减少请求次数。这属于性能优化。

---

## 十三、文件上传

```jsx
function UploadAvatar() {
  const [progress, setProgress] = useState(0);

  async function handleFileChange(e) {
    const file = e.target.files[0];
    if (!file) return;

    // FormData 是浏览器原生 API，用来构造文件上传的数据格式
    const formData = new FormData();
    formData.append('file', file);
    formData.append('userId', '123'); // 也可以加其他字段

    try {
      const response = await axios.post('/api/upload', formData, {
        // 文件上传要改 Content-Type
        headers: {
          'Content-Type': 'multipart/form-data'
        },
        // 上传进度回调，axios 独有的功能（fetch 不支持）
        onUploadProgress: (progressEvent) => {
          // progressEvent.loaded：已上传字节数
          // progressEvent.total：总字节数
          const percent = Math.round((progressEvent.loaded * 100) / progressEvent.total);
          setProgress(percent);
        }
      });

      console.log('上传成功，文件 URL:', response.data.url);
    } catch (err) {
      console.error('上传失败', err);
    }
  }

  return (
    <div>
      <input type="file" onChange={handleFileChange} />
      {progress > 0 && <p>上传进度：{progress}%</p>}
    </div>
  );
}
```

---

## 十四、并发请求

需要同时发多个请求，等所有都完成再处理：

```jsx
async function loadDashboard() {
  try {
    // Promise.all 接收一个 Promise 数组
    // 全部成功才进 then，任何一个失败立即进 catch
    const [userInfo, orders, statistics] = await Promise.all([
      axios.get('/api/user/info'),
      axios.get('/api/orders'),
      axios.get('/api/statistics')
    ]);

    // 三个数据都拿到了，可以一起处理
    console.log(userInfo.data, orders.data, statistics.data);
  } catch (err) {
    console.error('有请求失败', err);
  }
}
```

如果想让某个请求失败不影响其他请求，用 `Promise.allSettled`：

```jsx
const results = await Promise.allSettled([
  axios.get('/api/a'),
  axios.get('/api/b'),
  axios.get('/api/c')
]);

// results 是数组，每项是 { status, value } 或 { status, reason }
results.forEach((result, index) => {
  if (result.status === 'fulfilled') {
    console.log(`第 ${index} 个成功:`, result.value.data);
  } else {
    console.log(`第 ${index} 个失败:`, result.reason);
  }
});
```

---

## 十五、自定义 Hook 封装请求（最佳实践）

每个接口都写 loading / error / data，代码重复严重。封装成自定义 Hook：

```jsx
// src/hooks/useRequest.js

import { useState, useEffect } from 'react';

// 接收一个返回 Promise 的函数（通常是 API 调用）
export function useRequest(apiFn, deps = []) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false; // 防止组件卸载后还 set state

    async function load() {
      try {
        setLoading(true);
        setError(null);
        const result = await apiFn();

        // 如果组件已卸载，不更新 state
        if (!cancelled) {
          setData(result);
        }
      } catch (err) {
        if (!cancelled) {
          setError(err);
        }
      } finally {
        if (!cancelled) {
          setLoading(false);
        }
      }
    }

    load();

    // 清理函数：组件卸载或依赖变化时执行
    return () => {
      cancelled = true;
    };
  }, deps);

  return { data, loading, error };
}
```

组件里使用，一行搞定：

```jsx
import { useRequest } from '../hooks/useRequest';
import { getUserList } from '../api/user';

function UserList() {
  // 调用自定义 Hook，自动处理 loading / error / data
  const { data: users, loading, error } = useRequest(() => getUserList());

  if (loading) return <div>加载中...</div>;
  if (error) return <div>出错了：{error.message}</div>;

  return (
    <ul>
      {users.map(user => <li key={user.id}>{user.name}</li>)}
    </ul>
  );
}
```

第十一课会讲自定义 Hook 的完整设计原则。

---

## 十六、企业级方案对比

| 方案 | 定位 | 优缺点 |
| --- | --- | --- |
| 原生 fetch | 浏览器内置 | 简单但功能少 |
| axios | HTTP 请求库 | 最主流，企业必备 |
| axios + 自定义 Hook | 自己封装 | 灵活但要维护 |
| React Query (TanStack Query) | 服务端状态管理 | 自动缓存、去重、刷新，新项目首选 |
| SWR | Vercel 出品 | 轻量、API 简洁 |
| Apollo Client | GraphQL 专用 | 用 GraphQL 时才用 |
| RTK Query | Redux Toolkit 内置 | 用 Redux 时用 |

实际经验：

学习阶段 → axios + 自定义 Hook（理解原理）

中小型新项目 → React Query 或 SWR（生产力高）

大型企业项目 → axios 实例 + 接口集中管理 + React Query

老项目维护 → 跟现有方案

React Query 解决了一个 axios 不解决的问题：服务端数据缓存和同步。比如多个组件请求同一个用户信息，React Query 只发一次请求，多个组件共享结果。这种东西自己写要写很多代码，用现成的库省事。

---

## 十七、常见错误和坑

错误 1：在 useEffect 外层加 async

```jsx
// 错误
useEffect(async () => { ... }, []);

// 正确
useEffect(() => {
  async function load() { ... }
  load();
}, []);
```

错误 2：忘记加依赖数组

```jsx
// 错误：每次渲染都请求，无限循环
useEffect(() => {
  axios.get('/users').then(...);
});

// 正确：空数组只请求一次
useEffect(() => {
  axios.get('/users').then(...);
}, []);
```

错误 3：在 catch 里没处理错误

```jsx
// 不好：错误被吞了
try {
  await axios.get('/users');
} catch (err) {}

// 好：至少打日志或显示提示
try {
  await axios.get('/users');
} catch (err) {
  console.error(err);
  setError(err.message);
}
```

错误 4：直接修改 axios 默认配置污染全局

```jsx
// 不好：所有用 axios 的地方都受影响
axios.defaults.baseURL = 'https://api.example.com';

// 好：用实例，互不影响
const request = axios.create({ baseURL: 'https://api.example.com' });
```

错误 5：在组件卸载后还更新 state

```jsx
// 看控制台会有警告：Can't perform a React state update on an unmounted component
// 解决：用 AbortController 取消请求，或用 cancelled 标志位
```

---

## 十八、知识体系总结

axios 核心 API：

axios.get / post / put / delete / patch 五种请求方法

axios.create 创建实例，统一配置

axios.interceptors 请求和响应拦截器

axios.isCancel 判断是否是取消错误

企业级架构：

封装 axios 实例（utils/request.js）

接口集中管理（api/xxx.js）

请求拦截器加 token

响应拦截器统一处理错误

自定义 Hook 封装通用逻辑

React 配合：

useEffect 里发请求

loading / error / data 三件套

useEffect 清理函数处理取消请求

并发用 Promise.all / allSettled

避免的坑：

useEffect 外层不能直接 async

依赖数组不能忘

组件卸载后不要 set state

不要污染全局 axios 默认配置

---

## 十九、作业

实现一个完整的用户管理页面：

1. 用 jsonplaceholder.typicode.com 这个免费测试 API，地址是 https://jsonplaceholder.typicode.com/users
2. 进入页面时请求用户列表，展示加载中状态
3. 加载完成后渲染列表，每个用户显示 name、email、phone
4. 列表上方加一个搜索框，输入关键字按 name 过滤（前端过滤，不调接口）
5. 每个用户后面加"删除"按钮，点击调用 DELETE 请求（虽然这个 API 不会真删，但会返回 200）
6. 删除成功后从列表里移除该用户
7. 出错时显示错误提示

进阶要求：

把 axios 调用封装到 src/utils/request.js 里，加 baseURL 和拦截器

把所有接口集中放到 src/api/user.js

把数据请求逻辑提取成 useRequest 自定义 Hook

写完发给我看，或者遇到卡点随时问。
