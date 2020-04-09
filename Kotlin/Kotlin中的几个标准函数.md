# Kotlin中的几个标准函数

Kotlin中的标准函数是指**Standard.kt**文件中定义的函数，可以大大简化我们的开发。

## 1.let函数

```kotlin
public inline fun <T, R> T.let(block: (T) -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block(this)
}
```

用法：

```kotlin
obj.let {
    it.fun1()
    it.fun2() // lambda表达式的最后一行作为返回值
}
// 上面的代码等价于
obj.let { obj2 ->
    it.fun1()
    it.fun2()
}
```

let函数多用于配合**?.**操作符进行判空处理。

## 2.also函数

```kotlin
public inline fun <T> T.also(block: (T) -> Unit): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    block(this)
    return this
}
```

also函数和let函数类似，区别是返回值为调用的对象本身。

```kotlin
obj.also {
    it.fun1()
    it.fun2() // 返回值为obj对象
}
```

## 3.with函数

```kotlin
public inline fun <T, R> with(receiver: T, block: T.() -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return receiver.block()
}
```

with函数的第一个参数传入一个对象，返回值为lambda表达式最后一行。

```kotlin
with(obj) {
    fun1()
    fun2() // lambda表达式的最后一行作为返回值
}
```

## 4.run函数

```kotlin
public inline fun <R> run(block: () -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block()
}
```

run函数和with函数类似，返回值都为lambda表达式最后一行，区别是参数不需要传入对象。

```kotlin
obj.run() {
    fun1()
    fun2() // lambda表达式的最后一行作为返回值
}
```

## 5.apply函数

```kotlin
public inline fun <T> T.apply(block: T.() -> Unit): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    block()
    return this
}
```

与with函数和run函数不同，apply函数的返回值为调用对象本身，多用于对象的属性设置。

```kotlin
obj.apply() {
    fun1()
    fun2() // 返回值为obj对象
}
```

## 总结

|  函数名  |       代码块内部访问对象       |   返回值   |
| :---: | :-------------------: | :-----: |
|  let  |         使用it          | 代码块最后一行 |
| also  |         使用it          | 调用对象本身  |
| with  | 对象通过参数传入，代码块内可以直接访问对象 | 代码块最后一行 |
|  run  |        直接访问对象         | 代码块最后一行 |
| apply |        直接访问对象         | 调用对象本身  |
