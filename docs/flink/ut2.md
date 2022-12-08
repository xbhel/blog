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

类似地，一个使用 `org.apache.flink.util.Collector`（如：`FlatMapFunction` ，`ProcessFunction`）的 UDF 可以通过提供一个模拟对象来替代真实的 `Collector` 对象去轻松的进行测试。

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

- `OneInputStreamOperatorTestHarness` （用于基于 `DataStream` 的算子）
- `KeyedOneInputStreamOperatorTestHarness`（用于基于 `KeyedStream` 的算子）
- `TwoInputStreamOperatorTestHarness`（用于基于两个 `DataStream` 的 `ConnectedStream` 的算子）
- `KeyedTwoInputStreamOperatorTestHarness`（用于基于两个 `KeyedStream` 的 `ConnectedStream` 的算子）

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

现在，你可以使用 `test harnesses` 将输入数据和水印（watermarks）推送到用户定义的函数（UDF）或自定义算子中，并控制处理时间和对算子的输出进行最终的断言（包括侧边流输出）。

让我们实现一个简单的有状态的 `FlatMapFunction` 算子：

```java
public class StatefulFlatMapFunction implements CheckpointedFunction, FlatMapFunction<Long, Long> {  
  
    private static final long serialVersionUID = 1L;  
    private static final ListStateDescriptor<Long> MAX_VALUE_STATE_DESC =  
            new ListStateDescriptor<>("maxValueState", Types.LONG);  
  
    private transient ListState<Long> maxValueState;  
  
    @Override  
    public void flatMap(Long value, Collector<Long> out) throws Exception {  
        Long maxValue = value;  
        Iterable<Long> longIterable = maxValueState.get();  
        if (longIterable != null) {  
            for (Long preMaxValue: longIterable) {  
                maxValue = Math.max(preMaxValue, maxValue);  
            }  
        }  
        maxValueState.update(Lists.newArrayList(maxValue));  
        out.collect(maxValue);  
    }  
  
    @Override  
    public void snapshotState(FunctionSnapshotContext context) throws Exception {  
        // do nothing  
    }  
  
    @Override  
    public void initializeState(FunctionInitializationContext context) throws Exception {  
        maxValueState = context.getOperatorStateStore().getListState(MAX_VALUE_STATE_DESC);  
    }  
}
```

让我们看看如何使用 `test harnesses` 为其编写测试案例：

```java
class StatefulFlatMapFunctionTest {  
    
    private OneInputStreamOperatorTestHarness<Long, Long> testHarness;  
  
    @BeforeEach  
    void setupTestHarness() throws Exception {  
        // 实例化 UDF        
        StatefulFlatMapFunction statefulFlatMapFunction = new StatefulFlatMapFunction();  
        // 将 UDF 包装到相应的 TestHarness 中  
        testHarness = new OneInputStreamOperatorTestHarness<>(new StreamFlatMap<>(statefulFlatMapFunction));  
        // 可选地配置执行环境  
        testHarness.getExecutionConfig().setAutoWatermarkInterval(50);  
        // 打开 testHarness(也会调用 RichFunctions 的 open 方法)  
        testHarness.open();  
    }  
  
    @Test
	void testingStatefulFlatMapFunction() throws Exception {  
	    // 推送元素及与该元素关联的时间戳至算子中 
	    testHarness.processElement(2L, 100L);   
	    // 通过使用水印推进算子的事件时间来触发事件时间 timer 
	    testHarness.processWatermark(100L);  
	    // 通过直接提前算子的处理时间来触发处理时间 timer 
	    testHarness.setProcessingTime(100L); 
        
	    // 获取输出列表用于断言（包含 watermark）  
	    // 在 flink 中水印（watermark）就是通过推送一条记录实现的，这条记录只有时间戳。  
	    assertThat(testHarness.getOutput().toArray())  
            .isEqualTo(Arrays.array(  
                    new StreamRecord<>(2L, 100L),  
                    new Watermark(100L)  
            ));  
        
        // 推送元素及与该元素关联的时间戳至算子中 
        testHarness.processElement(6L, 110);
        
        // 获取输出列表用于断言，直接获取值 
        assertThat(testHarness.extractOutputValues())  
            .isEqualTo(Lists.newArrayList(2L, 6L));  
        
        // 获取 operator 状态用于断言
        ListState<Long> maxValueState = testHarness.getOperator()
                .getOperatorStateBackend()
                .getListState(new ListStateDescriptor<>("maxValueState", Types.LONG));
        assertThat(maxValueState.get()).isEqualTo(Lists.newArrayList(6L));
            
        // 获取侧边流的输出用于断言（仅在 ProcessFunction 可用）
        //assertThat(testHarness.getSideOutput(new OutputTag<>("invalidRecords")), hasSize(0))  
	}    
}
```

