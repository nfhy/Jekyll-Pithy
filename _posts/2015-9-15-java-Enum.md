---
layout: post
title: 关于java Enum
excerpt: 关于java Enum
category: JAVA
---

java的枚举类型声明非常简单，这里我们声明一个只有两个枚举值的TypeB

~~~java
public enum TypeB {
	A,B;
}
~~~

随后编译器生成TypeB.class文件，我们来研究下里面藏了什么

首先是静态变量和静态代码块：

~~~java
{
  public static final testAnything.TypeB A;
    
  public static final testAnything.TypeB B;
    //声明两个TypeB的实例，和我们声明的枚举值同名
  static {};
    flags: ACC_STATIC
    Code:
      stack=4, locals=0, args_size=0
         0: new           #1                  // class testAnything/TypeB
         3: dup           
         4: ldc           #13                 // String A
         6: iconst_0      
         7: invokespecial #14                 // Method "<init>":(Ljava/lang/String;I)V
        10: putstatic     #18                 // Field A:LtestAnything/TypeB;
         //调用Enum(String,int)方法，生成TypeB.A
         
        13: new           #1                  // class testAnything/TypeB
        16: dup           
        17: ldc           #20                 // String B
        19: iconst_1      
        20: invokespecial #14                 // Method "<init>":(Ljava/lang/String;I)V
        23: putstatic     #21                 // Field B:LtestAnything/TypeB;
         //调用Enum(String,int)方法，生成TypeA.B
         
        26: iconst_2      
        27: anewarray     #1                  // class testAnything/TypeB
        //生成一个长度为2的TypeB[]
        
        30: dup           
        31: iconst_0      
        32: getstatic     #18                 // Field A:LtestAnything/TypeB;
        35: aastore       
        36: dup           
        37: iconst_1      
        38: getstatic     #21                 // Field B:LtestAnything/TypeB;
        41: aastore       
        42: putstatic     #23                 // Field ENUM$VALUES:[LtestAnything/TypeB;
        //将A,B的值放入TypeB[]，并赋值给私有静态变量ENUM$VALUE
        45: return        
~~~

上面这部分字节码指令主要做了两件事：

1. 为两个公共静态变量A和B赋值
2. 将A,B的值存入私有静态变量ENUM$VALUES中

这里的ENUM$VALUES变量名是可变的，如果在enum类中声明了同名的变量，编译器选用其他变量名。下面是values()静态方法，它返回枚举类型的所有枚举值：

~~~java

public static testAnything.TypeB[] values();
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=5, locals=3, args_size=0
         0: getstatic     #25                 // Field ENUM$VALUES:[LtestAnything/TypeB;
         3: dup           
         4: astore_0      
         5: iconst_0      
         6: aload_0       
         7: arraylength   
         8: dup           
         9: istore_1      
        10: anewarray     #1                  // class testAnything/TypeB
        13: dup           
        14: astore_2      
        15: iconst_0      
        16: iload_1       
        17: invokestatic  #33                 // Method java/lang/System.arraycopy:(Ljava/lang/Object;ILjava/lang/Object;II)V
        20: aload_2       
        21: areturn       
    
~~~

这个方法做的事只有一件，拷贝ENUM$VALUE并返回。具体流程大家有兴趣可以自己画下运行栈和变量表。

接下来是继承自Enum类的valueOf方法:

~~~java
public static testAnything.TypeB valueOf(java.lang.String);
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=1
         0: ldc           #1                  // class testAnything/TypeB
         2: aload_0       
         3: invokestatic  #41                 // Method java/lang/Enum.valueOf:(Ljava/lang/Class;Ljava/lang/String;)Ljava/lang/Enum;
         6: checkcast     #1                  // class testAnything/TypeB
         9: areturn       
}
~~~

这个方法与它重写的方法唯一的区别就是把返回值类型由Object变为TypeB。

这样，我们就可以模拟写出TypeB.java了：

~~~java
public class TypeB extends Enum{
//上面的继承会报错，不能显式的继承Enum
	public static TypeB A;
	public static TypeB B;
	
	private static TypeB[] ENUM$VALUE ;
	
	static{
		A = new Enum(A,0);
		B = new Enum(B,1);
		//上面的调用无法编译通过，Enum的修饰符是protected
		TypeB[] newarr = new TypeB[2];
		newarr[0] = A;
		newarr[1] = B;
		ENUM$VALUE = newarr;
	}
	
	public static TypeB[] values(){
		int length = ENUM$VALUE.length;
		TypeB[] newArr = new TypeB[length];
		System.arraycopy(ENUM$VALUE, 0, newArr, 0, length);
		return newArr;
	}
	
	public static TypeB valueOf(String name){
		return (TypeB)Enum.valueOf(TypeB.class, name);
	}
	
}

~~~
