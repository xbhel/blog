# Flink 单元测试(二)

测试是每个软件开发过程中都不可或缺的一部分，同样 Apache Flink 也提供了一些工具可以在测试金字塔的多个级别上测试你的应用程序代码。

## 测试 UDF

通常，我们认为 Flink 在 UDF（user-defined function 用户定义的函数）之外产生的是正确结果，因此，建议尽可能用单元测试来测试那些包含主要业务逻辑的类。

### 单元测试 Stateless,Timeless UDFs

让我们举一个无状态的 `MapFunction` 的例子：

```java
public class IncrementMapFunction implements MapFunction<Long, Long> {

    private static final long serialVersionUID = 1L;

    @Override
    public Long map(Long value) throws Exception {
        return value + 1;
    }
}
```

为这个函数编写单元测试是非常简单，使用你喜欢的测试框架，传递合适的输入的参数并验证输出结果。

```java
@Test
void testIncrement() throws Exception {
    // 实例化你的函数
    IncrementMapFunction incrementMapFunction = new IncrementMapFunction();
    // 调用实现的方法
    assertThat(incrementMapFunction.map(2L)).isEqualTo(3L);
}
```

类似地，一个使用 `org.apache.flink.util.Collector`（如：`FlatMapFunction` ,`ProcessFunction`）的 UDF 可以通过提供一个模拟对象来替代真实的 `Collector` 对象去轻松的进行测试。

一个具有和 `IncrementMapFunction` 同样的功能 `FlatMapFunction` 测试案例如下：

```java
@Test
void testIncrement(@Mock Collector<Long> collector) throws Exception {
    // 实例化
    IncrementFlatMapFunction increment = new IncrementFlatMapFunction();
    // 调用实现的方法
    increment.flatMap(2L, collector);
    // 验证使用正确的输出调用了收集器
    Mockito.verify(collector, times(1)).collect(3L);
}
```

### 单元测试 Stateful,Timely UDFs 和自定义算子

对使用了托管状态（state）或定时器（timer）的 UDF 的功能进行测试难度相对大一些，因为它涉及测试用户代码和 Flink 运行时（runtime）之间的交互。为此，Flink 提供了一系列叫做 `test harnesses` 的测试工具，能够帮助我们测试此类用户定义的函数以及自定义算子：

- `OneInputStreamOperatorTestHarness` （用于基于 `DataStreamS` 的算子）
- `KeyedOneInputStreamOperatorTestHarness`（用于基于 `KeyedStreamS` 的算子）
- `TwoInputStreamOperatorTestHarness`（用于基于两个 `DataStreamS` 的 `ConnectedStreamS` 的算子）
- `KeyedTwoInputStreamOperatorTestHarness`（用于基于两个 `KeyedStreamS` 的 `ConnectedStreamS` 的算子）

要使用这些 `test harnesses` 你需要引入额外的依赖，参考 [Flink 单元测试(一)](ut1.md)。

如果需要对 Table API 进行测试，还需要引入 `flink-table-test-utils` 依赖，它是在 Flink 1.15 版本引入的，目前还处于 experimental（实验性）阶段。

```xml
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-table-test-utils</artifactId>
    <version>1.15.0</version>
    <scope>test</scope>
</dependency>
```

现在，你可以使用 `test harnesses` 将输入数据和水印（watermarks）推送到用户定义的函数或自定义算子中，并控制处理时间和对算子的输出进行最终的断言（包括侧边流输出）。

让我们实现一个简单的有状态的 `FlatMapFunction` 算子：

```java
public class StatefulFlatMapFunction extends RichFlatMapFunction<Long, Long> {

    private static final long serialVersionUID = 1L;

    private ValueState<Long> previousOutput;

    @Override
    public void open(Configuration parameters) throws Exception {
        previousOutput = getRuntimeContext().getState(
                new ValueStateDescriptor<>("previousOutput", Types.LONG)
        );
    }

    @Override
    public void flatMap(Long value, Collector<Long> out) throws Exception {
        Long largerValue = value;
        if(previousOutput.value() != null) {
            largerValue = Math.max(previousOutput.value(), value);
        }
        previousOutput.update(largerValue);
        out.collect(largerValue);
    }
}
```

让我们看看如何使用 `test harnesses` 为其编写测试案例：

```java

```



















