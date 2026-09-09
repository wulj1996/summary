# Pick

## 一、实现目标

自定义一个 `MyPick` 类型工具，从一个对象类型中挑选出指定的属性，构造一个新的对象类型。

## 二、代码实现

```typescript
type MyPick<T, K extends keyof T> = {
  [key in K]: T[key]
}

interface Todo {
  title: string
  description: string
  completed: boolean
}

type TodoPreview = MyPick<Todo, 'title' | 'completed'>

const todo: TodoPreview = {
    title: 'Clean room',
    completed: false,
}
```

## 三、用到的 TypeScript 功能

### 1. 泛型（Generics）
`type MyPick<T, K>` 定义了两个类型参数 `T` 和 `K`，使类型可以被复用：
- `T`：原始对象类型
- `K`：要挑选的属性名（联合类型）

### 2. 泛型约束（Generic Constraint）— `K extends keyof T`
`extends` 关键字限制 `K` 的取值范围，必须是 `T` 的属性名之一，防止传入不存在的键。

### 3. `keyof` 操作符
`keyof T` 返回 `T` 所有属性名组成的**联合类型**。例如：
```typescript
keyof Todo  // => 'title' | 'description' | 'completed'
```

### 4. 映射类型（Mapped Type）
`[key in K]` 遍历联合类型 `K` 中的每一个键，逐个生成新的属性。

### 5. 索引访问类型（Indexed Access Type）
`T[key]` 取出 `T` 中 `key` 对应的属性的类型。

## 四、使用示例