接下来我们看一下如何为同样逻辑的 `KeyedStream` 算子进行测试。

对于 `KeyedStream` 算子的测试，我们可以通过 `KeyedOneInputStreamOperatorTestHarness` 和 `KeyedTwoInputStreamOperatorTestHarness`，对此我们需要额外提供 `KeySelector` 和 `Key` 的类型去创建它们。如下：

```java
class KeyedStatefulFlatMapFunctionTest {

    private OneInputStreamOperatorTestHarness<Long, Long> testHarness;
    private KeyedStatefulFlatMapFunction keyedStatefulFlatMapFunction;

    @BeforeEach
    void setupTestHarness() throws Exception {
        // 实例化 UDF
        keyedStatefulFlatMapFunction = new KeyedStatefulFlatMapFunction();
        // 将 UDF 包装到相应的 TestHarness 中
        // 除了传递 UDF 实例，还需要 KeySelector, Key 的类型
        testHarness = new KeyedOneInputStreamOperatorTestHarness<>(
                new StreamFlatMap<>(keyedStatefulFlatMapFunction), (el) -> "1", Types.STRING);
        // 打开 testHarness(也会调用 RichFunctions 的 open 方法)
        testHarness.open();
    }

    @Test
    void testingStatefulFlatMapFunction() throws Exception {
        // 推送元素及与该元素关联的时间戳至算子中
        testHarness.processElement(2L, 100L);
        
        // 获取输出列表用于断言
        assertThat(testHarness.extractOutputValues())
                .isEqualTo(Lists.newArrayList(2L));
        
		// 推送元素及与该元素关联的时间戳至算子中 
        testHarness.processElement(6L, 110L);
        
        // 获取 keyed 状态用于断言，keyed 状态我们可以直接通过算子的 getRuntimeContext 获取
        // 当然也可以使用 testHarness 获取
        ListState<Long> maxValueState = keyedStatefulFlatMapFunction.getRuntimeContext()
                .getListState(new ListStateDescriptor<>("maxValueState", Types.LONG));
        assertThat(maxValueState.get()).isEqualTo(Lists.newArrayList(6L));
    }
}
```

