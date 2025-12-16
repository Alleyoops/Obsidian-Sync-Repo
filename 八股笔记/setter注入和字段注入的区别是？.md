Setter 注入和字段注入，都是在 Spring Bean 生命周期的「属性填充阶段（populateBean）」完成依赖注入的。

实例化->属性填充->初始化

---

# 一、什么是 setter 注入？（一句话定义）

> **Setter 注入：  
> 是指 Spring 在 Bean 实例化完成之后，通过调用 JavaBean 规范的 `setXxx()` 方法，把依赖对象注入进来。**

关键词只有三个：

- **实例化之后**
    
- **调用 set 方法**
    
- **注入依赖**
    

---

# 二、一个最标准的 setter 注入例子

```java
@Component
class A {

    private B b;

    @Autowired
    public void setB(B b) {
        this.b = b;
    }
}
```

对应 JavaBean 规范：

```text
属性名：b
setter：setB(...)
```

---

# 三、setter 注入在 Spring 中的完整流程（重点）

我们从 **Spring 启动 → Bean 创建** 的真实顺序走一遍。

---

## Step 1：Spring 决定创建 Bean A

- 在 `getBean("A")` 时触发
    
- A 是 singleton
    

---

## Step 2：实例化 A（Instantiation）

```java
A a = new A();
```

此时：

- 对象存在
    
- `b == null`
    
- 构造器已执行完
    

---

## Step 3：提前暴露 A（三级缓存介入点）

如果 A 是 singleton：

```text
singletonFactories.put("A", ObjectFactory)
```

> ⚠️ 注意：  
> **setter 注入发生在这一步之后**

---

## Step 4：属性填充阶段（populateBean）

Spring 开始扫描 A 上的注入点：

```java
@Autowired
public void setB(B b)
```

发现：A 依赖 B。

---

## Step 5：获取依赖 B

Spring 调用：

```java
getBean("B")
```

### 情况 1：B 尚未创建

- Spring 开始创建 B
    
- 若 B 又依赖 A → 循环依赖
    
- 从三级缓存 / 二级缓存中获取 A 的前期暴露对象
    

---

## Step 6：调用 setter 方法注入

```java
a.setB(b);
```

这一步是 setter 注入的“本体”。

---

## Step 7：初始化阶段（Initialization）

setter 注入完成后：

- @PostConstruct
    
- InitializingBean
    
- BeanPostProcessor
    
- AOP 代理（如果有）
    

---

## Step 8：放入一级缓存

```text
singletonObjects["A"] = A（或代理）
```

---

# 四、setter 注入的关键特性（结合流程理解）

|特性|原因|
|---|---|
|可以解决循环依赖|注入发生在实例化之后|
|可以被三级缓存支持|有“空壳对象”可暴露|
|可与 @Lazy 配合|setter 可接收 Lazy Proxy|
|依赖可选|setter 可不调用|
|可被覆盖|多次调用 setter|

---

# 五、和字段注入的流程对比（非常关键）

|对比点|setter 注入|字段注入|
|---|---|---|
|注入方式|调用方法|反射赋值|
|Spring 处理阶段|populateBean|populateBean|
|是否实例化后|✅|✅|
|是否支持循环依赖|✅|✅|
|是否支持 @Lazy|✅|✅|

👉 **Spring 内部流程几乎一样，差别只在“赋值方式”**

---

# 六、什么时候用 setter 注入？（实践建议）

✔ 依赖是可选的  
✔ 需要后期替换 / mock  
✔ Bean 生命周期复杂  
✔ 老代码 / XML 风格

❌ 强依赖  
❌ 希望对象不可变

---

# 七、一句话总结（面试版）

> **Setter 注入是 Spring 在 Bean 实例化之后、初始化之前，通过调用 set 方法完成依赖注入的一种方式；  
> 由于注入发生在实例化之后，它可以配合三级缓存解决单例循环依赖。**

如果你愿意，下一步我可以：

