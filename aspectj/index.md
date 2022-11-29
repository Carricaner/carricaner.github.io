# Intro to AOP & AspectJ


<!--more-->

## 前言 <a id="preface"></a>

一直都有聽說 AOP ( Aspect-Oriented Programming ) 的概念，但是一直沒有真正的動手實作。

利用這次寫這篇文章的機會，簡單整理一下 AOP 相關的概念以及實作。

( Fact: 在跟 2022年 say goodbye 之前，補一下部落格發文進度 :smirk: )

<br>
{{< figure src="images/waeving-unsplash.jpeg" caption="*Photo by Yaniv Knobel in Unsplash*">}}
<br>

## 本文 <a id="main-content"></a>

### 相關名詞介紹<a id="term-intro"></a>

#### AOP ( Aspect-Oriented Programming )

{{< admonition type=quote title="AOP's definition" open=true >}}
剖面導向程式設計 (AOP) 是電腦科學中的一種程式設計泛型，旨在將橫切關注點與業務主體進行進一步分離，以提高程式碼的模組化程度 --- wiki
{{< /admonition >}}

再用下面這張圖來說明：
<br>
{{< figure src="images/aop-demo.png" caption="*Photo derived from <a href='https://codegym.cc/groups/posts/543-what-is-aop-principles-of-aspect-oriented-programming'>Konstantin's article</a>*">}}
<br>

AOP 設計利用橫切的方式( cross-cutting ) 來加入額外邏輯。

像是圖中的「日誌 (Logging)」以及「安全驗證 (Security)」，這樣一來就可以避免在各個既有模組中一一加入邏輯，處理起來複雜、耗時且難以維護。
<br>

#### Join point
切面設計的切入點，會需要搭配 Pointcut 。

#### Pointcut
一種特殊的通用表示式 ( Regular Expression ) ，被用來定義是在哪些情況下應該要切入，並且依附某個切入點 ( Join Point )。

#### Advice
進入切入點之後要執行的邏輯。

#### Aspect
為收集各個地方的 cross-cutting concerns 之後獨立且可以重用的物件，通常一個 Aspect 會專注在一種邏輯上，像是日誌等等。

以上名詞之間的關係如下圖：

<br>
{{< figure src="images/aop-integration.jpeg" 
caption="*Photo derived from <a href='https://openhome.cc/Gossip/SpringGossip/AOPConcept.html'>openhom.cc</a> <br>( 感謝良葛格對台灣技術的貢獻，本人從中受益良多，R.I.P. -- 2022/11 )*">}}
<br>

### 使用 AspectJ<a id="aspectj"></a>
{{< admonition type=question title="What is AspectJ ?" open=true >}}
AspectJ 是一種實現 AOP 的 Java 擴充套件 --- wiki
{{< /admonition >}}

<br>

#### 安裝
For Maven
```xml
<dependency>
    <groupId>org.aspectj</groupId>
    <artifactId>aspectjrt</artifactId>
    <version>1.9.9.1</version>

    <groupId>org.aspectj</groupId>
    <artifactId>aspectjweaver</artifactId>
    <version>1.9.9.1</version>
</dependency>
```
<br>

#### 情境一：加入日誌 ( Logging )<a id="example-1-logging"></a>

:bulb: 假如我們想要針對 `org.sample.entity` 目錄底下所有的物件，在呼叫 `getters` & `setters` 的前後都紀錄我們客製化的日誌。

<br>

目錄底下的物件`Car`
```java
package org.example.entity;

public class Car {
  private String name;

  public Car(String name) {
    this.name = name;
  }

  public String getName() {
    return name;
  }

  public void setName(String name) {
    this.name = name;
  }
}
```

新增一個`EntityAspect`來處理這個橫切邏輯。
```java
package org.example.aop;

import java.util.logging.Logger;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.After;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;

@Aspect
public class EntityAspect {
  private final Logger logger = Logger.getLogger(EntityAspect.class.getName());

  @Pointcut("execution(* org.example.entity.*.get*()) "
      + "|| execution(* org.example.entity.*.set*(..))")
  public void getterAndSetterJointPoint() {}

  @Before("getterAndSetterJointPoint()")
  public void beforeGetterAndSetterAdvice() {
    logger.info("[EntityAspect] --> Before advice executed!");
  }

  @After("getterAndSetterJointPoint()")
  public void afterGetterAndSetterAdvice() {
    logger.info("[EntityAspect] --> After advice executed!");
  }

}
```
`@Pointcut`中的表示式包含 jointpoint signature ( 範例中為 `execution`) 以及 regular expression ( 範例中為 `* org.example.entity.*.get*()`)。

其中還可以搭配邏輯運算元 ( 範例中為 `||` )，更多的規則可以參考<a href="https://cloud.tencent.com/developer/article/1497814">連結</a>。

`@Before` & `@After` 指定 Advice 的邏輯是在 Join point 之前還是之後執行，裡面填入要綁定的 Pointcut method ( 範例中為`getterAndSetterJointPoint()` )。

<br>

在測試中看看結果

