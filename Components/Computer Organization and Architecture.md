[computer organisation](https://static001.geekbang.org/resource/image/12/ff/12bc980053ea355a201e2b529048e2ff.jpg?wh=3832*2540)

[linux os](https://static001.geekbang.org/resource/image/92/cb/92ec3d008c77bb66a148772d3c5ea9cb.png?wh=2160*1620)

[Spring注解bean加载生命周期时序图](https://static001.geekbang.org/resource/image/6f/8a/6ff70ab627711065bc17c54c001ef08a.png)

**Computer Organization and Architecture**： 計算機組成原理

```java
DefaultListableBeanFactory.class
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
    throws BeanCreationException {
    //省略非关键代码
  if (instanceWrapper == null) {
    instanceWrapper = createBeanInstance(beanName, mbd, args);
  }
  final Object bean = instanceWrapper.getWrappedInstance();

    //省略非关键代码
    Object exposedObject = bean;
    try {
       populateBean(beanName, mbd, instanceWrapper);
       exposedObject = initializeBean(beanName, exposedObject, mbd);
    }
    catch (Throwable ex) {
    //省略非关键代码
}
```

按执行顺序分别是第 5 行的 createBeanInstance，

第 12 行的 populateBean，

以及第 13 行的 initializeBean，

分别对应实例化 Bean， 	

注入 Bean 依赖，

以及初始化 Bean （例如执行 @PostConstruct 标记的方法 ）这三个功能，这也和上述时序图的流程相符。

如果有属性依赖注入，在注入bean依赖时会去实例化新的依赖bean

![堆栈的时序图](https://tva1.sinaimg.cn/large/e6c9d24egy1h39x2wa8noj20lb0fk77w.jpg)

```
//AbstractAutowireCapableBeanFactory.class#doGetBean
// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
//生成aop 动态代理
Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
```





而对于非 Spring Boot 程序，除了添加相关 AOP 依赖项外，我们还常常会使用 @EnableAspectJAutoProxy 来开启 AOP 功能。这个注解类引入（Import）AspectJAutoProxyRegistrar，它通过实现 ImportBeanDefinitionRegistrar 的接口方法来完成 AOP 相关 Bean 的准备工作。

## AOP动态代理源码

AnnotationAwareAspectJAutoProxyCreator

在完成原始 Bean 构建后的初始化 Bean（initializeBean）过程中。

![image-20220315165945529](https://tva1.sinaimg.cn/large/e6c9d24egy1h0ao9h1bqmj20kr0ah40r.jpg)

aop代理实现方式，

```java
// DefaultAopProxyFactory.class

public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}
```

CGlib

```java
# CglibAopProxy
			// fixedInterceptorMap only populated at this point, after getCallbacks call above
			enhancer.setCallbackFilter(new ProxyCallbackFilter(
					this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
			enhancer.setCallbackTypes(types);

			// Generate the proxy class and create a proxy instance.
			return createProxyClassAndInstance(enhancer, callbacks);
```

```java
# ObjenesisCglibAopProxy
protected Object createProxyClassAndInstance(Enhancer enhancer, Callback[] callbacks) {
		Class<?> proxyClass = enhancer.createClass();
		Object proxyInstance = null;

		if (objenesis.isWorthTrying()) {
			try {
				proxyInstance = objenesis.newInstance(proxyClass, enhancer.getUseCache());
			}
			catch (Throwable ex) {
				logger.debug("Unable to instantiate proxy using Objenesis, " +
						"falling back to regular proxy construction", ex);
			}
		}

		if (proxyInstance == null) {
			// Regular instantiation via default constructor...
			try {
				Constructor<?> ctor = (this.constructorArgs != null ?
						proxyClass.getDeclaredConstructor(this.constructorArgTypes) :
						proxyClass.getDeclaredConstructor());
				ReflectionUtils.makeAccessible(ctor);
				proxyInstance = (this.constructorArgs != null ?
						ctor.newInstance(this.constructorArgs) : ctor.newInstance());
			}
```

