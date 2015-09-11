---
layout: post
title: Java中的多态
excerpt: 多态在Java中的一些特点
category: JAVA
---

##写在前面
-------------
最近开始回头重读《Thinking in Java》，把感兴趣的部分再仔细看一遍，顺便做做读书笔记。子曰：学而时习之，不亦说乎？ 

##正文
-------------
**“多态通过分离 做什么 和 怎么做 ，从另一个角度讲接口和实现分离开来”**

**“多态的作用是消除 类型 之间的耦合关系”**

多态使得同一份代码可以不做任何改变就可以引用多种类的对象，代码只知道这些对象类的父类，至于这些对象所属的子类、这些子类的行为对这代码是完全透明的。这就省去了在代码中令人厌烦的用于判断对象类型的一堆if或者switch 语句。而当需求发生变化，需要引入更多的子类时，也不需要对这部分代码进行修改，这正符合“对修改关闭，对扩展开放”的原则。

 

**“将一个方法调用同一个方法主体关联起来被称作 绑定”**

**“在运行时根据对象类型进行绑定。后期绑定也叫做动态绑定或运行时绑定。”**

动态绑定是多态的实现方式。

Java中除了static方法、final方法、private方法（属于final方法）、构造方法（隐式声明的static方法）外，其他所有方法都是动态绑定的。

经常能看到如何提升Java代码性能的文章中提到尽可能声明final方法，因为这样编译器不需要考虑动态绑定，可以为final方法调用生成更有效的方法，但是书中说

**“大多数情况下，这样做对程序的整体性能不会有什么改观。所以，最好根据设计来决定是否使用fianl，而不是出于试图提高性能的目的来使用final”。**

 

关于动态绑定这一部分，经常见到这样的问题：

```java

public class A {
    
    public void show(C c){
        System.out.println("A show C");
    }
    public void show(A a){
        System.out.println("A show A");
    }
    public class C extends B{
        public void show(A a){
            System.out.println("C show A");
        }
    }
    public class D extends B{}
    
    public void test(){
        A a=new A();
        A b=new B();
        B b1=new B();
        C c=new A().new C();
        D d=new A().new D();
        b.show(b);//1
        b.show(c);//2
        b.show(d);//3
        b1.show(b);//4
        b1.show(d);//5
        c.show(d);//6
        }
}

public class B extends A{
    public void show(B b){
        System.out.println("B show B");
    }
    public void show(C c){
        System.out.println("B show C");
    }
    public static void main(String[] args){
        new A().test();
    }
    
}
```

运行B的main方法，打印结果是？

A show A

B show C

A show A

A show A

B show B

B show B

 

相信很多人和我一样，自信满满拿出笔，潇洒的写出答案，然后一对比，错了一半……

这个问题涉及到动态绑定时，方法的查找规则。

**对于一个方法foo(arg)，方法搜索的过程按照foo(arg)àsuper.foo(arg)àfoo((super)arg)àsuper.foo((super)arg)这一顺序进行.**

接下来针对上面的输出一个个说明。

*	b.show(b)，b的类型为A，引用了B的对象，根据搜索顺序，先找A.show(A)方法，在A中存在这一方法，虽然A引用了B的对象，但是B中并没有show(A)方法覆盖A.show(A)，所以执行A.show(A)方法，打印“A show A”。   
*	b.show(c)，b的类型为A，引用了B的对象，根据搜索顺序，先找A.show(C)方法，在A中存在这一方法，但是先别着急，因为A引用了B的对象，而B中也有方法show(C)，覆盖了A中的方法，因此实际上执行的是B.show(C)方法，打印“B show C”。
*	b.show(d)，b的类型为A，引用了B的对象，根据搜索顺序，先找A.show(D)，没有找到；接下来由于A没有父类了，super.foo(arg)不需要搜索；然后是foo((super)arg)，D继承自B，查找A.show(B)方法，依然没有；然后是super.foo((super)arg)，由于A没有父类，不需要搜索；这时，好像搜索已经结束了，其实不然，由于super.foo(arg)不存在，foo((super)arg)会替代当前foo(arg)，继续上面的流程。也就是说对方法A.show(B)继续走一遍搜索流程，这一次可以找到方法A.show(A)符合要求，可以执行，打印“A show A”。
*	b1.show(b)，b1类型为B引用B的对象，b的类型为A引用B的对象，首先找B.show(A)，没有找到；接下来找super.foo(arg)，B继承自A，而A.show(A)方法是存在的，打印“A show A”。
*	b1.show(d),b1类型为B引用B的对象，d的类型为D引用D的对象，首先找B.show(D)，没有找到；接下来找super.foo(arg)，同样没有A.show(D);接下来是foo((super)arg)，D继承自B，B.show(B)是存在的，可以执行，打印“B show B”。
*	c.show(d)，c类型为C引用C的对象，d的类型为D引用D的对象，首先找C.show(D)，没有找到；接下来找super.foo(arg)，C继承自B，B.show(D)同样不存在，B继承自A，而A.show(D)也不存在，没有找到；接下来找foo((super)arg)，D继承自B，C.show(B)不存在，没有找到；接下来是super.foo((super)arg)，即B.show(b)，方法存在，可以执行，打印“B show B”。
 

**“用继承表达行为间的差异，用字段表达状态上的变化”**

这个原则是作者在讲述在设计中选择继承还是组合时提出的。当你不确定应该选择哪种方式时，优先考虑组合。不合理的继承会加重设计的负担，增加不必要的复杂性；相反组合不会强制设计与类型层次结构耦合，设计更加灵活。