- 把 **setter / 字段 / 构造器注入** 的完整生命周期画成一张图
    
- 或结合你前面问的 **三级缓存**，再走一次 setter 注入 + 循环依赖的源码级流程






---
setter注入和字段注入的区别是？
## 一、先给结论（可以直接背）

> **在 Spring 中：**
> 
> - **字段注入（Field Injection）和 setter 注入（Setter Injection）  
>     在 Bean 生命周期、三级缓存、循环依赖能力上是等价的**
>     
> 
> 但在：
> 
> - **可测试性**
>     
> - **可维护性**
>     
> - **设计表达力**
>     
> 
> 上存在**本质差异**

---

## 二、从 Spring 内部看：为什么它们“等价”？

### 1️⃣ Spring 注入发生在同一阶段

无论字段注入还是 setter 注入，Spring 都是在：

```text
实例化完成之后
→ 属性填充阶段（populateBean）
```

完成依赖注入。

因此：

|维度|字段注入|setter 注入|
|---|---|---|
|是否在实例化后|✅|✅|
|是否可提前暴露|✅|✅|
|是否支持三级缓存|✅|✅|
|是否支持循环依赖|✅|✅|

👉 **从“循环依赖解决能力”看，两者完全一样**

---

## 三、那它们真正的区别在哪里？（重点）

区别不在 Spring 能不能用，而在**你如何“表达依赖关系”**。

---

## 四、字段注入（Field Injection）

### 示例

```java
@Component
class A {
    @Autowired
    B b;
}
```

### 特点

#### ✅ 优点

- 写起来最省事
    
- 代码简洁
    

#### ❌ 缺点（重要）

1. **依赖是“隐式的”**
    
    - 看构造器不知道需要什么
        
2. **无法设置 final**
    
3. **不利于单元测试**
    
4. **强耦合 Spring**
    
5. **IDE / 静态分析支持弱**
    

---

## 五、setter 注入（Setter Injection）

### 示例

```java
@Component
class A {
    private B b;

    @Autowired
    public void setB(B b) {
        this.b = b;
    }
}
```

### 特点

#### ✅ 优点

1. **依赖是显式的**
    
2. **支持可选依赖**
    
3. **更利于测试（可手动 set）**
    
4. **更符合 JavaBean 规范**
    

#### ❌ 缺点

- 有可能被“半初始化使用”
    
- 依赖非强制（可被忘记注入）
    

---

## 六、和构造器注入对比（你现在正好关心）

|注入方式|是否推荐|原因|
|---|---|---|
|构造器注入|⭐⭐⭐⭐⭐|不可变、显式、易测|
|setter 注入|⭐⭐⭐|适合可选依赖|
|字段注入|⭐|方便但隐藏问题|

> **Spring 官方文档推荐：**
> 
> - 强依赖 → 构造器注入
>     
> - 可选依赖 → setter 注入
>     
> - 不推荐字段注入
>     

---

## 七、为什么很多老项目大量用字段注入？

历史原因：

- XML / setter 注入 → 注解时代 → 字段注入更方便
    
- 早期 Spring 对构造器注入支持不友好
    
- 开发效率优先
    

但这并不代表是“好设计”。

---

## 八、和循环依赖的关系（你前面讨论的点）

|注入方式|循环依赖表现|
|---|---|
|字段注入|Spring 可自动解决|
|setter 注入|Spring 可自动解决|
|构造器注入|❌（除非 @Lazy）|

👉 **这也是为什么很多循环依赖项目“刚好没炸”**

---

## 九、一句话终极总结

> **字段注入和 setter 注入在 Spring 内部处理机制上几乎完全一致，但在“设计表达力和可维护性”上差异巨大；  
> setter 注入显式、可测试，字段注入隐式、脆弱。**

如果你愿意，我可以下一步帮你：

- 把 **字段 / setter / 构造器注入** 放到一张“设计权衡表”里
    
- 或结合你前面问的 **循环依赖 + @Lazy** 给出一个“最佳实践组合方案”