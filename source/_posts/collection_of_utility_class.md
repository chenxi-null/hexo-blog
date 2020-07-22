title: 提高代码简洁度的工具类合集
author: chenxi
tags:
  - clean-code
categories: 
  - clean-code
date: 2020-02-24 15:40:00
---

日常开发中整理的一些开箱即用的开源工具类，帮助减少冗余代码和项目中的重复轮子，主要提供商包括 `Spring`, `Apache common-lang3`, `Lombok`, `Guava`。

# Assert
```java
// original code
if (input == null) {
    throw new IllegalArgument("given argument must not be null");
}

if (num != 1) {
    throw new IllegalArgument("given number is not 1");
}

// clean code, powered by spring-core
Assert.notNull(input, "input is null");

Assert.isTrue(num == 1, "given number is not 1");
```

# Safe equals
```java
// not null-safe
!a.equals(b)

// better, powered by JDK-7
!Objects.equals(a, b)
```

# StringUtils
```java
// common-lang3
StringUtils.isEmpty(str);
StringUtils.isBlank(str);
```

# BooleanUtils
```java
Boolean b;

// original code
if (b == null || b == false) {
    //...
}

// clean code, powered by common-lang3
if (BooleanUtils.isNotTrue) {
    //...
}
```

# defaultIfNull
```java
// defaultIfNull common-lang3
int expiration = ObjectUtils.defaultIfNull(
                Utils.getConfigValueAsInt(PROPERTY_EXPIRATION),
                DEFAULT_EXPIRATION);
```

# Ignore Checked-Exception
Lombok `@SneakyThrow`
```java
public static void normalMethodWithThrowsIdentifier() throws IOException {
    throw new IOException();
}

@SneakyThrows
public static void methodWithSneakyThrowsAnnotation() {
    throw new IOException();
}
```

# Builder Pattern with Lombok

[lombok Builder](https://projectlombok.org/features/Builder)

## Required arguments with a lombok @Builder
```java
import lombok.Builder;
import lombok.ToString;

@Builder(builderMethodName = "hiddenBuilder")
@ToString
public class Person {

    private String name;
    private String surname;

    public static PersonBuilder builder(String name) {
        return hiddenBuilder().name(name);
    }
}
```
Copied from [SOF](https://stackoverflow.com/questions/29885428/required-arguments-with-a-lombok-builder/30867286#30867286)

   
# guava   
TODO

# retry strategy
TODO