---
layout: post
title: 关于Unsafe类的一点研究
category: JAVA
excerpt: 关于Unsafe类的一点研究
---
Unsafe类是java中非常特别的一个类。它名字就叫做“不安全”，提供的操作可以直接读写内存、获得地址偏移值、锁定或释放线程。

通过正常途径是无法获得Unsafe实例的，首先它的构造方法是私有的，然后，即使你调用它的getUnsafe方法，也会抛出SecurityException。

> A collection of methods for performing low-level, unsafe operations. Although the class and all methods are public, use of this class islimited because only trusted code can obtain instances of it.

任何关于Unsafe类的文章都不会推荐我们在代码中使用它，但这并不妨碍我们了解它可以做什么。下面我们来看下利用Unsafe类我们是否可以做点有趣的事情。

##获取Unsafe实例

~~~java
public static Unsafe getUnsafeInstance() throws Exception{
		Field unsafeStaticField = 
		Unsafe.class.getDeclaredField("theUnsafe");
		unsafeStaticField.setAccessible(true);
		return (Unsafe) unsafeStaticField.get(Unsafe.class);
}
~~~

通过java反射机制，我们跳过了安全检测，拿到了一个unsafe类的实例。

***我找遍了Unsafe类的API，没有发现可以直接获取对象地址的方法，Unsafe中操作地址相关的方法都要求提供一个Object类型的参数，用来获取对象的初始地址。***

##修改和读取数组中的值

~~~java
Unsafe u = getUnsafeInstance();
int[] arr = {1,2,3,4,5,6,7,8,9,10};
int b = u.arrayBaseOffset(int[].class);
int s = u.arrayIndexScale(int[].class);
u.putInt(arr, (long)b+s*9, 1);
for(int i=0;i<10;i++){
	int v = u.getInt(arr, (long)b+s*i);
	System.out.print(v+“ ”);
}
~~~

打印结果:1 2 3 4 5 6 7 8 9 1 ,可以看到，成功读出了数组中的值，而且最后一个值由10改为了1。

* arrayBaseOffset: 返回当前数组第一个元素地址相对于数组起始地址的偏移值，在本例中返回6。

* arrayIndexScale: 返回当前数组一个元素占用的字节数,在本例中返回4。

* putInt(obj,offset,intval): 获取数组对象obj的起始地址，加上偏移值，得到对应元素的地址，将intval写入内存。

* getInt(obj,offset): 获取数组对象obj的起始地址，加上偏移值，得到对应元素的地址，从而获得元素的值。

* 偏移值: 数组元素偏移值 = arrayBaseOffset+arrayIndexScalse\*i。

##获取对象实例

~~~java
/** Allocate an instance but do not run any constructor.
Initializes the class if it has not yet been. */
public native Object allocateInstance(Class cls) throws InstantiationException;
~~~

在不执行构造方法的前提下，获取一个类的实例，即使这个类的构造方法是私有的。

##修改静态变量和实例变量的值

先定义一个Test类

~~~java
public class Test {
	
	public int intfield ;
	
	public static int staticIntField;
	
	public static int[] arr;
	
	private Test(){
		System.out.println("constructor called");
	}
}
~~~

修改Test类的实例变量

~~~java
Unsafe u = getUnsafeInstance();

Test t = (Test) u.allocateInstance(Test.class);

long b1 = u.objectFieldOffset(Test.class.getDeclaredField("intfield"));

u.putInt(t, b1, 2);

System.out.println("intfield:"+t.intfield);
~~~

这里使用allocateInstance方法获取了一个Test类的实例，并且没有打印“constructor called”，说明构造方法没有调用。
修改实例变量与修改数组的值类似，同样要获取地址偏移值，然后调用putInt方法。

* objectFieldOffset: 获取对象某个属性的地址偏移值。

***我们通过Unsafe类修改了Java堆中的数据。***

修改Test类的静态变量

~~~java
Field staticIntField = Test.class.getDeclaredField("staticIntField");

Object o = u.staticFieldBase(staticIntField);

System.out.prinln(o==Test.class);

Long b4 = u.staticFieldOffset(staticIntField);
//因为是静态变量，传入的Object参数应为class对象
u.putInt(o, b4, 10);

System.out.println("staticIntField:"+u.getInt(Test.class, b4));	
~~~

打印结果：

true

staticIntField:10

静态变量与实例变量不同之处在于，静态变量位于于方法区中，它的地址偏移值与Test类在方法区的地址相关，与Test类的实例无关。

* staticFieldBase: 获取静态变量所属的类在方法区的首地址。可以看到，返回的对象就是Test.class。

* staticFieldOffset: 获取静态变量地址偏移值。

***我们通过Unsafe类修改了方法区中的信息。***

##调戏String.intern

在jdk7中，String.intern不再拷贝string对象实例，而是保存第一次出现的对象的引用。在下面的代码中，通过Unsafe修改被引用对象s的私有属性value达到间接修改s1的效果！

~~~java
String s = "abc";

//保存s的引用
s.intern();

//此时s1==s，地址相同
String s1 = "abc";

Unsafe u = getUnsafeInstance();

//获取s的实例变量value
Field valueInString = String.class.getDeclaredField("value");

//获取value的变量偏移值
long offset = u.objectFieldOffset(valueInString);

//value本身是一个char[],要修改它元素的值，仍要获取baseOffset和indexScale
long base = u.arrayBaseOffset(char[].class);

long scale = u.arrayIndexScale(char[].class);

//获取value
char[] values = (char[]) u.getObject(s, offset);

//为value赋值
u.putChar(values, base + scale, 'c');

System.out.println("s:"+s+" s1:"+s1);

//将s的值改为 abc
s = "abc";

String s2 = "abc";

String s3 = "abc";

System.out.println("s:"+s+" s1:"+s1);
~~~

打印结果：

s:acc s1:acc

s:acc s1:acc s2:acc s3:acc

***我们发现了什么？所有值为“abc”的字符串都变成了“acc”！！！***

Unsafe类果然不安全！！！




