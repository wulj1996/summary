# History 路由实现总结

## 1. 核心原理

利用 HTML5 `History API` 无刷新修改 URL，再配合前端条件渲染实现"页面切换"。

| 要素               | API / 方式                                     |
| ------------------ | ---------------------------------------------- |
| 修改 URL（不刷新） | `history.pushState(null, '', '/about')`        |
| 监听 URL 变化      | `popstate` 事件（仅前进/后退触发）             |
| 条件渲染           | 读 `location.pathname`，查路由表，渲染对应视图 |

## 2. 关键逻辑

```js
// ① 点击链接：拦截默认跳转 → pushState 改 URL → 手动渲染
link.addEventListener("click", (e) => {
  e.preventDefault();
  history.pushState(null, "", "/about");
  render();
});

// ② 前进/后退：popstate 时渲染（URL 已由浏览器改好）
window.addEventListener("popstate", render);

// ③ 条件渲染：根据 pathname 查表
function render() {
  const path = location.pathname;
  const view = routes[path] || routes["/home"];
  app.innerHTML = view();
}
```

## 3. 为什么需要服务器回退

刷新 `/about` 时浏览器会真实请求该路径，服务器上没有 `/about` 文件就会 404。解决方式：服务器将所有未知路径都返回同一个 `index.html`，让前端自己根据路径渲染。

```js
// history-router-server.js 的核心
if (真实文件存在) → 返回该文件
else → 一律返回 index.html
```

## 4. 与 Hash 路由对比

|                    | Hash 路由         | History 路由                             |
| ------------------ | ----------------- | ---------------------------------------- |
| URL 格式           | `/#/about`        | `/about`                                 |
| URL 变化是否发请求 | 否                | `pushState` 不发送，刷新会               |
| 需要服务器配置     | 否                | 需要（未知路径回退到 index.html）        |
| 驱动渲染方式       | 监听 `hashchange` | `pushState` 后手动渲染 + `popstate` 监听 |

## 5. 关键细节

- **`pushState 后要手动调 render()`**：浏览器不会因 `pushState` 触发任何事件
- **首次加载要手动调一次 `render()`**：否则页面是空的
- **`popstate` 只在前进/后退触发**，所以两个入口（pushState + popstate）都要处理
- **`preventDefault()` 拦截 `<a>` 的默认跳转**，否则会整页刷新，路由失效

## 6. 完整源码

### history-router-index.html（前端：路由表 + 条件渲染 + 事件绑定）

```html
<!DOCTYPE html>
<html lang="zh-CN">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>History 路由 + 条件渲染</title>
    <style>
      body {
        font-family:
          system-ui,
          -apple-system,
          sans-serif;
        max-width: 600px;
        margin: 40px auto;
        padding: 0 16px;
        color: #1f2937;
      }

      nav {
        display: flex;
        gap: 8px;
        margin-bottom: 24px;
        flex-wrap: wrap;
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
        background: #16a34a;
        border-color: #16a34a;
        color: #fff;
      }

      .panel {
        padding: 20px;
        border-radius: 8px;
        border: 1px solid #e5e7eb;
        background: #ffffff;
        min-height: 120px;
      }

      .panel h2 {
        margin-top: 0;
      }
      .empty {
        color: #9ca3af;
      }

      .btn {
        margin-top: 8px;
        padding: 8px 14px;
        border-radius: 6px;
        border: 1px solid #d1d5db;
        background: #fff;
        cursor: pointer;
      }
    </style>
  </head>
  <body>
    <nav>
      <a href="/home">首页</a>
      <a href="/about">关于</a>
      <a href="/contact">联系</a>
    </nav>

    <div id="app" class="panel empty">请选择一个页面…</div>

    <script>
      const routes = {
        '/home': () => \`
          <h2>首页 🏠</h2>
          <p>这是 History 路由的「首页」。</p>
          <button class="btn" data-go="/about">前往关于页</button>
        \`,
        '/about': () => \`
          <h2>关于我们 📖</h2>
          <p>History 路由修改 URL 不刷新页面，刷新则需要服务器回退。</p>
        \`,
        '/contact': () => \`
          <h2>联系我们 📧</h2>
          <p>可以在这里留言或发送邮件。</p>
        \`,
      };

      const app = document.getElementById('app');
      const links = document.querySelectorAll('nav a');

      function render() {
        const path = location.pathname;
        const view = routes[path];
        app.innerHTML = view();
      }

      document.querySelectorAll('nav a').forEach(link => {
        link.addEventListener('click', (e) => {
          e.preventDefault();
          const path = link.getAttribute('href');
          history.pushState({}, '', path);
          render();
        });
      });

      window.addEventListener('popstate', render);

      render();
    </script>
  </body>
</html>
```

### history-router-server.js（后端：静态文件服务 + SPA 回退）

```js
const http = require('http');
const fs = require('fs');
const path = require('path');

const PORT = 8080;
const ROOT = __dirname;
const INDEX = path.join(ROOT, 'history-router-index.html');

const server = http.createServer((req, res) => {
  let pathname = decodeURIComponent(
    new URL(req.url, \`http://\${req.headers.host}\`).pathname
  );

  if (
    pathname === '/' ||
    pathname === '/home' ||
    pathname === '/about' ||
    pathname === '/contact'
  ) {
    serveFile(INDEX, res);
    return;
  }

  let filePath = path.join(ROOT, pathname);
  if (fs.existsSync(filePath) && fs.statSync(filePath).isFile()) {
    serveFile(filePath, res);
    return;
  }

  serveFile(INDEX, res);
});

function serveFile(filePath, res) {
  fs.readFile(filePath, (err, data) => {
    if (err) {
      res.writeHead(500);
      res.end('Server Error');
      return;
    }
    const ext = path.extname(filePath);
    const contentType = {
      '.html': 'text/html; charset=utf-8',
      '.js': 'application/javascript; charset=utf-8',
      '.css': 'text/css; charset=utf-8',
      '.png': 'image/png',
      '.jpg': 'image/jpeg',
    }[ext] || 'application/octet-stream';

    res.writeHead(200, { 'Content-Type': contentType });
    res.end(data);
  });
}

server.listen(PORT, () => {
  console.log(\`History 路由服务器已启动: http://localhost:\${PORT}/home\`);
});
```

## 7. 启动方式

```bash
node history-router-server.js
# 访问 http://localhost:8080/home
```
