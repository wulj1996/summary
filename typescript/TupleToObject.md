# TupleToObject

## 一、实现目标

自定义一个 `TupleToObject` 类型工具，将一个只读元组（tuple）转换成一个对象类型：元组中的每个元素作为对象的键，且该键的值就是该键本身（通常用于将常量数组转为对象映射）。

## 二、代码实现

```typescript
type TupleToObject<T extends readonly (string | number | symbol)[]> = {
  [key in T[number]]: key
}
```

使用示例：
```typescript
const tuple = ['tesla', 'model 3', 'model X', 'model Y'] as const

type obj = TupleToObject<typeof tuple>
// 得到
// {
//   tesla: "tesla";
//   "model 3": "model 3";
//   "model X": "model X";
//   "model Y": "model Y";
// }
```

## 三、用到的 TypeScript 功能

### 1. `as const`
`as const` 将数组断言为**只读元组**：
- 使数组变为 `readonly`，元素值变为字面量类型
- 保证 `typeof tuple` 能拿到每个元素精确的字面量值，否则只会得到宽泛的 `string`

```typescript
const tuple = ['tesla', 'model 3', 'model X', 'model Y'] as const
// tuple 的类型是 readonly ["tesla", "model 3", "model X", "model Y"]
```

### 2. `typeof`
`typeof tuple` 在类型层面取出变量 `tuple` 的**类型**（只读元组类型），作为泛型实参传入。

### 3. 泛型约束（Generic Constraint）

`T extends readonly (string | number | symbol)[]` 限制 `T` 必须是一个只读数组/元组，其元素类型只能是 `string`、`number` 或 `symbol`（对象键的合法类型），防止传入不匹配的类型。

### 4. 索引访问类型（Indexed Access Type）— `T[number]`

`T[number]` 的含义是：**获取类型 `T` 上所有数字索引（0, 1, 2, ...）对应的值的类型，并将它们合并成一个联合类型（Union Type）**。

```typescript
// T = readonly ["tesla", "model 3", "model X", "model Y"]
// T[number] => "tesla" | "model 3" | "model X" | "model Y"
```

### 5. 映射类型（Mapped Type）

`[key in T[number]]: key` 遍历 `T[number]` 这个联合类型中的每一个值 `key`，把 `key` 作为对象的键，其类型值也设为 `key` 本身（字面量类型）。

### 6. 联合类型（Union Type）
`T[number]` 展开得到的所有元组元素值，以 `|` 连接构成联合类型。
