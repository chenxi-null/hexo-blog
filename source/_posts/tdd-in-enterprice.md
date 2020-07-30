title: 'TDD 在企业开发中的实践'
author: chenxi
tags: 
  - tdd
categories: 
  - tdd
date: 2019-04-04 15:30:00
---

## 前言

- Why  
    我们为什么要花额外的时间写测试，自动化测试可以给我们带来什么好处?

- How   
    分享测试方法论和一些测试实践, 在企业级项目中如何做好测试, 网上多是 Hello World 级别的测试 Demo， It's not enough! 


## 为什么写测试？

为什么花这么大精力写测试，需要利益驱动

测试用例扮演的角色：
1. ⛑️保护网
2. 💡引路人

### 1. 提高项目质量

#### 可靠性交付
代码质量保障  

自动化回归测试, 第一时间发现问题

对于黑盒测试和接口测试很难覆盖到的地方，可以进行细粒度/模块化的测试，我们可以对系统进行拆分, 针对某个模块进行测试

#### 提高代码质量
测试是重构的前提: "测试保护网" 让人放心重构 —— 重构是避免代码逐步腐化的必要手段

#### 测试即文档

测试是**可执行**的文档
**doc.txt vs doc.exe**

并且，采用 BDD 风格写的测试用例的可读性更强，代码的可维护性更好

#### 提高开发效率
降低代码维护成本 (降低自己在项目中的地位🐶) , 提高后期开发迭代效率


### 2. 自我提升

#### 测试驱动设计
写出可测试性的代码, 提升模块化设计的能力，思考功能的边界, 模块的松耦合, 加深对项目的理解

#### 测试驱动成长
- 测试是必备技能，是持续集成的基础，几乎所有讲敏捷开发的书都会提到 TDD
- 加深对各类技术的理解, 强迫你打开盒子，研究原理, e.g. 加深对 Spring 的理解


---

如何更好地写测试?

## TDD 开发模式

> TDD 有广义和狭义之分，常说的是狭义的 TDD，也就是 UTDD（Unit Test Driven Development）。
广义的 TDD 是 ATDD（Acceptance Test Driven Development），包括 BDD（Behavior Driven Development）和 Consumer-Driven Contracts Development 等。
> 
> TDD(单元测试驱动开发) 是敏捷开发中的一项核心实践和技术，也是一种设计方法论。
TDD的原理是在开发功能代码之前，先编写单元测试用例代码，测试代码确定需要编写什么产品代码。
TDD 是 XP（Extreme Programming）的核心实践。它的主要推动者是 Kent Beck。 

### Why
- 提前澄清需求, 明确流程，
- 测试不再成为一种负担, 而是设计的一部分
- 帮助开发人员做更好设计(代码职责更明确，代码可测试性更强)
- 快速反馈

