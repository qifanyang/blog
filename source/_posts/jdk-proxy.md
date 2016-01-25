---
title: jdk-proxy
date: 2016-01-22 16:38:22
tags: 设计模式
---

# 代理类型
	动态代理,静态代理

# 静态代理
在调用方法的时候,调用代理对象的方法,通过代理对象去间接调用被代理对象的方法.
实现代码:

	public class StaticProxy implements IProxy{
    public IProxy proxy;

    public StaticProxy(IProxy proxy) {
        this.proxy = proxy;
    }

    public void say(){
        System.out.println("静态代理方法调用拦截");
        proxy.say();
    }
    public static class ProxyHandler implements IProxy{

        public void say() {
            System.out.println("static handler invoke!!!");
        }
    }

    public static void main(String[] args) {
        ProxyHandler proxyHandler = new ProxyHandler();
        StaticProxy staticProxy = new StaticProxy(proxyHandler);
        staticProxy.say();
    }
	}
	
静态代理类中有一个被代理对象的引用,所以使用静态代理代理不同的对象都要单独创建代理类,重复编写类似的代码.
解决方法

	1.使用Object指向被代理对象,然后使用反射调用除了Object自带方法之外的方法,有点动态代理的效果
	2.代理类的代码很简单,实现指定接口的方法和包含一个指向接口的引用,可以使用字节码来生成这个类

所以动态代理产生了,不用每次新建类去创建代理类,使用一个通用的方法Proxy.newProxyInstance(...)来创建代理类

	public static void main(String[] args) {
        IProxy proxy = (IProxy) Proxy.newProxyInstance(DynamicProxy.class.getClassLoader(), new Class[]{IProxy.class}, new Handler(new A()));
        proxy.say();
    }

    public static class A implements IProxy{
        public void say() {
            System.out.println("i am A proxy implements!!!");
        }
    }

    public static class Handler implements InvocationHandler{

        private Object obj;
        public Handler(Object obj){
            this.obj = obj;
        }

        public Object invoke(Object proxya, Method method, Object[] args) throws Throwable {
            //proxya是动态创建的代理类,就是proxy.say()中的proxy, 包含属性就是传入的InvocationHandler
            //当调用say()方法,会反射调用InvocationHandler的invoke方法
            System.out.println("call proxy invoke before !!!");
            method.invoke(obj,args);
            return null;
        }
    }
	}


创建动态代理需要参数:类加载器,被代理接口,拦截对象(在被代理对象方法被执行前需要执行的拦截方法)
1.为什么要传入类加载器?
	
	因为java有类加载机制,当新创建的类使用之前必须需要先加载,所以需要明确指定类加载器,而且一旦加载了类加载器会缓存被加载了的字节码,
	如果创建代理类使用的类加载器不一样那么会重复创建代理类,
2.如果不传入类加载器,使用当前类加载器呢?
	
	Proxy.newProxyInstance(...)是系统类,所以使用的是java自带的rootClassLoader加载器加载的,但是我们的代码是在classPath中一般使用appClassLoader,rootClassLoader是appClassLoader的父加载器,根据双亲委派模型可知,父加载器是无法加载子加载器的类,而且还有自定义类加载
	器的情况,所以必须自己明确的指定类加载器
	

3.为什么要传入代理接口?

	从静态代理可以看出,创建代理类需要一个被代理对象的接口引用,动态代理只是把静态代理类代码实现自动化了.

# 动态代理产生的字节码

查看了jdk8中是动态代理实现,抽取部分代码生成字节码并反编译
Proxy.newProxyInstance(DynamicProxy.class.getClassLoader(), new Class[]{IProxy.class}, new Handler());生成的字节码
	
	package test.proxy.jdk;
	//package com.sun.proxy;

	import java.lang.reflect.InvocationHandler;
	import java.lang.reflect.Method;
	import java.lang.reflect.Proxy;
	import java.lang.reflect.UndeclaredThrowableException;
	import test.proxy.jdk.IProxy;

	//jdk 动态代理生成的字节码
	public final class $Proxy0 extends Proxy implements IProxy
	{
    private static Method m1;
    private static Method m2;
    private static Method m3;
    private static Method m0;

    public $Proxy0(InvocationHandler paramInvocationHandler)

    {
        super(paramInvocationHandler);
    }

    public final boolean equals(Object paramObject)

    {
        try
        {
            return ((Boolean)this.h.invoke(this, m1, new Object[] { paramObject })).booleanValue();
        }
        catch (Error|RuntimeException localError)
        {
            throw localError;
        }
        catch (Throwable localThrowable)
        {
            throw new UndeclaredThrowableException(localThrowable);
        }
    }

    public final String toString()
    {
        try
        {
            return (String)this.h.invoke(this, m2, null);
        }
        catch (Error|RuntimeException localError)
        {
            throw localError;
        }
        catch (Throwable localThrowable)
        {
            throw new UndeclaredThrowableException(localThrowable);
        }
    }

    public final void say()
    {
        try
        {
            this.h.invoke(this, m3, null);
            return;
        }
        catch (Error|RuntimeException localError)
        {
            throw localError;
        }
        catch (Throwable localThrowable)
        {
            throw new UndeclaredThrowableException(localThrowable);
        }
    }

    public final int hashCode()
    {
        try
        {
            return ((Integer)this.h.invoke(this, m0, null)).intValue();
        }
        catch (Error|RuntimeException localError)
        {
            throw localError;
        }
        catch (Throwable localThrowable)
        {
            throw new UndeclaredThrowableException(localThrowable);
        }
    }

    static
    {
        try
        {
            m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[] { Class.forName("java.lang.Object") });
            m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
            m3 = Class.forName("test.proxy.jdk.IProxy").getMethod("say", new Class[0]);
            m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
	//            return;
        }
        catch (NoSuchMethodException localNoSuchMethodException)
        {
            throw new NoSuchMethodError(localNoSuchMethodException.getMessage());
        }
        catch (ClassNotFoundException localClassNotFoundException)
        {
            throw new NoClassDefFoundError(localClassNotFoundException.getMessage());
        }
   	 }
	}

