---
title: 为cucumber 引入DI -- Cucumber Spring原理浅析
cover: imgs/theworst.jpg
date: 2019-12-17 18:41:14
tags:
desc: 一个越来越大型的多人合作项目不能没有依赖注入
---

# Cucumber-spring setup.

## Cucumber

### The mechanism cucumber glue.

Cucumber scan its glue classes. Find classes with method annotated with cucumber.api annotations like @When or @Before, @After, add them to ObjectFactory.

```java
package cucumber.api.java;

/**
 * Minimal facade for Dependency Injection containers
 */
public interface ObjectFactory {

    /**
     * Instantiate glue code <b>before</b> scenario execution. Called once per scenario.
     */
    void start();

    /**
     * Dispose glue code <b>after</b> scenario execution. Called once per scenario.
     */
    void stop();

    /**
     * Collects glue classes in the classpath. Called once on init.
     *
     * @param glueClass Glue class containing cucumber.api annotations (Before, Given, When, ...)
     * @return true if stepdefs and hooks in this class should be used, false if they should be ignored.
     */
    boolean addClass(Class<?> glueClass);

    /**
     * Provides the glue instances used to execute the current scenario. The instance can be prepared in
     * {@link #start()}.
     *
     * @param glueClass type of instance to be created.
     * @param <T>       type of Glue class
     * @return new Glue instance of type T
     */
    <T> T getInstance(Class<T> glueClass);
}
```

Cucumber use `getInstance` method to retrive the single instance of glue class in the factory.

Usually, objectFactory init an instance of for each glue class for each scenario. All glue classes will be recreated fresh for each scenario.

### Parallel

We use

```java
@DataProvider(parallel = true)
public Object[][] scenarios() {
    return super.scenarios();
}
```

Create 10 threads by default, with which invokes a run with one scenario.
ObjectFactory is thread-singleton. So glue instances will differ in each scenario.

## Spring

In this code practicing, we use ioc container of [spring](https://www.baeldung.com/inversion-control-and-dependency-injection-in-spring).

1. Without ioc, we should write code in a class to acquire for it's dependencies.
2. With ioc, we write code to acquire dependencies from ioc container(dependency lookup),
   or just declare the dependencies and make the container to inject it to you(dependency injection). See [here](https://stackoverflow.com/questions/28039232/what-is-the-difference-between-dependency-injection-and-dependency-look-up) for futher information.

## Cucumber & Spring

### Why cucumber-spring?

Two choices:

1. pico: https://github.com/cucumber/cucumber-jvm/tree/master/picocontainer

   Available, with a disadvantage.

   > All step classes and their dependencies will be recreated fresh for each scenario, even if the scenario in question does not use any steps from that particular class.

   Also, it supports constructor parameters injection.

2. spring: https://github.com/cucumber/cucumber-jvm/tree/master/spring

   Perfect. Also spring's ecosystem is good.

### Mechanism:

cucumber-spring inherit ObjectFactory, and load glue classes to it's `ConfigurableListableBeanFactory`. And all the beans are controlled by spring's bean factory.

### Configure it:

#### Dependencies:

The recommended version is:

```
        <spring.version>2.0.4.RELEASE</spring.version>
        <cucumber.version>4.2.2</cucumber.version>
```

with dependency:

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <dependency>
            <groupId>io.cucumber</groupId>
            <artifactId>cucumber-spring</artifactId>
            <version>${cucumber.version}</version>
            <scope>test</scope>
        </dependency>
```

#### Configure:

For junit, doc [here](https://github.com/cucumber/cucumber-jvm/tree/master/spring) is enough.
But as for CucumberTestngRunner, things get different.

Due to cucumber-spring's mechanism, we should write one and only one contextConfiguration for spring config.

```java
import org.springframework.test.context.ContextConfiguration;

/**
 * This is the only glue class with spring config.
 */
@ContextConfiguration(classes = Config.class)
public class GlueSpringConfig {
}

@Configuration
@ComponentScan("com.**.**....")
public class Config {
}
```

Then, we can use spring's features to make dependencies managment simple.

#### Scope.

A scope defines how spring register and retrive it's dependencies.
Through scope, you can customize the beans' lifecycle.

In this case, we need to define our own customized scope, with abilities to control bean's lifecycle.

> https://stackoverflow.com/questions/50477894/spring-custom-scope-lifecycle-bean-termination

> https://howtodoinjava.com/spring-core/spring-bean-life-cycle/

> http://iryndin.net/post/spring_beanpostprocessors/

For more details, see [here](../cucumber-spring-case-scope/)
