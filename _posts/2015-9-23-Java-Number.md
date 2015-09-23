---
layout: post
title: Java Number小结
excerpt: java Number小结
category: JAVA
---

####Byte

* 长度：1byte/8bit
* 最大值：127  0b0111_1111
* 最小值：-128 0b1000_0000

####Short

* 长度：2byte/16bit
* 最大值： 2^15  32767 0x7fff 0b0111_1111_1111_1111
* 最小值：-2^15 -32768 0x8000 0b1000_0000_0000_0000

####Integer

* 长度：4byte/32bit
* 最大值： 2^31 -21亿 0x7fffffff
* 最小值：-2^31 -21亿 0x80000000

####Long

* 长度：8byte/64bit
* 最大值： 2^63 约9*10^18
* 最小值：-2^63


###关于单精浮点数和双精浮点数结构

根据IEEE754标准，Float和Double的计算公式

~~~
在单精度时： 
  V=(-1)^s*2^(E-127)*M    
在双精度时： 
  V=(-1)^s*2^(E-1023)*M 
~~~

以float为例，float长度32位，位数从高到低

* 第1位为s符号位，1为负值，0为正值
* 第2-9位共8位为指数为E，取值范围0~255，减去偏移量127后，指数取值范围-127-128，其中取值0(全0)和255(全1)有特殊用途，因此指数实际取值范围-126~127
* 剩余23位为小数位，表示24数字，省略了前导整数位1，即一般情况下M≥1.0

当指数位取值1，小数位取最小值时，即0x00800000，获得float最小正标准值。

关于最小正标准值，摘抄一段stackoverflow中的回答：
>For the single format, the difference between a normal number and a subnormal number is that the leading bit of the significand (the bit to left of the binary point) of a normal number is 1, whereas the leading bit of the significand of a subnormal number is 0. Single-format subnormal numbers were called single-format denormalized numbers in IEEE Standard 754.

也就是说，最小正标准值是保证小数位前导位为1的最小值。

非规格化表示：

当指数位全0时，小数位前导整数位由1变为0，此时取值范围：

~~~
2^(-126)*2^(-23) ~ 2^(-126)*(1-2^(-23))
此时的最小值也是float的正最小值，2^(-149)
~~~

特殊表示：

* 当指数位和小数位全0时，表示0值，有+0和-0之分。
* 当指数位全1，小数位不全0时，表示NaN。Java中用0x7fc00000表示NaN。
* 指数部分全1，小数部分全0，表示无穷大，Java中用0x7f800000表示正无穷大，0xff800000表示负无穷大。

关于+0和-0,《java语言规范》中说的很明确：

>Positive zero and negative zero compare equal; thus the result of the expression 0.0==-0.0 is true and the result of 0.0>-0.0 is false. But other operations can distinguish positive and negative zero; for example, 1.0/0.0 has the value positive infinity, while the value of 1.0/-0.0 is negative infinity.

对于float类型的+0.0f与-0.0f

* +0.0 == -0.0
* +0.0 > -0.0 = false
* 1.0/+0.0 = 0x7f800000
* 1.0/-0.0 = 0xff800000
 

关于NaN，同样来自《java语言规范》：
>NaN is unordered, so:
>•The numerical comparison operators < , <= , > , and >= return false if either or both operands are NaN (§15.20.1).>• The equality operator == returns false if either operand is NaN.In particular, (x<y) == !(x>=y) will be false if x or y is NaN.>• The inequality operator != returns true if either operand is NaN (§15.21.1).In particular, x!=x is true if and only if x is NaN.

* 对于float值x，y，如果 x,y至少一个为NaN，那么 < , <= , > , >= ,== 都返回false。(x < y) == !(x >= y) 返回false。

* 当且仅当x = NaN时，x!=x返回true。


综合以上，可以得到float的正最小值为2^(-149)，即0x00000001，正最大值为0x7f7fffff，也即0x7fffffff-0x00000001(正无穷大-正最小值)

####Float

* 长度：4byte/32bit
* 正最大值：0x7f7fffff，约3.4*10^38
* 正最小值：0x00000001，月1.4*10^(-45)

对于Float的equals方法，比较的是两个float值的二进制表示，因此+0.0f.equals(-0.0f)返回false,NaN.equals(NaN)返回true，这是FLoat与float不同的地方。

####Double

* 长度：8byte/64bit
* 正最大值：1.8*10^308
* 正最小值：4.9*10^(-304)