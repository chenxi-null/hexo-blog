title: 写给 Javaer 看的 Spock 快速入门
author: chenxi
tags:
  - tdd
categories: 
  - tdd
date: 2021-04-14 22:00:00
---

# 写给 Javaer 看的 Spock 快速入门

## Spock 是什么？

看下它的 [github](https://github.com/spockframework/spock) 描述:

> The Enterprise-ready testing and specification framework.

[官网介绍](http://spockframework.org/):

> Spock is a testing and specification framework for Java and Groovy applications.
>
> What makes it stand out from the crowd is its beautiful and highly expressive specification language.
>
> Thanks to its JUnit runner, Spock is compatible with most IDEs, build tools, and continuous integration servers.
>
> Spock is inspired from JUnit, RSpec, jMock, Mockito, Groovy, Scala, Vulcans, and other fascinating life forms.

官方文档：[Spock Framework Reference Documentation](http://spockframework.org/spock/docs/1.2/all_in_one.html)
<br>

简而言之，Spock 是一款测试框架，支持 Java 或者 Groovy 应用，是采用 groovy 语言写的，得益于 groovy 的语言特性，Spock 相比较 JUnit 语法更简练、提供更强大的功能。

<br>

## 和其他 Java 测试框架的对比
对比其他的 Java 测试框架，包括常见的 JUnit, testNG, 一些 Java BDD 框架, 例如 Cucumber(Gherkin)，[JBehave](https://jbehave.org/), Spock 有如下优势:

- 基于它提供的一套 DSL，能更好地实践 BDD —— 测试用例即需求说明书。 (`given-when-then`)

- 写更少的测试代码，测试用例可读性更强，一些 awesome feature，后面会详细解释:
  - blocks, (no)thrown, '==',
  - where, table, database,
  - with, verifyAll
  - 更灵活的 Mock/Stub 语法

- 兼容 `JUnit`, 支持 JUnit 里的所有 `@Rule`，可以平滑地从 JUnit 迁移到 Spock

### Spock vs JUnit 
通过一个参数化测试的例子直观感受下：

Spock:
```groovy
import spock.lang.Specification
import spock.lang.Unroll

@Title("Testing file extension validation method")
class ImageValidatorShould extends Specification {
  
   @Unroll
   def "validate extension of #fileToValidate"() {
       when: "validator checks filename"
       def isValid = validate fileToValidate

       then: "return appropriate result"
       isValid == expectedResult

       where: "input files are"
       fileToValidate || expectedResult
       'some.jpeg'    || true
       'some.jpg'     || true
       'some.tiff'    || false
       'some.bmp'     || true
       'some.png'     || false
   }
}
```

JUnit:
```java
import org.junit.Test;
import org.junit.runner.RunWith;
import org.junit.runners.Parameterized;
import org.junit.runners.Parameterized.Parameters;

import java.util.Collection;

import static java.util.Arrays.asList;
import static org.junit.Assert.assertEquals;

@RunWith(Parameterized.class)
public class ImageValidator {

   @Parameters
   public static Collection<Object[]> data() {
       return asList(new Object[][]{
               {"some.jpeg", true},
               {"some.jpg", true},
               {"some.tiff", false},
               {"some.bmp", true},
               {"some.png", false}
       });
   }

   private String file;
   private boolean isValid;

   public ImageValidator(String input, boolean expected) {
       file = input;
       isValid = expected;
   }

   @Test
   public void validateFileExtension() {
       assertEquals(isValid, validate(file));
   }
}
```

上述例子摘自：[link](https://www.blazemeter.com/blog/spock-vs-junit-which-one-should-you-choose)

---

## 基本用法

### 测试方法的生命周期
和 JUnit 基本类似，详见：[对比 JUnit](http://spockframework.org/spock/docs/1.2/all_in_one.html#_comparison_to_junit)



![Blocks](http://spockframework.org/spock/docs/1.2/images/Blocks2Phases.png)

e.g.:
```groovy
def "events are published to all subscribers"() {
  given: "准备数据"
  def subscriber1 = Mock(Subscriber)
  def subscriber2 = Mock(Subscriber)
  def publisher = new Publisher()
  publisher.add(subscriber1)
  publisher.add(subscriber2)

  when: "触发被测试对象的一个行为"
  publisher.fire("event")

  then: "验证该行为的结果是否符合预期"
  // 验证 subscriber1 的 receive 方法是否被调用一次，且入参为 "event"
  1 * subscriber1.receive("event")
  1 * subscriber2.receive("event")
}
```

### 简洁的 Mock 语法
Spock 有自带的一套 Mock 框架，提供非常简洁直观的语法，从上面那个例子也能看出它的可读性很强，如果同样的代码用 Java Mock 框架，如 Mockito，来写的话，是这样：
```java
Mockito.verify(subscriber1, Mockito.times(1)).sendMail("event");
Mockito.verify(subscriber2, Mockito.times(1)).sendMail("event");
```

### 隐式断言

> Within the `then` and `expect` blocks, assertions are implicit

无需显示写 assert，语法简洁

```groovy
when:
//...

then:
// `then` 代码块里面直接写条件判断语句，无需显式调用 assert 方法
// 等价于: Assert.assertTrue(mailBox.containsMail(msg))
mailBox.containsMail(msg)

// 等价于: Assert.assertEquals(name, 'value')
name == 'value'
```



### 异常断言

`thrown, notThrown`
```groovy
when:
stack.pop()

then:
thrown(EmptyStackException)
stack.empty
```

### 各种 Groovy 的语法糖

Groovy 提供的很多语法糖帮助开发不需要写很多模板代码，使用 Spock 写的测试用例更简洁易读，得益于它提供的这套 DSL，整个测试用例的结构更清晰、看起来更接近自然语言。

e.g:
> Java’s == is actually Groovy’s is() method, and Groovy’s == is a clever equals()!

```groovy
str == "content"     // assert the equality between strings
list == [1, 2, 3, 4] // assert the equality between lists
```

```groovy
publisher.subscribers << subscriber // << is a Groovy shorthand for List.add()
publisher.subscribers - subscriber // to remove element from List
```

参考:
[groovy-lang.org/syntax.html](http://groovy-lang.org/syntax.html)  
[更多例子](https://learnxinyminutes.com/docs/groovy/)

### 支持 Java 的各种测试框架

包括 JUnit, Mockito, JAssert, Hamcrest, 等等

### 参数化测试

杀手级特性

#### Data Tables

```groovy
class MathSpec extends Specification {
  def "maximum of two numbers"(int a, int b, int c) {
    expect:
    Math.max(a, b) == c

    where:
    a | b | c
    1 | 3 | 3
    7 | 4 | 7
    0 | 0 | 0
  }
}
```

#### Data Pipes

```groovy
...
where:
a << [1, 7, 0]
b << [3, 4, 0]
c << [3, 7, 0]
```

### 报告生成

`given` `when` `then` 等代码块里的描述是可以导出到文档中的：

<img src="https://p5.music.126.net/obj/wo3DlcOGw6DClTvDisK1/8325479993/2df3/bece/ef4c/83442768d591857df3dc6bbd33a75d2e.png" width="500"/>

测试用例是给开发人员看的文档，BDD 风格的测试用例可读性更强，帮忙阅读者更快地熟悉复杂的业务逻辑。

---

## 从 Java 迁移到 Groovy 的注意事项

### java lambda expression vs groovy closure

java:

```java
() 
e.g: Stream.of(1, 2, 3).map(i -> i * 2)
```

groovy:

```groovy
{}
e.g: Stream.of(1, 2, 3).map { i -> i * 2 }
```

### 注解当中的：`[]` vs` {}`

java:

```java
@ContextConfiguration(classes = {A.class, B.class})
```

groovy:

```groovy
@ContextConfiguration(classes = [A.class, B.class])
@ContextConfiguration(classes = [A, B]) // the '.class' suffix can also be ignored
String[] arr = ["a", "b", "c"]
```

### groovy 更便捷的 getter & setter

```groovy
class Person {
    String name;
    int age;
}

Person person = new Person()
person.name = 'tom' // 等价于 person.setName("tom");
person.age = 10 // 等价于 person.setAge(10);
println person.name // 等价于 System.out.println(person.getName())
println person.age // 等价于 System.out.println(person.getAge())
```



## 落地实践

<img src="https://p6.music.126.net/obj/wo3DlcOGw6DClTvDisK1/8325266133/7d08/48cb/0fc4/a4d9b708ff9dd8ddb7eea5bfb2ada270.png" width="500"/>

生产代码用 java 写，测试代码用 groovy，享受新语言优势的同时又无需承担线上风险。
e.g.
<img src="https://p5.music.126.net/obj/wo3DlcOGw6DClTvDisK1/8325587220/e46f/595f/de6c/83ff35430e1c5f2c4a6ab0838cb86cf7.png" width="500"/>

### 学习成本
- groovy 号称是 "Java 的脚本的语言"，Java 程序员学习 groovy 的成本较低。
- Spock 中文资料不是很多，直接阅读[官方文档](https://github.com/spockframework/spock)是最推荐的学习方式

### 兼容性
如果对 groovy 语法不是太熟悉的话，可以选择在 Spock 里写 Java 代码，groovy 兼容绝大多数的 java 语法。 e.g.:
```java
// 各种 groovy 语法和 Java 语法的混用，
List list0 = new ArrayList()
List list = ['1', '2', '3']
list << '4'
list << "5"
list.add("6")
println(list[0])
println list.get(0)
System.out.println(list[0])
```

groovy 还兼容 JUnit (Spock 的底层是基于 Junit 引擎实现的)和 maven (因为 maven 构建只认 class 文件)，因此也兼容 CI 平台。

### 不足之处
1. 跨语言，有一定学习成本和推广难度

2. 社区活跃度和 JUnit 还不是一个数量级，我之前在使用过程中也遇到过一些小问题，不过影响不大

3. 习惯 Spock 之后容易厌弃 JUnit ......
分享一个之前在 spring-redis 源码里看到的彩蛋，这位老哥可能是写单测写太辛苦了，直接在代码注释里吐槽起了 JUnit，甚至还画了一个表情 (摊手)  
    <img src="https://p5.music.126.net/obj/wo3DlcOGw6DClTvDisK1/8325339447/c6e0/1999/0dd8/2857d4e659391fcb1aefcff280f9b453.png" width="500"/>

### maven 配置

pom.xml:
```xml
</dependencies>
    <!-- spock -->
    <dependency>
        <groupId>org.codehaus.groovy</groupId>
        <artifactId>groovy-all</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.spockframework</groupId>
        <artifactId>spock-core</artifactId>
        <version>1.2-groovy-2.4</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.spockframework</groupId>
        <artifactId>spock-spring</artifactId>
        <version>1.2-groovy-2.4</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>com.athaydes</groupId>
        <artifactId>spock-reports</artifactId>
        <version>1.6.1</version>
        <scope>test</scope>
    </dependency>
</dependencies>


<build>
    <plugins>
        <plugin>
            <groupId>org.codehaus.gmavenplus</groupId>
            <artifactId>gmavenplus-plugin</artifactId>
            <version>1.6</version>
            <executions>
                <execution>
                    <goals>
                        <goal>compileTests</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>

        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <configuration>
                <includes>
                    <include>**/*Test.java</include>
                    <include>**/*Spec.java</include>
                </includes>
            </configuration>
        </plugin>
    </plugins>
</build>
```