从反编译的代码中可以看出,新生成的代理类不但继承了IProxy接口,而且还继承了Proxy类,该类中有个字段
InvocationHandler h;创建动态代理传入的InvocationHandler就会作为$Proxy0的构造函数参数.
 IProxy proxy = (IProxy) Proxy.newProxyInstance(DynamicProxy.class.getClassLoader(), new Class[]{IProxy.class}, new Handler());
等同于
Iproxy proxy = new $Proxy0(new Handler());

proxy.say();内部执行的是this.h.invoke(this, m3, null);其中this就是$Proxy0的实例对象,m3就是方法test.proxy.jdk.IProxy.say()
可以在Handler中只有被代理对象,然后通过反射去调用被代理对象的方法,这里有个限制,被代理对象始终要实现代理接口



# 静态代理和动态代理的应用


# java反射方法调用使用了静态代理

反射调用 Method.invoke(Object o, Object... args), 通过MethodAccessor来执行被调用的方法,MethodAccessor作为代理接口
有不同的实现,反射调用次数默认小于15次时,使用本地方法执行

	private static native Object invoke0(Method var0, Object var1, Object[] var2);

当调用次数大于15次时,会使用生成的字节码来执行,就像直接方法调用一样

	public Object invoke(Object var1, Object[] var2) throws IllegalArgumentException, InvocationTargetException {
        if(++this.numInvocations > ReflectionFactory.inflationThreshold() && !ReflectUtil.isVMAnonymousClass(this.method.getDeclaringClass())) {
            MethodAccessorImpl var3 = (MethodAccessorImpl)(new MethodAccessorGenerator()).generateMethod(this.method.getDeclaringClass(), this.method.getName(), this.method.getParameterTypes(), this.method.getReturnType(), this.method.getExceptionTypes(), this.method.getModifiers());
            this.parent.setDelegate(var3);
        }

        return invoke0(this.method, var1, var2);
    }

从这里可以看出,测试java程序效率时预热是很有必要的.这里使用了静态代理而且可以动态替换,动态代理也可以实现
同样的功能,生成字节码的时候多加一个setDelegate方法,不过这里使用静态代理更简洁.

静态代理在ibatis中也有广泛的应用,一个很长的Cache链,可以扩展Cache链,加上自己的cache实现,这些就没必要使用
动态代理了,因为cache数量是可以控制预估的,使用静态代理更简洁高效

ibatis mapper也使用了动态代理, 使用接口操作,更加方便


# 动态代理应用

Spring AOP中大量使用了动态代理,比如拦截器的实现,就是代理了所有的方法调用.spring使用了cglib这个字节码库来
实现,它有个好处就是不用实现代理接口,cglib实例代码:

	package leon.aj.dynproxy.cglib;

	import java.lang.reflect.Method;

	import net.sf.cglib.proxy.Enhancer;
	import net.sf.cglib.proxy.MethodInterceptor;
	import net.sf.cglib.proxy.MethodProxy;

	public class CglibProxy implements MethodInterceptor {
	private Object target;  
	
    public Object getProxyInstance(Object target) {  
    	this.target = target;
        Enhancer enhancer = new Enhancer();  
        enhancer.setSuperclass(this.target.getClass());  
        enhancer.setCallback(this);  // call back method
        return enhancer.create();  // create proxy instance
    }  
	
	@Override
	public Object intercept(Object target, Method method, Object[] args, MethodProxy proxy) throws Throwable {
		System.out.println("before target method...");
		Object result = proxy.invokeSuper(target, args);
		System.out.println("after target method...");
		return result;
	}
	}
	//
	//
 	CglibProxy proxy = new CglibProxy();  
    Hello hello = (Hello) proxy.getProxyInstance(new HelloImpl());  

cglib直接生成了一个继承被代理类的子类,并且修改了方法调用,每次调用之前都闲调用intercept方法,然后父类方法,好处是被代理
对象不用实现特定的接口,反编译cglib代理生成的类