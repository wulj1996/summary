# Generator + co 自动执行器实现分析

## 1. 目标

用 Generator + 自动执行器（co 原理）模拟 `async/await`：`yield` 一个 Promise，执行器把它 `then` 起来，拿到结果再 `next` 回去继续走；若 Promise 被 reject，把错误抛回生成器内部。

## 2. 迭代一：`function*` 误用（test.js）

```js
function* generator(gg) {
  const g = gg();

  function runner(param) {
    const { value, done } = g.next(param);
    if (!done) {
      return Promise.resolve(value).then((v) => runner(v));
    }
  }

  return runner();
}

const g = generator();
g.next();
g.next();
```

### 问题

| # | 问题 | 说明 |
|---|------|------|
| 1 | `generator` 声明成 `function*` | 调用返回**生成器对象**而非 Promise，函数体要等 `.next()` 才执行；`generator()` 未传参导致体内 `gg()` 抛 TypeError |
| 2 | `done` 时丢返回值 | 返回 `undefined`，生成器的最终 `return` 值丢失 |
| 3 | 无错误处理 | `g.next()` 同步 throw 会中断；rejection 无法传回生成器 |

> 结论：自动执行器必须是普通函数，`function*` 只用于"被驱动的生成器"本身。

## 3. 迭代二：改为普通函数（仍有缺陷）

```js
function generator(gg) {
  const g = gg();

  function runner(param) {
    const { value, done } = g.next(param);
    if (!done) {
      return Promise.resolve(value).then((v) => runner(v));
    }
  }

  return runner();
}
```

### 实测结果

```js
function* happy() {
  const a = yield Promise.resolve(1);
  const b = yield 2;
  return a + b;                      // 期望 3
}
generator(happy).then(v => console.log(v));   // → undefined

function* withCatch() {
  try {
    const b = yield Promise.reject(new Error("boom"));
    return "ok: " + b;
  } catch (e) {
    return "caught: " + e.message;   // 期望进入这里
  }
}
generator(withCatch).then(v => console.log(v), e => console.log("rejected:", e.message));
                                      // → rejected: boom（try/catch 未生效）
```

### 仍存在的问题

| # | 问题 | 后果 |
|---|------|------|
| 1 | `done` 时返回 `undefined` | happy path 能跑通，但最终结果被丢弃，resolve 为 `undefined`（上例应为 3） |
| 2 | 无 rejection 分支、无 `g.throw(e)` | 被 `yield` 的 Promise reject 时，错误顺着外层链直接抛给调用方，**生成器内部的 `try/catch` 永远捕获不到**——与 `await`（rejection 转 throw）语义完全不同 |
| 3 | 链断后生成器永久挂起 | rejection 发生后 `.then` 不再补位 `runner`，后续 `yield` 永不执行 |

> 关键：`await` 把 rejection 转成 throw 的本质，就是自动执行器在 reject 时调用 `g.throw(e)`，把错误抛进生成器暂停的位置。

## 4. 修正版（完整）

```js
function generator(gg) {
  const g = gg();

  function runner(param) {
    let r;
    try {
      r = g.next(param);
    } catch (e) {
      return Promise.reject(e);
    }
    if (r.done) return Promise.resolve(r.value);
    return Promise.resolve(r.value).then(
      (v) => runner(v),
      (e) => {
        try {
          return runner(g.throw(e));   // 把 rejection 抛回生成器，内部 try/catch 可捕获
        } catch (err) {
          return Promise.reject(err);
        }
      }
    );
  }

  return runner();
}

function* work() {
  const a = yield Promise.resolve(1);
  try {
    const b = yield Promise.reject(new Error("x"));
    return a + b;
  } catch (e) {
    return e.message;   // 应走到这里
  }
}

generator(work).then((v) => console.log("result:", v));
```

## 5. 要点总结

- **角色分工**：`function*` 生成器 = 任务流程（yield 暂停/恢复）；普通函数 = 自动执行器（co）。
- **驱动协议**：执行器每轮 `g.next(param)` 把上一个 Promise 的结果喂回去；`done === true` 时以 `r.value` resolve 收尾。
- **错误协议**：yield 的 Promise reject → 执行器调 `g.throw(e)` → 生成器内该 `yield` 位置抛错 → 被生成器里的 try/catch 捕获，这正是 `await` 抛错的机制。
- **非 Promise 值**：`Promise.resolve(value)` 统一包装，数字/普通值直接透传，无需特殊处理。
- **迭代一 vs 迭代二**：普通函数修复了"返回生成器对象"的致命问题，但若不加 `g.throw`，错误语义仍不完整（try/catch 失效），不算正确实现。
