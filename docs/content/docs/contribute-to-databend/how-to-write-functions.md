+++
title = "如何为 Databend 添加新的函数"
description = "Databend 非常重视函数的设计与实现，这篇文章将会介绍如何为 Databend 添加新的函数。"
draft = false
weight = 650
sort_by = "weight"
template = "docs/page.html"

[extra]
toc = true
top = false
giscus = true
+++


## 迁移函数到全新表达式框架

如果你对自定义类型系统或者数据库项目的研发感兴趣，可以看看 Databend 是如何做的。

现在 Databend 正在尝试将一些旧的函数迁移到全新表达式框架中，你愿意来试试看吗？

- [Migrate String functions to new expression framework #6766](https://github.com/datafuselabs/databend/issues/6766)
- [Migrate control-flow functions to new expression framework #6833 ](https://github.com/datafuselabs/databend/issues/6833)

### 背景

近期，Databend 围绕全新表达式框架的设计与实现开展了许多工作，将会带来一些有意思的特性。

- 类型检查
- 类型安全的向下转型
- 使用 Enum 分发列
- 泛型

### 如何迁移

旧的函数位于 `common/functions/src/scalars` ，它们需要被迁移到 `common/functions-v2/src/scalars/` 。

通常情况下，旧函数实现中的核心逻辑是可以复用的，只需要进行少量重写使其符合新的实现方案。

类似地，旧的测试位于 `common/functions/tests/it/scalars/` ，也应该迁移到 `common/functions-v2/tests/it/scalars/` 。

新测试将会使用 `goldenfile` 进行编写，所以可以轻松生成测试用例而无需大量繁重的手写工作。

### 示例

`OCTET_LENGTH` 将会按字节数返回字符串的长度。

仅仅使用 6 行，就可以在 `common/functions-v2/src/scalars/strings.rs` 中实现 `OCTET_LENGTH` 函数。

```rust
registry.register_1_arg::<StringType, NumberType<u64>, _, _>(
    "octet_length",
    FunctionProperty::default(),
    |_| None,
    |val| val.len() as u64,
);
```

由于 `LENGTH` 是 `OCTET_LENGTH` 的同义函数，只需为其添加一个函数别名即可，仅用一行。

```rust
registry.register_aliases("octet_length", &["length"]);
```

接下来，需要写一些测试，来确保函数实现的正确性。编辑 `common/functions-v2/tests/it/scalars/string.rs`。

```rust
fn test_octet_length(file: &mut impl Write) {
    run_ast(file, "octet_length('latin')", &[]);
    run_ast(file, "octet_length(NULL)", &[]);
    run_ast(file, "length(a)", &[(
        "a",
        DataType::String,
        build_string_column(&["latin", "кириллица", "кириллица and latin"]),
    )]);
}
```

将其注册到 `test_string` 函数中：

```rust
#[test]
fn test_string() {
    let mut mint = Mint::new("tests/it/scalars/testdata");
    let file = &mut mint.new_goldenfile("string.txt").unwrap();

    ...
    test_octet_length(file);
    ...
}
```

通过命令行，可以直接生成完整的测试用例，并附加到对应的 `goldenflie` 中：

```bash
REGENERATE_GOLDENFILES=1 cargo test -p common-functions-v2 --test it
```

请使用 `git diff` 检查一下生成的测试是否符合预期，如果一切顺利，`OCTET_LENGTH` 函数的迁移工作就完成了。

## 函数进阶使用

-  注册方法解析:

function 中暴露了多套注册方法, 根据函数接受的参数个数不同, 分为: `register_0_arg`, `register_1_arg` ..

另外, 根据不同的功能需求, 我们提供了不同Level的注册API

|                                     | Auto Vectorization | Access Output Column Builder | Auto Null Passthrough | Auto Downcast | Throw Runtime Error | Varidic | Tuple |
| ----------------------------------- | ------------------ | ---------------------------- | --------------------- | ------------- | ------------------- | ------- | ------- |
| register_n_arg                      | ✔️                 | ❌                           | ✔️                    | ✔️            | ❌                  | ❌      | ❌      |
| register_passthrough_nullable_n_arg | ❌                 | ✔️                           | ✔️                    | ✔️            | ✔️                  | ❌      | ❌      |
| register_n_arg_core                 | ❌                 | ✔️                           | ❌                    | ✔️            | ✔️                  | ❌      | ❌      |
| register_function_factory           | ❌                 | ✔️                           | ❌                    | ❌            | ✔️                  | ✔️      | ✔️      |

-  Domain解析:

Domain是函数的输入的值域经过函数转换后得出的值域, 一些函数计算是符合单调性等特性的, 利用这类特性我们轻量级计算出函数的值域,这对后续的Partition Prune 有很大帮助, 例如: 数据在底层是通过 timestamp 排序的, 在索引层我们会有timestamp列的 Min/Max 索引, 那么对于带 `where to_date(timestamp) > '2020-01-01'` 过滤条件的SQL查询, 根据索引数据可以利用 `Domain` 计算出 `to_date(timestamp)` 列的 Min/Max 索引,从而进入 Prune 逻辑。


- 类型自动转换规则:

1. 数值类型在精度不丢失的情况下能自动向上转型,如` i8 --> i16 --> i32 --> i64`
2. 所有 integer 类型能转为 int64 类型, 如 `u32 --> i64, u64 ---> i64`, 转型过程出现溢出会抛出错误
3. 所有数值类型能转为 float64 类型, 如 `i32 --> f64, u64 --> f64`, 转型过程出现溢出会抛出错误
4. `null`类型能转为`nullable<T>`类型, 如: `null --> nullable<i32>`
5. `T` 类型能转为 `nullable<T>` 类型, 如: `i32 --> nullable<i32>`
6. 嵌套规则: 如果 `T` 能转为 `U` 类型,则 `nullable<T>` 类型能转为 `nullable<U>` 类型, 则 `Array<T>` 类型能转为 `Array<U>` 类型


由于数值类型较多,大部分情况下我们只需要定义 最大类型即可,如 `asin` 只需要定义 `NumberType<f64>` 类型的参数, 接收到其他类型的参数时会自动转型, 例如: `asin(i32)` 会自动转型为 `asin(f64)`; 在少数性能敏感计算,我们会给较小范围的 数值参数定义额外函数重载, 如 `plus`, `minus`, 此时由于自动转换规则的存在,我们必须注意重载的函数必须定义在最大类型的函数之前, 因为函数的查找是按注册顺序进行查找, 只捕获符合条件的第一个函数。

所以:  `i32` 优先 定义于 `i64`, `u64` 优先定义于 `i64`, `i64` 优先定义于 `f64`, `null` 优先定义于 `nullable`。

### Learn More

- [RFC: Formal Type System](https://github.com/datafuselabs/databend/discussions/5438)
- [Tracking issue for new expression framework](https://github.com/datafuselabs/databend/issues/6547)

