# JavaScript new 的原理与手写实现

## 一、new 的四个步骤

`new Foo(...args)` 的执行过程分四步：

1. **创建一个全新的空对象** `obj`。
2. **将 `obj` 的原型指向构造函数的 `prototype`**（建立原型链）。
3. **以 `obj` 作为 `this` 调用构造函数**，初始化实例属性。
4. **返回结果**：若构造函数显式返回了**对象或函数**，则返回该结果；否则（返回 `null`、基本类型或无 return）返回 `obj`。

---

## 二、手写实现

```js
function myNew(constructor, ...args) {
  // 步骤 1 + 2：创建空对象，并将其原型指向 constructor.prototype
  const obj = Object.create(constructor.prototype);

  // 步骤 3：以 obj 为 this 调用构造函数（apply 第二参传数组）
  const result = constructor.apply(obj, args);

  // 步骤 4：构造函数返回了对象或函数则用之，否则返回 obj
  // 注：typeof null === 'object'，需先排除 null
  if (
    result !== null &&
    (typeof result === "object" || typeof result === "function")
  ) {
    return result;
  }
  return obj;
}
```

### 使用示例

```js
function Person(name) {
  this.name = name;
}
Person.prototype.greet = function () {
  return `Hi, ${this.name}`;
};

const p = myNew(Person, "张三");
console.log(p.name); // 张三
console.log(p.greet()); // Hi, 张三
console.log(p instanceof Person); // true
```

---

## 三、总结

| 步骤           | 作用                                |
| -------------- | ----------------------------------- |
| 创建空对象     | 新实例的载体                        |
| 指向原型       | 继承 constructor.prototype 的方法   |
| 绑定 this 调用 | 初始化实例属性                      |
| 返回值判断     | 返回对象/函数时用之，否则返回新对象 |
