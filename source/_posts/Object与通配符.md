---
title: Object与通配符
date: 2016-04-01 15:32:14
tags: java
---

## 背景

使用集合时偶尔会用到通配符,例如List<? extends Shape> shapes, 向shapes添加对象必须是Shape或者其子类,曾经以为List<?>和List<Object>是等价的,但是测试结果表明他们不是等价的.

	 ArrayList<?> l1 = new ArrayList<>();
	//l1.add(new Object()); 无法通过编译, 

	 ArrayList l1 = new ArrayList<>();
	l1.add(new Object()); //ok 


## 通配符
通配符代表未知类型,add(E), E为集合的类型, 当使用List<?>时, ?就是E, 所以add传入的参数必须是未知类型的子类型,但是我们又不知道?是什么类型,所以不能传递任何数据类型(除了null),泛型就是为了解决类型安全问题,会检查add的类型是不是和E匹配


## 通配符可以这么用

	void printCollection(Collection<?> c) {//改方法可以传入任意集合
    for (Object e : c) {
        System.out.println(e);
    }
	}

因为Object是所有类型的父类型,所以get()可以转换为Object, 是类型安全的


## Bounded Wildcards 有界通配符

	public void drawAll(List<Shape> shapes) {//无法传入List<Circle>
    for (Shape s: shapes) {
        s.draw(this);
    }
	}

	public void addRectangle(List<? extends Shape> shapes) {//可以传入List<Circle>, ?可以是子类也可以是Shape
    // Compile-time error!
    shapes.add(0, new Rectangle());//? extends Shape, 未知类型, 是不能放进去的, 因为我们不知道Rectangle类型的父类有哪些,
	//泛型的存在就是为了保证类型安全,集合中放入不确定类型的对象就和原生Object没有区别了
	}