更多关于 `test harnesses` 的使用案例可以在 [Flink 代码仓库](https://github.com/apache/flink/tree/master/flink-streaming-java/src/test/java/org/apache/flink/streaming/runtime/operators/windowing)找到：

- `org.apache.flink.streaming.runtime.operators.windowing.WindowOperatorTest` 是一个测试基于处理或事件时间的算子和用户定义函数（UDF）的很好例子。

>[!note]
>请注意，`AbstractStreamOperatorTestHarness` 及其派生类目前不是公共 API 的一部分，可能会发生变化.。

### 单元测试 ProcessFunction

鉴于它的重要性，除了能够直接使用前面提到的 `test harnesses` 对 `ProcessFunction` 进行测试之外，Flink 还提供了一个名为 `ProcessFunctionTestHarnesses` 的测试工具工厂类，以便简化测试工具（`test harness`）的实例化，让我们看一个例子：

```java
public class PassThroughProcessFunction extends ProcessFunction<Integer, Integer> {  
  
    private static final long serialVersionUID = 1L;  
  
    @Override  
    public void processElement(Integer value,  
                               ProcessFunction<Integer, Integer>.Context context,  
                               Collector<Integer> out) throws Exception {  
        out.collect(value);  
    }  
}
```

使用 `ProcessFunctionTestHarnesses` 对  `ProcessFunction` 进行测试非常简单，传递合适的参数并验证输出：

```java
class PassThroughProcessFunctionTest {  
  
    @Test
    void testPassThrough() throws Exception {
        // 实例化 UDF
        PassThroughProcessFunction processFunction = new PassThroughProcessFunction();

        // 包装 UDF 至相应的 TestHarness 中
        OneInputStreamOperatorTestHarness<Integer, Integer> testHarness =
                ProcessFunctionTestHarnesses.forProcessFunction(processFunction);

        // 推送元素及与该元素关联的时间戳至算子中
        testHarness.processElement(1, 10);

        // 获取输出并断言
        assertThat(testHarness.extractOutputValues())
                .isEqualTo(Collections.singletonList(1));
    }
  
}
```

有关如何使用 `ProcessFunctionTestHarnesses` 来测试不同类型的 `ProcessFunction`（如 `KeyedProcessFunction`, `KeyedCoProcessFunction`, `BroadcastProcessFunction`）的更多示例，可以查看 [Flink 代码仓库](https://github.com/apache/flink/tree/master/flink-streaming-java/src/test/java/org/apache/flink/streaming/util) 的 `ProcessFunctionTestHarnessesTest`。

## 测试 Flink Job

### JUnit 4 Rule MiniClusterWithClientResource

在 JUnit 4 中 Apache Flink 提供了一个名为  `MiniClusterWithClientResource` 的 Rule（规则），用于在本地嵌入一个迷你集群，以便测试完整的 Job。

让我们以上面提到的 `IncrementMapFunction` 为例，在本地 Flink 集群测试一个使用 `MapFunction` 的简单 Job：

```java
public class JUnit4ExampleIntegrationTest {  
  
    @ClassRule  
    public static MiniClusterWithClientResource flinkCluster = new MiniClusterWithClientResource(  
            new MiniClusterResourceConfiguration.Builder()  
                    .setNumberSlotsPerTaskManager(2)  
                    .setNumberTaskManagers(1)  
                    .build());  
  
    @Test  
    public void testIncrementPipeline() throws Exception {  
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();  
  
        // configure your test environment  
        env.setParallelism(2);  
  
        // values are collected in a static variable  
        CollectSink.values.clear();  
  
        // create a stream of custom elements and apply transformations  
        env.fromElements(1L, 21L, 22L)  
                .map(new IncrementMapFunction())  
                .addSink(new CollectSink());  
  
        // execute  
        env.execute();  
  
        // verify your results  
        assertTrue(CollectSink.values.containsAll(Lists.list(2L, 22L, 23L)));  
    }  
  
    // create a testing sink  
    private static class CollectSink implements SinkFunction<Long> {  
  
        // must be static  
        public static final List<Long> values = Collections.synchronizedList(new ArrayList<>());  
  
        @Override  
        public void invoke(Long value, SinkFunction.Context context) throws Exception {  
            values.add(value);  
        }  
    }  
}
```

关于使用 `MiniClusterWithClientResource` 集成测试的几点说明：

- 为了避免在测试时拷贝整个 job pipeline 的代码，你需要将你生产代码的 source 和 sink 设计成可插拔的，以便在测试中注入特殊的测试的 source 和 测试的 sink。
- 这里使用 `CollectSink` 中的静态变量，是因为 Flink 在将所有算子分布到整个集群之前先对其进行了序列化，解决此问题的方法之一是与本地 Flink 迷你集群通过实例化算子的静态变量进行通信，或者，你可以使用测试的sink 将数据写入临时目录中的文件。
- 如果你的 job 使用了事件时间 timer ，你可以实现自定义的并行 source 函数来发送水印（watermark）。
- 建议始终使用 `parallelsim > 1` 的方式在本地测试 pipeline，以便发现那些只有 pipeline 在并行执行时才会出现的 bug。
- 优先使用 `@ClassRule` 而不是 `@Rule`，这样多个测试能够共享同一个 Flink 集群，这样做能够节约大量的时间，因为 Flink 集群的启动和关闭通常占据了实际测试的执行时间。
- 如果你的 pipeline 包含自定义状态处理，则可以通过启用 checkpoint 并在迷你集群中重新启动作业来测试其正确性，为此，你需要在 pipeline 的（仅在测试使用的）用户自定义函数中抛出异常来触发失败。

### JUnit 5  MiniClusterExtension

在 JUnit 5 中已不再支持 `@ClassRule` 和 `@Rule`，分别使用 `@RegisterExtension` 和 `@ExtendWith` 去替换。

在 JUnit 5 中，Flink 提供了 `MiniClusterExtension` 的扩展用于在本地启动一个 Flink 集群并注册相应的执行环境，使用 JUnit 5 编写  `IncrementMapFunction` Job 的测试用例如下：

```java
@ExtendWith(MiniClusterExtension.class)
class JUnit5ExampleIntegrationTest {
	@Test
	void testIncrementPipeline() {
		ExecutionEnvironment execEnv = ExecutionEnvironment.getExecutionEnvironment();
		//...omit
	}
}
```

如果需要调整集群的配置，你可以使用 `@RegisterExtension` ：
```java
class JUnit5ExampleIntegrationTest {

	@RegisterExtension
	public static final MiniClusterExtension MINI_CLUSTER_RESOURCE = new 
		MiniClusterExtension(
			new MiniClusterResourceConfiguration.Builder()
				 .setNumberTaskManagers(1)
				 .setConfiguration(new Configuration())
				 .build());
		
	@Test
	void testIncrementPipeline() {
		ExecutionEnvironment execEnv = ExecutionEnvironment.getExecutionEnvironment();
		//...omit
	}
}
```

## 总结

本节我们展示了如何在 Apache Flink 中为 **无状态、有状态和时间感知（timer）** 的算子编写单元测试，以及如何为 Job 进行集成测试。

👋 [访问 GitHub 获取源代码](https://github.com/xbhel/flink-starter/tree/main/flink-unit-test)。


