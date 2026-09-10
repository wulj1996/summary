# First

## 一、实现目标

自定义一个 `First` 类型工具，取出一个元组（数组）类型中的**第一个**元素的类型；若元组为空，则返回 `never`。

## 二、代码实现

```typescript
type First<T extends any[]> = T extends [infer F, ...infer Reset] ? F : never;
```

使用示例：
```typescript
type arr1 = ['a', 'b', 'c']                   // 取到 "a"
type arr2 = [() => 123, { a: string }]        // 取到 () => 123
type arr3 = []                                // 空元组，取到 never
```

## 三、用到的 TypeScript 功能

### 1. 泛型（Generics）
`type First<T extends any[]>` 定义了一个类型参数 `T`，`T` 代表传入的元组类型。

### 2. 泛型约束（Generic Constraint）— `T extends any[]`
`extends any[]` 限定 `T` 必须是一个数组或元组类型，防止传入非数组类型（如 `string`、`number`）。

### 3. 条件类型（Conditional Type）
`T extends ... ? F : never` 是类型层面的三目运算：
- 若 `T` 能匹配左边的模式，取值为真分支 `F`
- 否则取假分支 `never`

### 4. 类型推断（Type Inference）— `infer`
`infer F` 在模式匹配时**声明并推断**出一个新的类型变量 `F`，用于捕获被匹配的部分。`infer` 只能出现在条件类型的 `extends` 右侧。

### 5. 数组剩余元素（Rest Element）— `...infer Reset`
`...infer Reset` 匹配元组中**除第一个元素以外的所有剩余元素**，类似 JavaScript 中的剩余参数。即使元组只有一个元素，`Reset` 也能匹配为空数组。

### 6. 元组模式匹配
`T extends [infer F, ...infer Reset]` 将 `T` 拆解为"第一个元素 + 剩余元素"，从而捕获首元素。
