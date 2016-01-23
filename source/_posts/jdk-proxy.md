---
title: jdk-proxy
date: 2016-01-22 16:38:22
tags: proxy
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

	public class DynamicProxy {
    public static void main(String[] args) {
        IProxy proxy = (IProxy) Proxy.newProxyInstance(DynamicProxy.class.getClassLoader(), new Class[]{IProxy.class}, new Handler());
        proxy.say();
    }

    public static class Handler implements InvocationHandler{

        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            System.out.print("call proxy invoke!!!");
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


# 静态代理和动态代理的应用


java反射方法调用使用了静态代理

反射调用 Method.invoke(Object o, Object... args)

MethodAccessor 当反射调用次数超过一定次数,生成字节码替换被代理对象的实现.这个时候就必须使用静态代理了