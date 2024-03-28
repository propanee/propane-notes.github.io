# 二进制操作

```java
// ^ 异或 1011 ^ 0010 = 1001 相同为0，不同为1
// | 或   1011 | 0010 = 1011 至少有一个1，则为1，否则为0
// & 与	 1011 & 0110 = 0010 全为1，则为1，否则为0
```

## 交换两个元素

```java
// 交换两个元素
s[l] ^= s[r]; // a ^ b 存到a   		1011 ^ 0010
s[r] ^= s[l]; // b ^ a ^ b = a 存到b  0010 ^ 1011 ^ 0010
s[l] ^= s[r]; // a ^ b ^ a = b 存到a  1011 ^ 0010 ^ 1011
```

## 求余

给定x, y两个数值，想求x与y的余数，只需要x & (y-1)即可，如想求45和12（45和8）的余数，只要求45 & 11（45 & 7）。

```java
// 当x=2^n(n为自然数)时，
// a % x = a & (x  - 1 )
public ModTest{
    public static void main(String[] args){
        System.out.println(45 & 11);
        System.out.println(45 & 7);
    }
/**result:3, 5*/
```

## 左移<<

空位补0，被移除的高位对其，空缺位补0，在一定范围内，数据每向左移动一位，相当于原数据*2。（正数、负数都适用）

当左移的位数n超过该数据类型的总位数时，相当于左移（n-总位数）位

```java
2<<leftDepth // 相当于 2^(leftDepth+1)次方
```

```java
3<<4 //类似于 3*2的4次幂 => 3*16 => 48
```

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/202401032137916.jpeg" alt="img" style="zoom: 67%;" />

```java
-3<<4  //类似于  -3*2的4次幂 => -3*16 => -48
```

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/202401032137651.jpeg" alt="img" style="zoom:67%;" />

## **右移：>>**

运算规则：在一定范围内，数据每向右移动一位，相当于原数据/2。（正数、负数都适用）

【注意】如果不能整除，`向下取整`。

```java
69>>4 //类似于 69/2的4次 = 69/16 =4
```

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/202401032139060.jpeg" alt="img" style="zoom:67%;" />

```java
-69>>4 //类似于 -69/2的4次 = -69/16 = -5
```

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/202401032139871.jpeg" alt="img" style="zoom:67%;" />

## 无符号右移：>>>

运算规则：往右移动后，左边空出来的位直接补0。（正数、负数都适用）

```java
69>>>4 //类似于 69/2的4次 = 69/16 =4
```

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/202401032139408.jpeg" alt="img" style="zoom:67%;" />

```java
-69>>>4 //结果：268435451
```

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/202401032139128.jpeg" alt="img" style="zoom:67%;" />
