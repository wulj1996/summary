# Hash 路由实现总结

## 1. 什么是 Hash 路由

Hash 路由是指利用 URL 中 `#` 后面的部分（即 `hash`）作为页面状态的一种前端路由方案。

```
https://example.com/index.html#/home
                       │
                       └── hash 部分："#/home"
```

特点：

- hash 变化不会导致浏览器向服务器发送新请求，属于纯前端行为；
- 页面不会因 hash 变化而整体刷新；
- 可以通过监听 `hashchange` 事件感知 hash 的每一次变化。

## 2. 核心原理

| 要素         | 说明                                                                     |
| ------------ | ------------------------------------------------------------------------ |
| 读取当前状态 | `window.location.hash` 获取当前 hash                                     |
| 修改状态     | 点击 `<a href="#/...">`、`location.hash = '...'`、`history.pushState` 等 |
| 监听变化     | `window.addEventListener('hashchange', handler)`                         |
| 触发时机     | 点击锚点、浏览器前进/后退、手动修改地址均会触发                          |

写入 hash 与监听 `hashchange` 配合，即可在单页内实现"页面切换"，这就是 hash 路由的基本闭环。

## 3. 条件渲染（核心逻辑）

对照表驱动方式渲染：用一个 `routes` 对象把每个 hash 映射到对应的渲染函数，`render()` 根据当前 hash 查表，命中则渲染对应内容，否则渲染空状态。

```js
const routes = {
  "#/home": () => `<h2>首页</h2><p>首页的内容</p>`,
  "#/about": () => `<h2>关于</h2><p>关于的内容</p>`,
  "#/contact": () => `<h2>联系</h2><p>联系的内容</p>`,
};

function render() {
  const hash = window.location.hash || "#/home"; // 默认首页
  const view = routes[hash];

  if (view) {
    // 命中路由：渲染视图
    app.innerHTML = view();
  } else {
    // 未命中：渲染空状态
    app.innerHTML = "没有找到该页面";
  }

  // 同步高亮当前激活的导航链接
  links.forEach((link) =>
    link.classList.toggle("active", link.getAttribute("href") === hash),
  );
}
```

## 4. 监听与初始化

```js
// 监听 hash 变化：点击链接、前进/后退、手改地址都会触发
window.addEventListener("hashchange", render);

// 页面首次加载时手动渲染一次（否则首屏是空的）
render();
```

> 注意：`hashchange` 只在 hash **发生变化**时触发，首次加载并不会产生变化，因此必须手动调用一次 `render()`。

## 5. 完整代码

```html
<!DOCTYPE html>
<html lang="zh-CN">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Hash 路由 + 条件渲染</title>
    <style>
      body {
        font-family: system-ui, sans-serif;
        max-width: 600px;
        margin: 40px auto;
      }
      nav {
        display: flex;
        gap: 8px;
        margin-bottom: 24px;
      }
      nav a {
        padding: 8px 16px;
        border: 1px solid #d1d5db;
        border-radius: 6px;
        text-decoration: none;
        color: #374151;
        background: #f9fafb;
      }
      nav a.active {
        background: #2563eb;
        border-color: #2563eb;
        color: #fff;
      }
      .panel {
        padding: 20px;
        border-radius: 8px;
        border: 1px solid #e5e7eb;
        min-height: 120px;
      }
      .empty {
        color: #9ca3af;
      }
    </style>
  </head>
  <body>
    <nav>
      <a href="#/home">首页</a>
      <a href="#/about">关于</a>
      <a href="#/contact">联系</a>
    </nav>
    <div id="app" class="panel empty">请选择一个页面…</div>

    <script>
      const routes = {
        "#/home": () => `<h2>首页 🏠</h2><p>首页内容</p>`,
        "#/about": () => `<h2>关于我们 📖</h2><p>关于内容</p>`,
        "#/contact": () => `<h2>联系我们 📧</h2><p>联系内容</p>`,
      };

      const app = document.getElementById("app");
      const links = document.querySelectorAll("nav a");

      function render() {
        const hash = window.location.hash || "#/home";
        const view = routes[hash];

        if (view) {
          app.classList.remove("empty");
          app.innerHTML = view();
        } else {
          app.classList.add("empty");
          app.innerHTML = "没有找到该页面，请从上方选择。";
        }

        links.forEach((link) =>
          link.classList.toggle("active", link.getAttribute("href") === hash),
        );
      }

      window.addEventListener("hashchange", render);
      render();
    </script>
  </body>
</html>
```

## 6. 常见问题

### 6.1 为什么首次打开没有内容？

`hashchange` 只在 hash 变化时触发。首次加载没有变化，需手动调用一次 `render()`。

### 6.2 默认路由如何处理？

在 `render()` 中使用 `window.location.hash || '#/home'`，当 hash 为空时回退到默认页面。

### 6.4 与 History 路由的区别

|                    | Hash 路由          | History 路由               |
| ------------------ | ------------------ | -------------------------- |
| URL 格式           | `#/about`          | `/about`                   |
| 是否需要服务器配置 | 否                 | 需要（如 Nginx try_files） |
| 刷新行为           | 不向后端发请求     | 会向后端发请求             |
| 适用场景           | 静态托管、快速原型 | SEO 友好的正式项目         |
