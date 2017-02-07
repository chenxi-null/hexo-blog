title: 'java枚举类初探: 反编译&语法糖'
author: chenxi
tags: []
categories: []
date: 2016-09-04 20:55:00
---


最近接触了“**语法糖”**这个概念，今天又看了一下**枚举类**的知识点，主要还是看它的用法，之前一直没有怎么用过java枚举类，看了李刚那本《疯狂java讲义》的枚举类章节，算是把它的用法弄明白了。

可是枚举类是一种“语法糖”，也就是说**只有编译器知道“enum”关键字，jvm是不知道的，字节码文件中没有枚举这一概念**。实际上，书中一开头就提到了在枚举出现前，程序员需要自己编程实现枚举（具体就不展开了），比较复杂，JDK1.5之后出现了enum枚举类（可以看做是一种特殊的java类）——作为一种“语法糖”让程序员更方便地实现枚举，对程序员更加友好、“甜蜜”了，哈哈。

我对java语法糖的理解是这样的：可以看做编译器先把原来的enum类编译成“正常的java类”，然后再以正常的方式编译成字节码，所以字节码当然不知道有“语法糖”的存在啦，不知道我的理解有没有错，应该是没有什么大问题，如果有的话还请各位大神指正。
为了更好的理解枚举类是怎么实现的，我先是按照书中对枚举类的定义写了一个对应于enum类的“正常的java类”，代码如下：

```
import java.lang.Enum;
/*
public enum TrueEnum {
	WHITE, BLACK;
}
*/
public class MyEnum extends Enum { // 这样写肯定是通不过编译的，因为编译器不允许我们的类显示的继承Enum类
	
	private MyEnum() {}
	
	public static final MyEnum e1 = new MyEnum("WHITE", 0);
	public static final MyEnum e2 = new MyEnum("BLACK", 1);
	
	public MyEnum[] values() {
		return new MyEnum[]{e1, e2}; // 这个地方有问题，等下指出
	}
}
```
后来想反编译一下TrueEum.class(这个是enum类），和自己写的代码对比一下看对不对，就下了一个反编译程序jd-gui，没想到这个程序太高级了，直接反编译成enum类了（囧），又不知道怎么弄成“低级”的，就直接上网搜了别人的博客，看他们贴出的反编译代码，发现自己的猜想还是基本正确的，除了values()方法，正确的实现应该是返回数组的一个浅拷贝（return array.clone()），而不是像我这样简单的返回一个数组引用，后来想想：如果调用者搞破坏怎么办？比如通过数据引用把某个数据元素换成其他值不就破坏了原对象的封装性了吗。看来自己还是too young。

看了两三篇博客，这篇写的不错：[Java 枚举源码分析](http://blog.jrwang.me/2016/java-enum/)主要是排版看起来比较舒服，分析的也比较全面。

不过还有一点值得注意的是，这篇文章没有指出enum类实现接口和添加抽象方法这两种情况，这样的话，枚举值就不是作为枚举类的实例了，而是作为枚举类的匿名子类的实现类。
也就是说枚举值要么是Enum的直接子类要么是Enum的“孙子类”（这个词是我发明的，理解意思就好，蛤蛤），知道这点的话，下面这段Enum的源码就很好理解了：

```
// 得到枚举常量所属枚举类型的Class对象  
public final Class<E> getDeclaringClass() {  
    Class clazz = getClass();  
    Class zuper = clazz.getSuperclass();  
    return (zuper == Enum.class) ? clazz : zuper;  
}
```


这是我基于刚才提到的那篇博客做的一点补充，做了一点微薄的贡献，谢谢大家。
第一次发表博客，就先这样吧，献丑了，欢迎各位朋友批评指正。