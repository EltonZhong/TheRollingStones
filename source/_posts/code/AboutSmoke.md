---
title: 'About Smoke: 防御性编程与契约式编程'
desc: 🔑A declarative Java validation framework for dtos, beans, and objects.. Use annotations to validate your class.
cover: /imgs/11.png
tags:
  - Code
categories:
  - Code
date: 2018-12-23 21:10:27
---
# Smoke 与契约式、防御性编程
Smoke 是我写的一个给Dto做Validation 的小工具。 起因很简单,之前我在写Http client的时候, 觉得在很多地方都要对同一个对象进行验证比较麻烦(防御性编程)。 有个注解形式声明, 在最初的地方对对象进行一次验证, 就可以在编码时假设数据是OK的了。注解形式的validation是声明式的, 减少了程序员对数据是否符合规范的恐惧。

>想想, 如果这个东西是命令式的, 即你在第一次拿到这个对象的时候, 对它进行了基本的验证, 但是到后面的dal 层, service 层, 往往你还会对这个数据进行再一次的重复的验证, 才去执行自己的业务逻辑。


防御性编程给我们带来的好处是系统的错误尽早发现处理, 而不是在很后面抛出一个NPE, 到时候造成的损失可能是难以估量的。 而这又带来一个重复代码的问题, 代码到处验证, 十分臃肿。

所以很多人采用约定式的编程, 即我假设你传过来的数据是符合我的规范的, 出事怪你。 我个人觉得这个简直太可怕了, 就算自己写的代码, 我自己也难以信任, 更何况是别人的？ 所以, 有个这么一个annotation声明在类上面, 就像是一个契约文档。 我去调用类的方法, 可以大概知道哪些field是安全的。 这个是对契约性编程的一种补充。

当然, 这远远不能达到 `保证数据一定是安全的` 需求。 那我们如何做呢？
- 保证Dto(或者bean)是immutable 的
- 在每一次对类修改, 都进行验证

以上两个条件满足其一即可。

当然上面都是我个人的一些看法, 看了一些防御性编程与进攻性编程, 觉得他们有部分都没有讲清楚(或者是小弟愚昧看不明白),所以我强行理解了一波, 给出的感触可能是这样的。 

# 写Smoke的感触 

Smoke 不是一个玩具。 它是我利用业余时间写的, 并在我日常工作上已经有应用的小工具(或者说框架)。

当初因为validation重复代码太多的问题, 上网找了一些解决方案, 比如[hibernate的validator](https://github.com/hibernate/hibernate-validator), 还有命令式的, 最终决定针对自己需求写一个。

刚开始信心满满,发布到jcenter, 自己想了好几个feature, 虽然用不到也实现了, 然后后面发现实在无人问津,就没再管了。 留了个VRule没有实现, 差不多一个月过去了, 今天还是决定把它弄完, 虽然没有星星, 做事还是要有始有终。 (`谁叫我feature没做好先把readme写好了。。承诺过的feature, 浪费周末也要搞完。`)

当然除了在我自己的工作中用到之外也不是全无收获, 搞了个travis自动test、publish到jcenter, 还踩了一对java库发布的坑(fucking java)。之前欠了好多技术债,一个一个慢慢还。

下面是Smoke的文档~


# Smoke [![Build Status](https://travis-ci.org/EltonZhong/smoke.svg?branch=master)](https://travis-ci.org/EltonZhong/smoke)
A declarative validator framework for dtos, beans, or just objects.

## Install [ ![Download](https://api.bintray.com/packages/ez/tools/smoke/images/download.svg) ](https://bintray.com/ez/tools/smoke/_latestVersion)
Get the latest version at [Jcenter](https://bintray.com/ez/tools/smoke/_latestVersion)
```grovvy
// For gradle
repositories {
    jcenter()
}

dependencies {
    compile 'com.ez.tools:smoke:5.0.0'
}
```

## Basic Usage
```java
class UserDto {
    @VString(shouldBe = {"John1", "John2"})
    @VNotNull
    String name = null;
}

// Will throw java.lang.IllegalStateException when field name is null
com.ez.tools.validator.Smoke.validate(new UserDto())
```

Go to package `com.ez.tools.validator.annotations` to find other features.


## Advance features
Smoke 5.0 focused on the following topics:
- Field validation, with built-in constraints: @VString, @VInt, @VNotNull.
- `Getter method` validation, contraints can be put on method with no arguments, when validating, the getter method will be called, and the value returned will be validated.
- @VRule on class level, validate this kind of object using additional rule.

### VString
The regex feature has been released! Use regex:
```java
class UserDto {

    @VString(shouldMatch = {Regexps.EMAIL})
    String email;

    // Custom regex: Name should match all values in should match
    // And name should not match any values in should not match
    @VString(shouldMatch = {".+", "..."})
    @VString(shouldNotMatch = {"..."})
    String name;

    @VString(...)
    public void get***() {
    }
}
```

### VRule
Use additional rules to validate your dto.

Add additional rule:
```java
com.ez.tools.validator.Smoke.validate(new UserDto(), IRule...rules)
```
or:
```java
/**
* Add your rule class
**/
@VRule(com.ez.tools.validator.core.rules.AllFieldsShouldNotBeNull.class)
class UserDto {}

// Will validate using the rule specified in VRule value.
com.ez.tools.validator.Smoke.validate(new UserDto())
```

Also you can write Additional rules:

```java
class YourRule implements IRule {
    @Override
    public void validate(Object o) {
        if (!o instanceof UserDto) {
            throw new IllegalArgumentException();
        }
        UserDto dto = (UserDto) o;
        if(dto.name == "elton" && dto.age > 100) {
            throw new Exception("Elton is dead after at 99 years old.")
        }
    }
}

// Use it in your common interface:
@VRule(YourRule.class)
interface SatisfyYourRule {}

// Implements the interface:
class YourDto implements SatisfyYourRule {}

// When validate your dto, the rule will be invoked. And as rule above, Exception("Elton is dead after at 99 years old.") will be thrown
Smoke.validate(new UserDto("elton", 100))
```

### VRecursive
Wanna validate details of a field with smoke? Try VRecursive:
```java
class UserDto {
    @VRecursive
    AddressDto address = new AddressDto(this);
}

@VRule(AllFieldsShouldNotBeNull.class)
class AddressDto {
    public AddressDto(UserDto dto) {
        user = dto;
    }

    String a = null;

     @VString(shouldMatch = {Regexps.EMAIL})
    String getEmail() {
        return "123@qq.com";
    }

    @VRecursive
    UserDto user = null;
}

// Will validate userDto's address, with respect to rules and other annotation con
// And don't worry that it might cause stackoverflow exception:
// smoke will not validate same object twice.
com.ez.tools.validator.Smoke.validate(new UserDto())
```

***`Remeber! When field is null, smoke will ignore this field's validation.`***

***`You can use VNotNull to handle this situation`***
```java
class UserDto {
    @VString(shouldMatch = "^elton.+")
    String name = null;
}

// Will not validate when name is null. If you want to, add @VNotNull annotation constraints.
com.ez.tools.validator.Smoke.validate(new UserDto())
```