```java
import org.example.entity.Car;
import org.junit.jupiter.api.Test;

class RawTest {
  @Test
  void testGetterAdnSetter() {
    Car car = new Car("SpeedWagon");
    System.out.println("The car's name: " + car.getName());
  }

  // Output:
  // Nov 28, 2022 3:06:22 PM org.example.aop.EntityAspect beforeGetterAndSetterAdvice
  // INFO: [EntityAspect] --> Before advice executed!
  // Nov 28, 2022 3:06:22 PM org.example.aop.EntityAspect afterGetterAndSetterAdvice
  // INFO: [EntityAspect] --> After advice executed!
  // Car's name: SpeedWagon
}
```
可以看到在呼叫 getter 的前 ( Before ) 後 ( After ) 成功橫切插入我們要執行的邏輯。

<br>

#### 情境二：檢查物件 ( Validation )<a id="example-2-validation"></a>

:bulb: 假如我們想要針對 `org.sample.entity` 目錄底下所有的物件在使用 `new` 來產生 instance 的時候，檢查裡面的欄位是否符合我們要的邏輯。

<br>

目錄底下的物件`Triangle`
```java
package org.example.entity;

import java.util.Optional;

public class Triangle {
  private float[] sides;

  public Triangle(float[] sides) {
    this.sides = sides;
  }

  // To validate if the triangle is a valid one.
  public Optional<String> validateTheTriangle() {
    if (sides.length != 3) return Optional.of("Sides number must be 3.");

    for (float side : sides) {
      if (side <= 0) return Optional.of("Sides must be larger than 0.");
      if (sides[0] + sides[1] <= sides[2]
          || sides[1] + sides[2] <= sides[0]
          || sides[0] + sides[2] <= sides[1])
        return Optional.of("The sides cannot form a triangle.");
    }

    return Optional.empty();
  }
}
```

<br>

新增一個`TriangleAspect`來處理這個橫切邏輯。
```java
package org.example.aop;

import java.util.Optional;
import java.util.logging.Logger;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.After;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.example.entity.Triangle;

@Aspect
public class TriangleAspect {
  private final Logger logger = Logger.getLogger(TriangleAspect.class.getName());

  @Pointcut("execution(org.example.entity.*.new(..))")
  public void triangleJointPoint() {  }

  @After("triangleJointPoint()")
  public void afterCreatingTriangleAdvice(JoinPoint joinPoint) {
    Triangle instance = (Triangle) joinPoint.getTarget();
    Optional<String> op = instance.validateTheTriangle();
    if (op.isPresent()) {
      logger.info("Invalid due to: " + op.get());
    } else {
      logger.info("Valid!");
    }
  }
}
```

<br>

在測試中看看結果

``` java
import org.example.entity.Triangle;
import org.junit.jupiter.api.Test;

class RawTest {
  @Test
  void testValidTriangle() {
    Triangle triangle = new Triangle(new float[]{12F, 13F, 14F});
  }
  // Output:
  // Nov 28, 2022 4:50:48 PM org.example.aop.TriangleAspect afterCreatingTriangleAdvice
  // INFO: Valid! 

  @Test
  void testInValidTriangle() {
    Triangle triangle = new Triangle(new float[]{50F, 13F, 14F});
  }
  // Output:
  // Nov 28, 2022 4:48:21 PM org.example.aop.TriangleAspect afterCreatingTriangleAdvice
  // INFO: Invalid due to: The sides cannot form a triangle.
}
```
可以看到在呼叫 new 的時候，我們成功的執行了在 `Triangle` 裡面的檢查 (`validateTheTriangle`)，這樣一來就可以利用 AOP 的方式來判斷三角形是不是合理的。

<br>

## 總結<a id="conclusion"></a>
- 剖面導向程式設計 AOP ( Aspect-Oriented Programming ) 是一種設計方式，旨在將橫切關注點與業務主體分離，提高分離程度。

- AOP 相關的概念：Join point, Pointcut, Advice & Aspect。

- AspectJ 是一種實現 AOP 的 Java 擴充套件。相關的套件還有 <a href="https://docs.spring.io/spring-framework/docs/2.5.5/reference/aop.html">Spring AOP</a>。 

- AOP 可以利用的情境有 Tracing, Profiling & Logging, Pre- and Post- conditions 等等，可以參考官方列出的 <a href="https://www.eclipse.org/aspectj/doc/released/progguide/starting-development.html">情境</a>。

<br>

## 參考<a id="refs"></a>
- [AOP 觀念與術語](https://openhome.cc/Gossip/SpringGossip/AOPConcept.html)

- [Intro to AspectJ](https://www.baeldung.com/aspectj)

- [AOP 與 Pointcut 淺談](https://bingdoal.github.io/backend/2020/11/aop-and-point-cut-in-spring-boot/)

- [【Spring Boot】第20課－切面導向程式設計（AOP）](https://chikuwa-tech-study.blogspot.com/2021/06/spring-boot-aop-introduction.html)

- [Introduction to Pointcut Expressions in Spring](https://www.baeldung.com/spring-aop-pointcut-tutorial)

- [【小家Spring】Spring AOP中@Pointcut切入点表达式最全面使用介绍](https://cloud.tencent.com/developer/article/1497814)

- [來談談 AOP (Aspect-Oriented Programming) 的精神與各種主流實現模式的差異](https://tech-blog.cymetrics.io/posts/maxchiu/aop/)


