---
title:  JAVA 设计模式——代理模式
date: "2018-12-03 11:13:17"
sidebar: false
categories:
 - 技术
tags:
 - 设计模式
 - java
publish: true
---



### 1、静态代理

**<font size=4>是什么：</font>**

A 是接口，B 是 A 接口的实现类。C 是代理类，实现 A 接口，属于 B 的扩展。

**<font size=4>代码：</font>**

```java
public interface A{
	void show();
}


public class B implements A{

	public void show(){
		System.out.println("show....");
	}
}

public class C implements A{

	private A a;
	public C(A a){
		this.a = a;
	}

	public void show(){
		System.out.println("preShow.....");
		a.show();
		System.out.println("afterShow....");
	}

}
```

**<font size=4>测试：</font>**

```java
public class Application(){
	public static void main(String[] args){
		//需要代理的对象
		B b = new B();
		C c = new C(b);
		//执行代理方法
		c.show();
	}
}
```

**<font size=4>总结：</font>**

    优点：
    1、使用静态代理，可以在不修改目标对象(B)的情况下，对目标功能进行扩展。

    缺点：
    1、由于代理对象(C)需要与目标对象(B)实现相同的接口(A)，所以可能会有很多的代理类。并且一旦接口(A)要添加新方法，目标对象(B)和代理对象(C)都需要维护修改。

由此，可以引入动态代理的方式解决这一缺点。

---

### 2、动态代理

**<font size=4>是什么：</font>**
代理对象(C)不需要实现接口(A)，代理对象(C)的实现由 JDK 反射代理类(`java.lang.reflect.Proxy`)在内存中动态构建。(需要指定创建代理对象(C)/目标对象(B)实现的接口(A)类型)

JDK 反射代理类 Proxy：
实现代理需要使用其中的`newProxyInstance`方法。

```java
static Object newProxyInstance(ClassLoader loader,Class<?>[] interface,InvocationHandler h)
```

该方法在 Proxy 中属于静态方法，其中参数为：

- `ClassLoader loader`:指定当前目标对象使用类加载器，获取加载器的方法是固定的
- `Class<?>[] interface`:目标对象实现的接口的类型，使用泛型确认类型
- `InvocationHandler h`:事件处理，执行目标对象的方法时，会触发事件处理器的方法，会把当前执行目标对象的方法作为参数传入

**<font size=4>代码：</font>**

```java
/**
 * 创建动态代理对象
 * 动态代理不需要实现接口，但需要指定接口的类型
 */
public class ProxyFactory{

	//需要代理的目标对象
	private Object target;
	public ProxyFactory(Object target){
		this.target = target;
	}

	//为目标对象生成代理对象
	public Object getProxyInstance(){
		return Proxy.newProxyInstance(
					target.getClass().getClassLoader(),
					target.getClass().getInterfaces(),
					new InvocationHandler(){
						public Object invoke(Object proxy,Method method,Object[] args) throws Throwable{
							System.out.println("");
							//执行目标方法
							Object o = method.invoke(target,args);
							System.out.println("");
							return o;
						}
					}
			);
	}

}
```

**<font size=4>测试：</font>**

```java
public class Application{
	public static void main(String[] agrs){
		//目标对象
		A a = new B();
		//创建代理对象
		A aProxy =(A) new ProxyFactory(a).getProxyInstance();
		//执行代理方法
		aProxy.show();
	}
}
```

**<font size=4>总结：</font>**

动态代理的代理对象(ProxyFactory.getProxyInstance())不需要实现接口，但是目标对象(B)需要实现接口(A)

---

### 3、Cglib 代理

**<font size=4>是什么：</font>**

不管静态代理还是动态代理，都是要求目标对象(B)实现接口(A)。但是有时候需要代理的对象并没有实现任何接口，这时候需要以目标对象的子类作为代理对象实现代理。所以 Cglib 也叫做子类代理。

Cglib 是一个代码生成包，它可以在运行期扩展 java 类和接口。被许多 AOP 框架使用，例如 spring 的 aop 和 synaop，为它们提供 interception(拦截)

Cglib 底层是通过使用一个小而快的字节码处理框架 ASM 来转换字节码并生成新的类(不推荐直接使用 ASM，因为要求你必须对 jvm 内部结构包括 class 文件的格式和指令集非常熟悉)

**<font size=4>代码：</font>**

```java
//没有实现接口，且非final/static的类
public class D(){

	public void show(){
		System.out.println("show...");
	}
}

/**
 * Cglib子类代理工厂
 * 在内存中动态构建目标对象的子类对象
 */
public class CglibProxyFactory implements MethodInterceptor{

	//目标对象
	private Object target;

	public CglibProxyFactory(Object target){
		this.target = target;
	}

	//创建代理(子类)对象
	public Object getProxyInstance(){
		//工具类
		Enhancer en = new Enhancer();
		//设置父类
		en.setSuperclass(target.getClass());
		//设置回调函数
		en.setCallback(this);
		//创建代理(子类)对象
		return en.create();
	}

	@Override
	public Object intercept(Object obj,Method method,Object[] args,MethodProxy proxy) throws Throwable{
		System.out.println("preShow......");

		Object o = proxy.invokeSuper(obj,args);

		System.out.println("afterShow......");

		return o;
	}

}
```

**<font size=4>测试：</font>**

```java
public class Application(){

	public static void main(String[] args){
		//目标对象
		D d = new D();
		//代理对象(子类)
		D dProxy = (D) new CglibProxyFactory(d).getProxyInstance();
		//执行代理方法
		dProxy.show();
	}
}
```

**<font size=4>总结：</font>**

在 spring 的 aop 编程中，如果加入容器的目标对象有实现接口，则使用 JDK 代理，如果目标对象没有实现接口，则使用 Cglib 代理。

---
