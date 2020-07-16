title: Java Integer的自动装箱和缓存机制 -- 由一套面试题引出的
author: chenxi
tags:
  - java
categories: []
date: 2018-06-26 14:35:00
---
### 背景
最近遇到一道Java面试题, 感觉很有意思, 和大家分享一下.   
是远程在线做题的, 可以使用自己的IDE. 

### 题目
```
    private static void swap(Integer a, Integer b) {
    }

    public static void main(String[] args) {
        Integer a = 1;
        Integer b = 2;
        System.out.println("before:a=" + a + ",b=" + b);
        swap(a, b);//实现此swap函数；交换位置
        System.out.println("after:a=" + a + ",b=" + b);
    }
```

### 实际过程中(错误的)解法 
#### 分析: 
第一感觉还是比较简单的, 由于Integer是不可变对象, 所以利用反射修改他们内部维护的那个'value'字段. 

#### 代码如下:
```
    private static void swap(Integer a, Integer b) {
        int valueA = b;
        int valueB = a;

        try {
            final String innerFieldName = "value";
            Field field = Integer.class.getDeclaredField(innerFieldName);
            field.setAccessible(true);

            modifyViaReflection(a, valueA, field);
            modifyViaReflection(b, valueB, field);
        } catch (NoSuchFieldException | IllegalAccessException e) {
            throw new RuntimeException(e);
        }
    }

    private static void modifyViaReflection(Integer obj, int val, Field field) throws IllegalAccessException {
        field.set(obj, val);
    }
```

#### 运行结果: a可以修改成功而b不可以. 

#### 调试
当发现下面的调试输出结果时我是有点崩溃的:
```
    // 传入的val为1
    private static void modifyViaReflection(Integer obj, int val, Field field) throws IllegalAccessException {
        log.debug("before obj = {}", obj); // print 2
        log.debug("val = " + val); // print 1
        field.set(obj, val);
        log.debug("after obj = {}", obj); // print 2 !!! (使用反射修改字段值失败)
    }
```
当时调了好久, 明明传入的val是1, 为什么会修改不成功呢?
+ 第一次修改是成功的, 第二次就不行, 调换了a和b的次序, 同样如此;
+ 甚至怀疑是JDK版本的原因, 还从JDK8切换到了JDK6: 无果;

面试完之后, 继续探索: 
```
    // 传入的val为1
    private static void modifyViaReflection(Integer obj, int val, Field field) throws IllegalAccessException {
        log.debug("before obj = {}", obj); // print 2
        log.debug("val = " + val); // print 1
        log.debug("val = {}", val); // print 2 !!!  (加了这一行调试语句后后发现了新大陆)
        field.set(obj, val);
        log.debug("after obj = {}", obj); // print 2 !!! (使用反射修改字段值失败)
    }
```

### 正确的解法
```
    private static void modifyViaReflection(Integer obj, int val, Field field) throws IllegalAccessException {
        log.debug("val = " + val); // when val == 1, print 1
        log.debug("val = {}", val); // when val ==1, print 2 (很奇怪吧, 看下面的解释)

        /*
        如果按照下面那行"错误写法"那样写的话, 当入参val为1时, 它会被解糖为"Integer.value(1)",
        由于Integer的cache机制, "Integer.value(1)"和a会是同一个对象, 指向的都是"Integer Cache"中的那个对象.
        然而这个对象的value字段已经被我们改成2了!
        所以就会出现明明传入的val为1, 但是调用完field#set方法之后, obj还是2的奇怪现象.
        */
        // field.set(obj, val); // 错误的写法! 会被解糖为: field.set(obj, Integer.valueOf(val));
        field.set(obj, new Integer(val));
    }
```


### 复盘
- 心态: 限时一小时, 总共两题, 第一题就卡主了, 有点紧张, 有点慌了. 
- 知识储备: Integer的自动拆装箱, 前258位的缓存机制, 这些其实都懂, 但是做题的时候没有把这两个联系到一起; 
- 调试原则: 遇到问题需要最先怀疑还是自己写的代码, 其次怀疑编译器, 操作系统, 社会环境之类的问题; 
    比如当时应该重点关注`field.set(obj, val);`这句代码, set方法第二个参数是Object, 不是int, 是会发生自动装箱的, 
    当时要是能意识到这个就能很快定位到问题了. 

