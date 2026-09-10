# Length

## 一、实现目标

自定义一个 `Length` 类型工具，获取一个元组（数组）类型中元素的**数量**（即元组的长度）。

## 二、代码实现

```typescript
type Length<T extends readonly any[]> = T['length'];

type l = Length<typeof tesla>
```

使用示例：
```typescript
const tesla = ['tesla', 'model 3', 'model X', 'model Y'] as const

// Length<typeof tesla> 得到 4
```

## 三、用到的 TypeScript 功能

### 1. `as const`
`as const` 将数组断言为**只读元组**，保留每个元素的字面量类型，防止类型被扩大为宽泛的 `string`。

```typescript
const tesla = ['tesla', 'model 3', 'model X', 'model Y'] as const
// tesla 的类型是 readonly ["tesla", "model 3", "model X", "model Y"]
```
若不加 `as const`，数组会退化为 `string[]`，此时元组元素的精确类型会丢失。

### 2. `typeof`
`typeof tesla` 在类型层面取出变量 `tesla` 的**类型**（只读元组类型），作为泛型实参传给 `Length`。

### 3. 泛型（Generics）
`type Length<T extends readonly any[]>` 定义了一个类型参数 `T`，`T` 代表传入的元组类型。

### 4. 泛型约束（Generic Constraint）
`T extends readonly any[]` 限制 `T` 必须是**只读数组或只读元组**，防止传入无法获取 `length` 的类型（如 `string`、`number`、`object`）。

### 5. `readonly` 关键字
`readonly` 修饰数组，表示这是一个只读元组。`as const` 产生的数组是只读的，因此约束中也需要 `readonly` 才能匹配。

### 6. 索引访问类型（Indexed Access Type）— `T['length']`
`T['length']` 通过索引访问元组类型上的 **`length` 属性**。元组类型自带一个字面量类型的 `length` 属性：
```typescript
T = readonly ["tesla", "model 3", "model X", "model Y"]
T['length']  // => 4（字面量类型）
```
正因为 `as const` 保留了元组的精确信息，`T['length']` 才能得到具体的字面量数字 `4`，而不是宽泛的 `number`。