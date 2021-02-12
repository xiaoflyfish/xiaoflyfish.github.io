---
title: Bean生命周期
categories: 
- Spring
---

1.实例化Bean对象，这个时候Bean的对象是非常低级的，基本不能够被我们使用，因为连最基本的属性都没有设置，可以理解为连Autowired注解都是没有解析的；

2.填充属性，当做完这一步，Bean对象基本是完整的了，可以理解为Autowired注解已经解析完毕，依赖注入完成了；

3.如果Bean实现了BeanNameAware接口，则调用setBeanName方法；

4.如果Bean实现了BeanClassLoaderAware接口，则调用setBeanClassLoader方法；

5.如果Bean实现了BeanFactoryAware接口，则调用setBeanFactory方法；

6.调用BeanPostProcessor的postProcessBeforeInitialization方法；

7.如果Bean实现了InitializingBean接口，调用afterPropertiesSet方法；

8.如果Bean定义了init-method方法，则调用Bean的init-method方法；

9.调用BeanPostProcessor的postProcessAfterInitialization方法；当进行到这一步，Bean已经被准备就绪了，一直停留在应用的上下文中，直到被销毁；

10.如果应用的上下文被销毁了，如果Bean实现了DisposableBean接口，则调用destroy方法，如果Bean定义了destory-method声明了销毁方法也会被调用。

