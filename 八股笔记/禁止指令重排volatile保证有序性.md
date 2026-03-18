指令重排可能会导致双重检查锁失效，比如下面的单例模式代码：

```
public class Singleton {
    private static Singleton instance;

    public static Singleton getInstance() {
        if (instance == null) { // 第一次检查
            synchronized (Singleton.class) {
                if (instance == null) { // 第二次检查
                    instance = new Singleton(); // 可能发生指令重排
                }
            }
        }
        return instance;
    }
}
```

如果线程 A 执行了 `instance = new Singleton();`，但构造方法还没执行完，线程 B 可能会读取到一个未初始化的对象，导致出现空指针异常。

![三分恶面渣逆袭：双重校验单例模式异常情形](https://cdn.paicoding.com/tobebetterjavaer/images/sidebar/sanfene/javathread-22.png)

三分恶面渣逆袭：双重校验单例模式异常情形

正确的方式是给 instance 变量加上 `volatile` 关键字，禁止指令重排。

```
class Singleton {
    private static volatile Singleton instance;

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton(); // 由于 volatile，禁止指令重排
                }
            }
        }
        return instance;
    }
}
```

# 1. 先给结论（面试必背）

1. **synchronized 可以保证：最终结果有序**
2. **synchronized 不能禁止：代码块内部的指令重排**
3. **synchronized 能保证：代码块外部不乱进、乱出**
4. **真正能完全禁止指令重排的只有 volatile（内存屏障）**

---

# 2. 最核心区别

## **volatile = 禁止指令重排（内存屏障），保证可见性和有序性**

## **synchronized = 不禁止内部重排，但保证原子性 + 可见性**+有序性
**有序性**：
- 不是 “禁止内部指令重排”
- 而是**临界区内重排不影响结果**，且**指令不能重排进出锁**
- 对外表现 = **完全有序**