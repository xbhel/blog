## 在 `main` 和 `test`  方法中获取路径会出现不同的结果?

当我们在使用多模块（聚合）项目时，在 `main` 和 Junit `test` 方法中获取路径可能会得到不同的结果。假设我们有如下项目结构，app 作为父工程统一管理依赖，*app* 下包含 *web* 和 *common* 两个子模块：

```css
app
| -- web
| -- common
| -- src/test/java
|	| -- PathRepresentationTest.java
```

*PathRepresentationTest.java* 的内容如下：

```java
import org.junit.jupiter.api.Test;
import java.nio.file.Path;
import java.nio.file.Paths;

class PathRepresentationTest {
    
    public static void main(String[] args) {
        Path path = Paths.get(".");
        System.out.println("main: " + path.toAbsolutePath());
    }
    
    @Test
    void testPathRepresentation() {
        Path path = Paths.get(".");
        System.out.println("test: " + path.toAbsolutePath());
    }
}

// output
// main: D:\projects\app\.
// test: D:\projects\app\common\.
```

在 `main` 方法中输出的当前路径是当前**工程**的根路径，而在 `test` 方法中输出的当前路径是当前**模块**的根路径。相同的代码却得到不同的结果，这是因为 `main` 针对是整个工程，所以起始路径是当前工程的根路径，而 Junit 针对的是当前模块，对当前模块进行测试，所以起始路径是当前模块的根路径。

这个问题通常只在多模块项目中出现，因为单模块项目当前模块就是整个工程。

 