### How
![the process of TDD](https://upload-images.jianshu.io/upload_images/279826-49f2f708aefb567f)

### Steps
- red: 明确意图; 保证测试代码抛出预期的错误（对测试的测试）
- green
    - 快速实现
    - 实现真正的产品代码
- refactor

### Principle 
- 提前设计, 划分好任务的粒度, 小步走
- 严格按照"先测试再实现"的顺序

---

介绍完 TDD，我们来进行测试方法论及最佳实践的探讨，最终目标：
- 测试的完备性：如何更好地保护代码
- 开发效率：通过最佳实践、工具选择等减轻写测试的负担
- 测试代码可读性和可维护性


## 测试方法论:

### 分类原则
优先按“主题” (系统模块, Feature, Bug) 分类, 其次才是 Class


### 集成测试和单元测试的关系
举个例子，比如要测试这个方法：
```java
public void makeAndDeliverCake(FlavorEnum flavor) {
    // 耗时短，测试用例复杂
    Cake cake = makeCake(flavor);

    // 耗时长，测试用例简单
    deliverCake(cake);
}
```

test `makeCake`: 
```groovy
def "succeed to makeCake"() {
    when:
    Cake actualCake = makeCake(falvor)
    then:
    expectedCake == actualCake

    where:
    falovr | expectedCake
    apple  | appleCake
    banana | bananaCake
    orange | orangeCake
}

def "failed to makeCake: 库存不足"() {
    when:
    Cake actualCake = makeCake(falvor)
    then:
    def e = thrown(IllegalArgumentException)
    e.message == '库存不足'

    where:
    falovr << [mango, grape]  
}

def "failed to makeCake: 暂不支持这种口味"() {
    when:
    Cake actualCake = makeCake(falvor)
    then:
    def e = thrown(IllegalArgumentException)
    e.message == '暂不支持这种口味'

    where:
    falovr << [dog, cat]  
}
```

test `makeAndDeliverCake`: 
```groovy
def "test makeAndDeliverCake"() {
    given:
    // mock cakeShop & prepare some data

    when:
    makeAndDeliverCake(apple)

    then:
    new PollingConditions().within(20, { cakeShop.reveive(appCake)})
}
```

单元测试里进行复杂的用例测试; 
集成测试只做简单的"连通性"测试

1. `CakeDistributionTest`: only test `makeCake`  
2. `CakeDistributionIntegrationTest`: test `makeAndDeliverCake`  

如果前期就写复杂的集成测试: 违背小步走原则，测试太耗时，影响开发节奏

### 测试即文档
测试用例的可读性!

### 把测试代码当成功能代码来写
提取重复代码: 测试基类, 工具类  
重复配置, 重复的 setup 和 cleanup 工具
测试架构的设计

## 测试场景及实践:

### 善用构建工具🔧
e.g. 使用 maven pom 文件中的 `profile` 标签进行多版本测试，比如我们要测试程序在不同 spring 版本下是否运行正常：
```xml
<profiles>
    <profile>
        <id>spring5</id>
        <dependencyManagement>
            <dependencies>
                <dependency>
                    <groupId>org.springframework</groupId>
                    <artifactId>spring-core</artifactId>
                    <version>5.0.10.RELEASE</version>
                </dependency>
            </dependencies>
        </dependencyManagement>
    </profile>
</profiles>
```


### Spring 容器 🍃

### 外部系统调用 
DB, Redis, ZK, MQ, Http-Server

例如 mock 一个 http server:
```groovy
class WireMockTest extends Specification {

    @Rule
    WireMockRule wireMockRule = new WireMockRule(18080)

    def wireMock = new WireMockGroovy(18080)

    def "WireMockRule"() {
        given:
        wireMockRule.stubFor(any(urlEqualTo("/some/thing?k1=v1"))
                .willReturn(aResponse()
                        .withHeader("Content-Type", "text/plain")
                        .withBody("Hello world!")));

        when:
        def res = org.apache.http.client.fluent.Request.Post("http://localhost:18080/some/thing?k1=v1")
                .execute().returnContent().asString()

        then:
        // assert result ...
        and:
        verify(1, postRequestedFor(urlEqualTo("/some/thing?k1=v1")));
    }

    // http://wiremock.org/docs/stubbing/
    def "WireMock (mock http server) | groovy style"() {
        given:
        wireMock.stub {
            request {
                method "GET"
                url "/book"
            }
            response {
                status 200
                body """[
                      {"title": "Book 1", "isbn": "4711"},
                      {"title": "Book 2", "isbn": "4712"}
                    ]
                 """
                headers { "Content-Type" "application/json" }
            }
        }

        when: "we invoke the REST client to find all books"
        def res = request()

        then: "we expect two books to be found"
        // assert result ....
        and: "the mock to be invoked exactly once"
        1 == wireMock.count {
            method "GET"
            url "/book"
        }
    }
}
```

### 异步场景
- Java: [Awaitility](https://github.com/awaitility/awaitility)
- Spock: `PollingConditions` 

### Interaction
- Mock & Stub & Spy [link](http://spockframework.org/spock/docs/1.2/all_in_one.html#_interaction_based_testing)
- Mock 静态方法: `PowerMock`
- JUnit Rule `SystemOutRule`
- 自己实现

### 其他场景 
[flaky-test](https://docs.qameta.io/allure/#_flaky_tests)
- 被测试的事件不稳定, 有一定概率失败
- 单独运行时正常，一起运行时失败，需要做好对象的清理工作


## 测试工具:

### 测试框架：
- [Spock](http://spockframework.org/spock/docs/1.2/all_in_one.html#_spock_primer)
- [JUnit 5](https://junit.org/junit5/docs/current/user-guide/#writing-tests-assertions)       

工具目的：提升写测试的效率；让测试更易读    

### 测试执行报告:
- [Allure](http://allure.qatools.ru/): `mvn allure:server` 

![命令行输出的测试结果不好阅读](https://user-gold-cdn.xitu.io/2019/10/25/16e00748a5dad9e7?w=2848&h=1606&f=png&s=451667)


![Allure 展示执行结果](https://user-gold-cdn.xitu.io/2019/10/25/16e00756799f986f?w=2878&h=1544&f=png&s=644674)


![Allure 展示测试用例耗时](https://user-gold-cdn.xitu.io/2019/10/25/16e0076c87e80345?w=2878&h=1558&f=png&s=225975)

Allure 还可以和 Jenkins 集成，查看测试执行结果的趋势变化


