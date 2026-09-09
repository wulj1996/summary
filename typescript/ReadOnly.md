# Readonly

## 一、实现目标

自定义一个 `MyReadOnly` 类型工具，将一个对象类型中的所有属性都标记为只读（`readonly`）。这是 TypeScript 类型体操中的一道入门题，与官方内置的 `Readonly` 对应。

## 二、代码实现

```typescript
type MyReadOnly<T> = {
  readonly [key in keyof T]: T[key]
}

interface Todo {
    title: string
    description: string
}

const todo: MyReadOnly<Todo> = {
    title: "Hey",
    description: "foobar"
}
```

## 三、用到的 TypeScript 功能

### 1. 泛型（Generics）
`type MyReadOnly<T>` 定义了一个类型参数 `T`，`T` 代表要被转换的原始对象类型，使类型可以被复用。

### 2. `keyof` 操作符
`keyof T` 返回 `T` 所有属性名组成的**联合类型**。例如：
```typescript
keyof Todo  // => 'title' | 'description'
```

### 3. 映射类型（Mapped Type）
`[key in keyof T]` 遍历 `T` 的所有属性名，逐个重新生成属性，从而对每个属性应用统一的变换。

### 4. 只读操作符 `readonly`
`readonly` 将每个属性标记为只读，一旦赋值后便不可再修改，为属性提供只读保护。

### 5. 索引访问类型（Indexed Access Type）
`T[key]` 取出 `T` 中 `key` 对应的属性的原始类型，保证转换前后属性类型不改变。
