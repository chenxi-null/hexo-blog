title: A Spock Tutorial for Javaer
author: chenxi
tags:
  - tdd
categories: 
  - tdd
date: 2019-01-28 17:00:00
---

# A Spock Tutorial for Javaer

### What is it?

[](https://github.com/spockframework/spock)

The description of [its github repository](https://github.com/spockframework/spock):

> The Enterprise-ready testing and specification framework.

<br>

A Quote from the [office website](http://spockframework.org/):

> Spock is a testing and specification framework for Java and Groovy applications.
>
> What makes it stand out from the crowd is its beautiful and highly expressive specification language.
>
> Thanks to its JUnit runner, Spock is compatible with most IDEs, build tools, and continuous integration servers.
>
> Spock is inspired from JUnit, RSpec, jMock, Mockito, Groovy, Scala, Vulcans, and other fascinating life forms.

<br>

[Spock Framework Reference Documentation](http://spockframework.org/spock/docs/1.2/all_in_one.html)

### Advantage

- Specifications as Documentation (BDD); `given-when-then` ; generating-report.

- less code, more readable, some awesome feature:

  - blocks, (no)thrown, '==',
  - where, table, database,
  - with, verifyAll
  - more fluent Mock syntax

- support `JUnit`, it's suppot all `@Rule`s in JUnit.

### Comparision

- JUnit, testNG
- Cucumber(Gherkin). **"one file solution"**
- Java BDD framework, like: [JBehave](https://jbehave.org/)
- RSpec [spock/issues/106](https://github.com/spockframework/spock/issues/106)

### Spock Quick Start

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

---

### Usage Example

[Comparision to JUnit](http://spockframework.org/spock/docs/1.2/all_in_one.html#_comparison_to_junit)

#### Blocks

![Blocks](http://spockframework.org/spock/docs/1.2/images/Blocks2Phases.png)
```groovy
def "events are published to all subscribers"() {
  given:
  def subscriber1 = Mock(Subscriber)
  def subscriber2 = Mock(Subscriber)
  def publisher = new Publisher()
  publisher.add(subscriber1)
  publisher.add(subscriber2)

  when:
  publisher.fire("event")

  then:
  1 * subscriber1.receive("event")
  1 * subscriber2.receive("event")
}
```

#### Conditions

Within the `then` and `expect` blocks, assertions are implicit

#### some Shorthand (syntactic sugar in Groovy)

 The test code wrote by Spock are more concise and easier to read. 
 There's less clutter, boilerplate code, and your test cases can be structured better.

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

see also:
[groovy-lang.org/syntax.html](http://groovy-lang.org/syntax.html)  
[some demos](https://learnxinyminutes.com/docs/groovy/)

#### Exception Conditions

`thrown, notThrown`
```groovy
when:
stack.pop()

then:
thrown(EmptyStackException)
stack.empty
```

#### Support Java Testing Lib

including JUnit, Mockito, JAssert, Hamcrest, etc.

#### Data Tables

A awesome feature!

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

---

### Migration from Java to Groovy

#### java lambda expression vs groovy closure

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

#### [] vs {}

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

#### groovy handy no-setter (or no-getter) syntax

declare a field without access modifier



#### Precaution

- IntelliJ-IDEA: must set test directory otherwise it reports `Empty Test Suite` error.

<br>

- JUnit `@ClassRule`
  ```groovy
  // it works:
  @ClassRule
  @Shared
  public MyRule rule = new MyRule()

  // doesn't work (with static modifier):
  @ClassRule
  @Shared
  public static MyRule rule = new MyRule()
  ```
