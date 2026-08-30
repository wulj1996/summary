# 并发控制（Concurrency）实现总结

## 1. 需求与语义

对一组异步任务 `list`，以 `limit` 的并发上限调度执行，所有任务完成后返回结果数组。

实现采用 **Promise.allSettled 风格语义**：每个任务无论成功失败都会被收集，错误不中断整体流程，最终统一 resolve 全部结果（成功为值、失败为错误对象）。

## 2. 核心实现（test.js）

```js
function concurrency(limit, list) {
  if (list.length === 0 || limit <= 0) return Promise.resolve([]);
  return new Promise((resolve) => {
    let i = 0;
    let setCount = 0;
    const results = [];
    async function run() {
      const index = i;
      if (i >= list.length) return;
      i++;
      try {
        const result = await list[index]();
        results[index] = result;
        setCount++;
      } catch (error) {
        results[index] = error;
        setCount++;
      } finally {
        if (setCount === list.length) resolve(results);
        else run();
      }
    }
    for (let k = 0; k < Math.min(limit, list.length); k++) run();
  });
}
```

### 关键设计

| 要素            | 说明                                                                                                 |
| --------------- | ---------------------------------------------------------------------------------------------------- |
| `i`（任务游标） | 每轮 `run()` 同步取下标并 `i++`，**在 `await` 之前**完成，保证并发抢占时下标唯一、不会重复取任务     |
| `run()` 自调度  | 任务完成（finally）后若未满 `list.length`，递归启动下一个任务 → 经典**线程池/滑窗**模式，空闲即补位  |
| 并发上限        | 初始只启动 `Math.min(limit, list.length)` 个 `run()`，之后"完成一个补一个"，任意时刻在跑任务 ≤ limit |
| 结果保序        | `results[index] = ...` 按下标写入，即使任务乱序完成，最终数组顺序与 `list` 一致                      |

### 正确性分析

- **无重复执行**：`i++` 同步发生在 `await` 挂起点之前，单线程下两个 `run()` 不会拿到同一下标。
- **不会重复 resolve**：`setCount++` 与 `setCount === list.length` 判断处于同一同步块，只有"最后一个完成的任务"能命中，其余调用方被 JS 忽略（Promise resolve 幂等）。
- **错误不阻塞调度**：catch 捕获错误、finally 仍执行 `run()`，池子不会因单个任务失败而停摆。
- **同步抛错也安全**：`list[index]()` 若同步 throw，同样被 try/catch 捕获。

## 3. 边界问题与修复

原实现（未加首行判断）存在**挂死 bug**：

| 场景         | 现象                 | 原因                                               |
| ------------ | -------------------- | -------------------------------------------------- |
| `list` 为空  | Promise 永不 resolve | 初始循环启动 0 个 `run()`，`setCount` 永远达不到 0 |
| `limit <= 0` | 同上                 | 同样无任务启动                                     |

修复：入口处兜底 `if (list.length === 0 || limit <= 0) return Promise.resolve([]);`
