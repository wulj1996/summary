# JavaScript 防抖（debounce）原理与实现

## 一、什么是防抖

**防抖（debounce）**：事件被频繁触发时，**只在最后一次触发后的 `time` 毫秒内没有再次触发，才执行函数**。用于"等待用户停下来再进行操作"。

典型场景：

- 搜索框输入（停止输入后再请求）
- 窗口 resize（停止调整后再处理）
- 按钮提交避免连点

---

## 二、核心思路

用 `timer` 记录一次 `setTimeout`：

- 每次触发都 **先清除上一个定时器**，再重新计时；
- 只要在 `time` 内再次触发，就不断重置；
- 直到停止触发 `time` 毫秒，回调才真正执行。

---

## 三、基础版（尾部防抖）

```js
function debounce(func, time) {
  let timer;
  return function (...args) {
    const that = this; // 同步捕获 this，避免丢失
    clearTimeout(timer); // 先清除上一个定时器
    timer = setTimeout(() => {
      func.apply(that, args); // 停止触发 time 后才执行
    }, time);
  };
}
```

特点：**首调延迟 `time` 才执行**，只有最后一次触发会生效。

---
