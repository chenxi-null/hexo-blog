title: 编程随想录 - 代码注释
author: chenxi
tags:
  - clean-code
categories: []
date: 2019-07-02 16:00:00
---

关于代码注释应该有不少书籍讨论过了, 比如《Clean Code》, 这里我还是想结合一些个人经验, 简单表达一些关于代码注释最佳实践的看法.
关于注释我们在项目中可能会遇到如下几种情况:

## 无用的注释
![](https://user-gold-cdn.xitu.io/2019/6/25/16b8f1f0a34f7c67?w=1080&h=1119&f=jpeg&s=110638)

《Clean Code》里列举了很多种Bad Comments, 其中就包括Noise Comments:
> Sometimes you see comments that are nothing but noise. They restate the obvious and provide no new information.

```java
/** The day of the month. */
private int dayOfMonth;
```


《Clean Code》也吐糟了 Java Doc 注释的滥用, 作者并不认同每个字段和方法都必须写注释, 因为类似于下面的这种注释是毫无意义的, 在代码之外并没有提供更多的信息. 代码的简单“镜像”? 意义真的不是很大.

但是很多知名的开源框架还是奉行奉行每个字段和方法都必须有注释的原则, 如`Spring`, `Apache Common`, 这样做是考虑到生成 Java Doc 后便于大家阅读文档, 也有它存在的合理性吧. 

总之, 注释的原则是让人更好地理解代码, 如果做不到这点, 就不是好的注释.

## 必要的注释
贴心的注释, 解释代码的动机, 方便后来者阅读, 不需要大家去猜代码, 程序员的沟通原则就是简单直接高效, 编码不是谈恋爱, 有话直说不要靠猜. 

告诉你为什么要加`@SuppressWarnings`注解:
![](https://user-gold-cdn.xitu.io/2019/7/2/16bb275ead0e0bc0?w=1664&h=576&f=png&s=125259)

告诉你这段代码的意图:
![](https://user-gold-cdn.xitu.io/2019/7/2/16bb281ae2597b11?w=1796&h=1312&f=png&s=222114)

## 几乎不写注释
我个人就很鄙视这种情况, 包括git commit messeage写得非常很少很简单的情况, 明明修改了很多地方, 却只简单地几个字带过, 这种系统很难维护, 代码意图基本靠猜, 会让接管代码的人很崩溃.   

我称这种人为时间杀手或者偷时间的人, 这样的代码会让维护者花更多的时间猜测代码的意图, 甚至会让维护者被被过分简单的注释所误导而走不少弯路, 成功地浪费了维护者的工作效率.


### 不写注释的一些原因:

- **没有写注释的意识**: 技术短视, 没有意识到自己的代码写出来不仅仅是能完成功能的, 还是要给人阅读和维护的. 

- **懒得写注释**: 缺乏协作精神，只图自己省事, 哪管他人维护不易

- **用"好的代码是不需要注释的"来为自己不写注释的行为开脱**:  
	确实好的代码本身很多时候可以清晰地表达出自己在做什么, 但有时却无法表达自己这样做的原因, 这时候注释的作用就体现出来了.   
    记住: 在表意方面, 代码不是万能的. 代码不够, 注释来凑, 不是什么丢人的事情.  
    不要写坏注释不等于不写注释