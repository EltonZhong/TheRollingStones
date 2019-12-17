---
title: 如何引入 Spring 的 scope 到 Cucumber 项目中?
cover: imgs/lips.png
date: 2019-12-17 18:41:59
tags:
desc:
---

# Spring scope: Case scope.

A scope defines how spring register and retrive it's dependencies.
Through scope, you can customize the beans' lifecycle.

With links below, you will be familar with scope and spring di mechanism.

> https://www.baeldung.com/spring-custom-scope

> https://stackoverflow.com/questions/50477894/spring-custom-scope-lifecycle-bean-termination

> https://howtodoinjava.com/spring-core/spring-bean-life-cycle/

> http://iryndin.net/post/spring_beanpostprocessors/

## Spring scope introduction.

### Usage

```java
@Component
@Scope("singleton")
class ABean {

}

// Or in config:
@Configuration
@ComponentScan("com.**.**....")
public class Config {

    @Bean
    @Scope("prototype")
    public void getABean() {
        return new ABean();
    }
}
```

Available and common scopes are here:

```java
	/**
	 * Scope identifier for the standard singleton scope: "singleton".
	 * Custom scopes can be added via {@code registerScope}.
	 * @see #registerScope
	 */
	String SCOPE_SINGLETON = "singleton";

	/**
	 * Scope identifier for the standard prototype scope: "prototype".
	 * Custom scopes can be added via {@code registerScope}.
	 * @see #registerScope
	 */
	String SCOPE_PROTOTYPE = "prototype";

    /**
	 * Scope identifier for request scope: "request".
	 * Supported in addition to the standard scopes "singleton" and "prototype".
	 */
	String SCOPE_REQUEST = "request";

	/**
	 * Scope identifier for session scope: "session".
	 * Supported in addition to the standard scopes "singleton" and "prototype".
	 */
	String SCOPE_SESSION = "session";
```

### How they work

See `org.springframework.beans.factory.config.Scope`

```java
public interface Scope {
    /**
	 * Return the object with the given name from the underlying scope,
	 * {@link org.springframework.beans.factory.ObjectFactory#getObject() creating it}
	 * if not found in the underlying storage mechanism.
	 * <p>This is the central operation of a Scope, and the only operation
	 * that is absolutely required.
	 * @param name the name of the object to retrieve
	 * @param objectFactory the {@link ObjectFactory} to use to create the scoped
	 * object if it is not present in the underlying storage mechanism
	 * @return the desired object (never {@code null})
	 * @throws IllegalStateException if the underlying scope is not currently active
	 */
	Object get(String name, ObjectFactory<?> objectFactory);

    /**
	 * Register a callback to be executed on destruction of the specified
	 * object in the scope (or at destruction of the entire scope, if the
	 * scope does not destroy individual objects but rather only terminates
	 * in its entirety).
	 * <p><b>Note: This is an optional operation.</b> This method will only
	 * be called for scoped beans with actual destruction configuration
	 * (DisposableBean, destroy-method, DestructionAwareBeanPostProcessor).
	 * Implementations should do their best to execute a given callback
	 * at the appropriate time. If such a callback is not supported by the
	 * underlying runtime environment at all, the callback <i>must be
	 * ignored and a corresponding warning should be logged</i>.
	 * <p>Note that 'destruction' refers to automatic destruction of
	 * the object as part of the scope's own lifecycle, not to the individual
	 * scoped object having been explicitly removed by the application.
	 * If a scoped object gets removed via this facade's {@link #remove(String)}
	 * method, any registered destruction callback should be removed as well,
	 * assuming that the removed object will be reused or manually destroyed.
	 * @param name the name of the object to execute the destruction callback for
	 * @param callback the destruction callback to be executed.
	 * Note that the passed-in Runnable will never throw an exception,
	 * so it can safely be executed without an enclosing try-catch block.
	 * Furthermore, the Runnable will usually be serializable, provided
	 * that its target object is serializable as well.
	 * @throws IllegalStateException if the underlying scope is not currently active
	 * @see org.springframework.beans.factory.DisposableBean
	 * @see org.springframework.beans.factory.support.AbstractBeanDefinition#getDestroyMethodName()
	 * @see DestructionAwareBeanPostProcessor
	 */
	void registerDestructionCallback(String name, Runnable callback);
}
```

So, in a simple way, we can define a `scopeHolder`, which store the instances, and remove the instances when scope destroyed.

## Customize Injection

With custom inject policy, we can reduce a lot of effort for wiring `android & ios page`.
http://iryndin.net/post/spring_beanpostprocessors/

So we can wirte steps like this:

```java
class ExampleSteps {

    /**
     * Automatically wire AndroidBaseSomePage or IosBaseSomePage to page.
    */
    @InjectPage
    private IBaseSomePage page;

    @When("")
    public void example() {
        page.doSomething();
    }
}
```

But still we need to handle mobileDevice properly, because when `ExampleSteps` inits, the page, which depends on mobileDevice will be wired.

So we can separate the appium session and MobileDevice creation:

```java
class MobileDevice {
    AppiumDriver driver;

    getAppiumDriver() {
        if (driver == null) {
            throw IllegalStateException("You should create appium session first.");
        }
        return driver;
    }
}
```

or we can lazy init the page.

```java
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        ReflectionUtils.doWithFields(bean.getClass(), field -> {
            if (field.getAnnotation(InjectPage.class) != null) {
                ReflectionUtils.makeAccessible(field);
                hooks.add(() -> field.set(bean, getPage(bean)));
            }
        });
        return bean;
    }


    @Before
    public void initSession() {
        createAppiumDriver();
        injectPages();
    }

    private void injectPages() {
        hooks.forEach(Hook::call)
    }

    interface Hook {
        void call();
    }
```
