## Lambda


### Lambda 表达式表示

Lambda 表达式表示结构如下，使用箭头进行分隔，箭头左边是参数列表，右边方法体：

```java
LambdaExpression lambda = (...) -> {...};
```

Lambda 参数列表和普通方法的参数列表表示方式并没有差别，参数间使用逗号分隔，参数名称前是参数类型；方法体也和普通方法的方法体相同，使用花括号表示，包含语句和返回值（如果有的话）：

```java
BiFunction<Integer, Integer, Integer> add = (Integer x, Integer y) -> {  
    System.out.println(x + '+' + y);  
    return x + y;  
};
```

但在 Lambda 表达式中有几个例外：
1. 参数列表中的参数类型并不是必须的，可以省略，编译器可以根据目标类型自动推断，但如果参数个数较多在影响代码可读性的情况下建议加上：

```java
BiFunction<Integer, Integer, Integer> add = (x, y) -> {  
    System.out.println(x + '+' + y);  
    return x + y;  
};
```
2. 当参数只有一个时，参数列表的括号可以省略（无参括号不能省略）：

```java
Consumer<String> consumer = str -> {
	System.out.println(str)
};

// 无参不能省略括号
Supplier<String> supplier = () -> {
	return "hello"
};
```

3. 当方法体里只有一行语句时，方法体花括号可以省略（如果最后一行是 `return` 语句，`return` 也可省略）：

```java
Consumer<String> consumer = str -> System.out.println(str);

Supplier<String> supplier = () -> "hello";
```

以下是各种形式的 Lambda 表达式表示形式：

```java
// 无参无返回值
Runnable runnable = () -> {...};

// 单个参数无返回值
Consumer<String> consumer = (str) -> System.out.println(str);

// 多个参数无返回值
BiConsumer<String, String> biConsumer = (s1, s2) -> {
	System.out.println(s1);
	System.out.println(s2);
}

// 无参有返回值
Supplier<String> supplier = () -> “hello lambda!“;

// 单个参数有返回值
Function<Integer, String> func = i -> {
	System.out.println(i);
	return i + "";
}

// 多个参数有返回值
BiFunction<Integer, Integer, Integer> add = (x, y) -> x + y;
```