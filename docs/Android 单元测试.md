单元测试有很多框架：
Java单元测试框架：**Junit、Mockito、Powermockito**等；
Android：**Robolectric、AndroidJUnitRunner、Espresso**等。

基本语法：

### assertEquals

**验证是否相等：**
Assert.assertEquals(result, 3); // 验证result==3，如果不正确，测试不通过

### Verify
**验证函数是否被调用（以及调用了多少次）：**
 
 calculater.add(1, 2);
 verify(calculater).add(1, 2);  // 验证calculater.add(a, b)是否被调用过，且a==1 && b==2

### Mockito框架

```java
public interface IMathUtils {
    public int abs(int num); // 求绝对值
}
public class MockTest {
    public static void main(String[] args) {
        IMathUtils mathUtils = mock(IMathUtils.class); // 生成mock对象
        when(mathUtils.abs(-1)).thenReturn(1); // 当调用abs(-1)时，返回1
        int abs = mathUtils.abs(-1); // 输出结果 1
        Assert.assertEquals(abs, 1);// 测试通过
    }
}
```

可以发现 IMathUtils 是一个接口，根本就没有实现，用 Mockito 框架 mock 之后，IMathUtils.abs(-1) 就有返回值 1 了。这就是 Mockito 神奇的地方！Mockito 代理了 IMathUtils.abs(num) 的行为，**只要调用时符合指定参数（代码中指定参数 -1），就可以得到映射的返回值。**  
Mockito 的语法 **when...thenReturn...** 相当直观。
