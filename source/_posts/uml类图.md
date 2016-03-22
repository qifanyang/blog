---
title: uml类图
date: 2016-03-22 10:43:56
tags: 杂七杂八
---

## uml类图对象关系

### 依赖关系（Dependence）(虚线箭头)
A类变化引起B类变化,则说B类依赖A类
![](/img/uml_dependence.png)

	public class Driver  
	{  
    public void drive(Car car)  
    {  
        car.move();  
    }  
    ……  
	}  
	public class Car  
	{  
    public void move()  
    {  
        ......  
    }  
    ……  
	}  
依赖关系有如下三种情况：

1、A类是B类中的（某中方法的）局部变量；

2、A类是B类方法当中的一个参数；

3、A类向B类发送消息，从而影响B类发生变化；

### 泛化关系（Generalization)(实现三角箭头)
继承,A是B和C的父类，B,C具有公共类（父类）A，说明A是B,C的一般化（概括，也称泛化
![](/img/uml_generalizatiaon.png)	

	public class Person   
	{  
    	protected String name;  
    	protected int age;  
    	public void move()   
    	{  
        ……  
    	}  
    	public void say()   
    	{  
        ……  
    	}  
	}  
	public class Student extends Person   
	{  
    	private String studentNo;  
    	public void study()   
    	{  
        ……  
    	}  
	}  
在UML当中，对泛化关系有三个要求：

1、子类与父类应该完全一致，父类所具有的属性、操作，子类应该都有；

2、子类中除了与父类一致的信息以外，还包括额外的信息；

3、可以使用父类的实例的地方，也可以使用子类的实例；

### 关联关系（Association）
类之间的联系，如客户和订单，每个订单对应特定的客户，每个客户对应一些特定的订单，再如篮球队员与球队之间的关联

单向关联(箭头表示)

双向关联(直线表示)

聚合关系（Aggregation）:表示的是整体和部分的关系，整体与部分 可以分开.空心
![](/img/uml_juhe.png)

组合关系（Composition）:也是整体与部分的关系，但是整体与部分不可以分开. 实心
![](/img/uml_zuhe.png)

### 实现关系（Implementation)
java类实现接口
![](/img/uml_impl.png)

	
	public interface Vehicle   
	{  
    	public void move();  
	}  
	public class Ship implements Vehicle  
	{  
    	public void move()   
    	{  
    	……  
    	}  
	}  
	public class Car implements Vehicle  
	{  
    	public void move()   
    	{  
    	……  
    	}  
	}  