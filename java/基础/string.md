# String

## String不可变性

[如何理解 String 类型值的不可变？ - 知乎提问](https://www.zhihu.com/question/20618891/answer/114125846)

不是在原内存地址修改数据，而是重新指向新对象：

![img](https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1021%20java%20string%E4%B8%8D%E5%8F%AF%E5%8F%98%E6%80%A7.webp)

```java
public final class String implements java.io.Serializable, Comparable<String>, CharSequence {
    private final char value[];
	//...
}
```

- 保存字符串的数组被 `final` 修饰且为私有的，`String`中的方法都没有动Array中的元素，并且`String` 类没有提供/暴露修改这个字符串的方法。
- 整个`String` 类被 `final` 修饰导致其不能被继承，进而避免了子类破坏 `String` 不可变。

> `String` 类中使用 `final` 关键字修饰字符数组来保存字符串，但这不是根本原因：
>
> 被 `final` 关键字修饰的类不能被继承，修饰的方法不能被重写，修饰的变量是基本数据类型则值不能改变，修饰的变量是**引用类型则不能再指向其他对象**。
>
> 因此，虽然value是不可变，也只是引用地址不可变，但Array数组是可变的，即这个数组保存的字符串是可变的。
>
> ![img](https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1021%20java%20string%E4%B8%8D%E5%8F%AF%E5%8F%98final.webp)
>
> 也就是说Array变量只是stack上的一个引用，数组的本体结构在head堆。String类中value用final修饰，只是stack中叫value的引用地址不可变，但堆中array本身的数据是可变的。
>
> ```java
> final int[ ] value={1,2,3}int[ ] another={4,5,6};
> value=another;//编译器报错，final不可变
> // value用final修饰，编译器不允许我把value指向堆区另一个地址。但如果我直接对数组元素动手，分分钟搞定。
> final int[ ] value={1,2,3};
> value[2]=100;//这时候数组里已经是{1,2,100}
> ```

> **Java 9 将 `String` 的底层实现由 `char[]` 改成了 `byte[]` **
>
> 新版的 String 其实支持两个编码方案：Latin-1 和 UTF-16。如果字符串中包含的汉字没有超过 Latin-1 可表示范围内的字符，那就会使用 Latin-1 作为编码方案。Latin-1 编码方案下，`byte` 占一个字节(8 位)，`char` 占用 2 个字节（16），`byte` 相较 `char` 节省一半的内存空间。JDK 官方表示绝大部分字符串对象只包含 Latin-1 可表示的字符。如果字符串中包含的汉字超过 Latin-1 可表示范围内的字符，`byte` 和 `char` 所占用的空间是一样的。

> **补充ASCII、UNICODE、LATIN-1、UTF-8-16-32**
>
> ***ASCII***
>
> - 单字节，在 0 ~ 127 范围是基础码，在128 ~255 是扩展码；
>
> - 在0 ~ 127范围内是全球标准，所有单字节编码（latin-1）,或长度可变编码（utf-8）都兼容。而 -1 ~ -128 范围则不兼容，每种编码规范所定义的标准都不同；
>
> ***Latin-1***
>
> 1. latin-1 就是 iso-8895-1;
>
> 2. latin-1 是单字节编码； 
> 3. 在 0 ~ 127 范围与 ascii 的一致；在128 ~ 255 则不同； 
> 4. 因为latin-1(ISO-8859-1)编码范围使用了单字节内的所有空间，在支持ISO-8859-1的系统中传输和存储其他任何编码的字节流都不会被抛弃。
> 5. 虽然Unicode中的前256个字符及编码值与Latin-1完全一致，但UTF-8只对前128个即ASCII字符采用单字节编码，其余128个Latin-1字符是采用2个字节编码。因此ASCII编码的文件可以直接以UTF-8方式读取，而Latin-1的文件若包含值为128-255的字符则不可以。
>
> ***UNICODE***
>
> - 一种规范，只是为全球定义了每一个字符的唯一的二进制编号，但并未指定值以何种形式存储。
> - UNICODE规范的常见实现方式：UTF-8、UTF-16、UTF-32。
>
> ***UTF-8***
>
> 不同范围内UNICODE的字符，UTF-8所存储的字节数都不同，用1到4个字节编码，是可变长度的编码方式。第一个字节leading byte+后续字节continuation byte。
>
> - 0xxxxxxx：单字节编码; 0000 ~ 007F; 1 Byte;
> - 110xxxxx 10xxxxxx：双字节编码; 0080 ~ 07FF; 2 Byte;
> - 1110xxxx 10xxxxxx 10xxxxxx：三字节编码;  0800 ~ FFFF; 3 Byte;
> - 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx：四字节编码; 10000 ~ 1FFFFF; 4 Byte;
> - 如 "田" 01110101,00110000 U+7530：1110*0111*, 10*010100*, 10*110000*
>
> ***UTF-16***
>
> ![img](https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1022%20UTF16.webp)
>
> 编码长度既是可变，亦是固定：采用存储的字节长度为2Byte 或 4Byte(lead/trail surrogate pair)。
>
> D800–DBFF和DC00–DFFF为保留区间，若落在D800–DBFF则作为lead surrogate，DC00–DFFF为trail surrogate；
>
> UTF-16编码分为 UTF-16 little endian（LE，小端，将低序字节存储在起始地址） 和 UTF-16 big endian （BE，大端，将高序字节存储在起始地址）
>
> - 如"中" 4E2D: LE为 0~15 *4E* 15~31 *2D*; BE为 0~15 *2D* 15~31 *4E*；
> - windows 采用是 utf-16 le ,而 mac 采用是 utf-16 be；
>
> UTF-16与UTF-8都是self-synchronizing的，即某个16bit是否是一个字符的开始无需检查其前一个或者后一个16bit。与UTF-8的不同之处是，UTF-16不支持从某个随机的字节开始读取。如0041, D801DC01，若第一个字节丢失即从第二个字节读取，则UTF-16认为是41D8,01DC,01，而UTF-8不存在这个问题。
>
> ***UTF-32***
>
> 每个字符都采用 4byte 字节来存储，浪费存储空间。
>
> ***BOM(byte-order mark)***
>
> - 标识字节顺序，即大端小端，该符号位于文本序列起始字节之前；
> - UTF-8中的BOM是 **0xEF,0xBB,0xBF**，但UTF8编码单位是字节，字节的先后顺序在编码、传输和解码过程中均不改变，因此BOM在UTF8中的唯一作用是标识序列使用的是UTF8编码，在由其它使用BOM的编码转换为UTF8编码的情况下，建议在UTF8序列中保留BOM以便转换回原编码的时候不丢失BOM信息；
> - UTF-16中为**U+FEFF**（BE）和**U+FFFE**（LE），可以使用编码方式UTF-16BE和UTF-16LE来显式地标明字节顺序，当使用了UTF-16BE和UTF-16LE时，BOM不应该再出现在字节序列中，很多应用直接忽略该字符若其仍然存在；
> - UTF-16中为0000FEFF和FFFE0000。

## String、StringBuffer、StringBuilder 的区别

***可变性***

- `String` 是不可变的

- `StringBuilder` 与 `StringBuffer` 可变；
  - 都继承自 `AbstractStringBuilder` 类；
  - 在 `AbstractStringBuilder` 中也是使用字符数组保存字符串，不过没有使用 `final` 和 `private` 关键字修饰；
  - 最关键的是这个 `AbstractStringBuilder` 类还提供了很多修改字符串的方法比如 `append` 方法。

```java
abstract class AbstractStringBuilder implements Appendable, CharSequence {
    char[] value;
    public AbstractStringBuilder append(String str) {
        if (str == null)
            return appendNull();
        int len = str.length();
        ensureCapacityInternal(count + len);
        str.getChars(0, len, value, count);
        count += len;
        return this;
    }
  	//...
}
```

**线程安全性**

- `String` 中的对象是不可变的，也就可以理解为常量，线程安全；

- `StringBuffer` 对方法加了同步锁或者对调用的方法加了同步锁，所以是**线程安全**的；`StringBuilder` 并没有对方法进行加同步锁，所以是**非线程安全**的；
  - `AbstractStringBuilder` 是 `StringBuilder` 与 `StringBuffer` 的公共父类，定义了一些字符串的基本操作，如 `expandCapacity`、`append`、`insert`、`indexOf` 等公共方法。

**性能**

- 每次对 `String` 类型进行改变的时候，都会生成一个新的 `String` 对象，然后将指针指向新的 `String` 对象；

- `StringBuffer` 每次都会对 `StringBuffer` 对象本身进行操作，而不是生成新的对象并改变对象引用；

- 相同情况下使用 `StringBuilder` 相比使用 `StringBuffer` 仅能获得 10%~15% 左右的性能提升，但却要冒多线程不安全的风险。

**总结**

1. 操作少量的数据: 适用 `String`；
2. 单线程操作字符串缓冲区下操作大量数据: 适用 `StringBuilder`；
3. 多线程操作字符串缓冲区下操作大量数据: 适用 `StringBuffer`。

## 字符串常量池

 JVM 为了提升性能和减少内存消耗针对字符串（String 类）专门开辟的一块区域，主要目的是为了避免字符串的重复创建。

```java
// 在堆中创建字符串对象”ab“
// 将字符串对象”ab“的引用保存在字符串常量池中
String aa = "ab";
// 直接返回字符串常量池中字符串对象”ab“的引用
String bb = "ab";
System.out.println(aa==bb);// true
```

### 字符串对象创建

```java
String s1 = new String("abc");// 这句话创建了几个字符串对象？
```

- 如果字符串常量池中不存在字符串对象“abc”的引用，那么它将首先在字符串常量池中创建，然后在堆空间中创建，因此将创建总共 2 个字符串对象。

![img](https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1022%20java%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%AF%B9%E8%B1%A1%E5%88%9B%E5%BB%BA.png)

`ldc` 命令用于判断字符串常量池中是否保存了对应的字符串对象的引用，如果保存了的话直接返回，如果没有保存的话，会在堆中创建对应的字符串对象，并将该字符串对象的引用保存到字符串常量池中。

- 如果字符串常量池中已存在字符串对象“abc”的引用，则只会在堆中创建 1 个字符串对象“abc”。

```java
// 字符串常量池中已存在字符串对象“abc”的引用
String s1 = "abc";
// 下面这段代码只会在堆中创建 1 个字符串对象“abc”
String s2 = new String("abc");
```

### String.intern()

`String.intern()` 是一个 native（本地）方法，其作用是将指定的字符串对象的引用保存在字符串常量池中，可以简单分为两种情况：

- 如果字符串常量池中保存了对应的字符串对象的引用，就直接返回该引用。
- 如果字符串常量池中没有保存了对应的字符串对象的引用，那就在常量池中创建一个指向该字符串对象的引用并返回。

```java
/**
 * Returns a canonical representation for the string object.
 * <p>
 * A pool of strings, initially empty, is maintained privately by the
 * class {@code String}.（常量池）
 * <p>
 * When the intern method is invoked, if the pool already contains a
 * string equal to this {@code String} object as determined by
 * the {@link #equals(Object)} method, then the string from the pool is
 * returned. Otherwise, this {@code String} object is added to the
 * pool and a reference to this {@code String} object is returned.
 * <p>
 * It follows that for any two strings {@code s} and {@code t},
 * {@code s.intern() == t.intern()} is {@code true}
 * if and only if {@code s.equals(t)} is {@code true}.
 * <p>
 * All literal strings and string-valued constant expressions are
 * interned. String literals are defined in section {@jls 3.10.5} of the
 * <cite>The Java Language Specification</cite>.
 *
 * @return  a string that has the same contents as this string, but is
 *          guaranteed to be from a pool of unique strings.
 */
// 在堆中创建字符串对象”Java“
// 将字符串对象”Java“的引用保存在字符串常量池中
String s1 = "Java";
// 直接返回字符串常量池中字符串对象”Java“对应的引用
String s2 = s1.intern();
// 会在堆中在单独创建一个字符串对象
String s3 = new String("Java");
// 直接返回字符串常量池中字符串对象"Java"对应的引用
String s4 = s3.intern();
// s1 和 s2 指向的是堆中的同一个对象
System.out.println(s1 == s2); // true
// s3 和 s4 指向的是堆中不同的对象
System.out.println(s3 == s4); // false
// s1 和 s4 指向的是堆中的同一个对象
System.out.println(s1 == s4); //true
```

## 字符串拼接 

### "+" or StringBuilder？

[还在无脑用 StringBuilder？来重温一下字符串拼接吧](https://juejin.cn/post/7182872058743750715)

- Java 语言本身并不支持运算符重载，“+”和“+=”是专门为 String 类重载过的运算符，也是 Java 中仅有的两个重载过的运算符。
- 字符串对象通过“+”的字符串拼接方式，实际上是通过 `StringBuilder` 调用 `append()` 方法实现的，拼接完成之后调用 `toString()` 得到一个 `String` 对象 。

```java
String str1 = "he";
String str2 = "llo";
String str3 = "world";
String str4 = str1 + str2 + str3;
```

Java8字节码：

![img](https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1022%20java%20string%E6%8B%BC%E6%8E%A5%E5%AD%97%E8%8A%82%E7%A0%81.png)

在循环内使用“+”进行字符串拼接，存在缺陷：**编译器不会创建单个 `StringBuilder` 以复用，会导致创建过多的 `StringBuilder` 对象**。`StringBuilder` 对象是在循环内部被创建的，这意味着每循环一次就会创建一个 `StringBuilder` 对象。

```java
String[] arr = {"he", "llo", "world"};
String s = "";
for (int i = 0; i < arr.length; i++) {
    s += arr[i];
}
System.out.println(s);
```

![img](https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1022%20java%E5%AD%97%E7%AC%A6%E4%B8%B2%E6%8B%BC%E6%8E%A5%E5%AD%97%E8%8A%82%E7%A0%812.png)

如果直接使用 `StringBuilder` 对象进行字符串拼接的话，就不会存在这个问题了。

```java
String[] arr = {"he", "llo", "world"};
StringBuilder s = new StringBuilder();
for (String value : arr) {
    s.append(value);
}
System.out.println(s);
```

![img](https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1022%20java%E5%AD%97%E7%AC%A6%E4%B8%B2%E6%8B%BC%E6%8E%A5stringbuilder%E5%AD%97%E8%8A%82%E7%A0%81.png)

> 不过，使用 “+” 进行字符串拼接会产生大量的临时对象的问题在 JDK9 中得到了解决。在 JDK9 当中，字符串相加 “+” 改为了用动态方法 `makeConcatWithConstants()` 来实现，而不是大量的 `StringBuilder` 了。这个改进是 JDK9 的 [JEP 280open in new window](https://openjdk.org/jeps/280) 提出的，这也意味着 JDK 9 之后，你可以放心使用“+” 进行字符串拼接了。关于这部分改进的详细介绍，推荐阅读这篇文章：还在无脑用 [StringBuilder？来重温一下字符串拼接吧open in new window](https://juejin.cn/post/7182872058743750715) 。

### String 类型的变量和常量做“+”运算时发生了什么?（常量折叠）

```java
String str1 = "str";
String str2 = "ing";
String str3 = "str" + "ing";
String str4 = str1 + str2;
String str5 = "string";
System.out.println(str3 == str4);//false
System.out.println(str3 == str5);//true
System.out.println(str4 == str5);//false
```

**对于编译期可以确定值的字符串，也就是常量字符串 ，jvm 会将其存入字符串常量池。并且，字符串常量拼接得到的字符串常量在编译阶段就已经被存放字符串常量池，这个得益于编译器的优化。**

编译过程中，Javac 编译器（下文中统称为编译器）会进行一个叫做 **常量折叠(Constant Folding)** 的代码优化。常量折叠会把常量表达式的值求出来作为常量嵌在最终生成的代码中，这是 Javac 编译器会对源代码做的极少量优化措施之一(代码优化几乎都在即时编译器中进行)。对于 `String str3 = "str" + "ing";` 编译器会给你优化成 `String str3 = "string";` 。

并不是所有的常量都会进行折叠，只有编译器在程序编译期就可以确定值的常量才可以：

- 基本数据类型( `byte`、`boolean`、`short`、`char`、`int`、`float`、`long`、`double`)以及字符串常量。
- `final` 修饰的基本数据类型和字符串变量
- 字符串通过 “+”拼接得到的字符串、基本数据类型之间算数运算（加减乘除）、基本数据类型的位运算（<<、>>、>>> ）

**引用的值在程序编译期是无法确定的，编译器无法对其进行优化。**

对象引用和“+”的字符串拼接方式，实际上是通过 `StringBuilder` 调用 `append()` 方法实现的，拼接完成之后调用 `toString()` 得到一个 `String` 对象 。

```java
String str4 = new StringBuilder().append(str1).append(str2).toString();
```

我们在平时写代码的时候，尽量避免多个字符串对象拼接，因为这样会重新创建对象。如果需要改变字符串的话，可以使用 `StringBuilder` 或者 `StringBuffer`。

不过，字符串使用 `final` 关键字声明之后，可以让编译器当做常量来处理。

```java
final String str1 = "str";
final String str2 = "ing";
// 下面两个表达式其实是等价的
String c = "str" + "ing";// 常量池中的对象
String d = str1 + str2; // 常量池中的对象
System.out.println(c == d);// true
```

被 `final` 关键字修饰之后的 `String` 会被编译器当做常量来处理，编译器在程序编译期就可以确定它的值，其效果就相当于访问常量。

如果 ，编译器在运行时才能知道其确切值的话，就无法对其优化。

示例代码（`str2` 在运行时才能确定其值）：

```java
final String str1 = "str";
final String str2 = getStr();
String c = "str" + "ing";// 常量池中的对象
String d = str1 + str2; // 在堆上创建的新的对象
System.out.println(c == d);// false
public static String getStr() {
      return "ing";
}
```

### Java9中的字符串拼接

Java 9 里字符串拼接的字节码使用了 **`InvokeDynamic`**。

```java
$ javac Concat.java
$ javap -c Concat
      ...
      13: aload_2
      14: aload_3
      15: aload         4
      17: invokedynamic #6,  0              // InvokeDynamic #0:makeConcatWithConstants:(Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;
      ...
```

```java
CallSite callSite = StringConcatFactory.makeConcatWithConstants(   
    MethodHandles.lookup(), 
    "", 
    MethodType.methodType(String.class, String.class, String.class, String.class),
    "[[[ \u0001\u0002 \u0001\u0002 \u0001 ]]]",
    "!",
    "~" ); 
System.out.println((String)callSite.dynamicInvoker().invokeExact("Hello", "World", "Invoke Dynamic"));
```

上面这段代码输出的结果是 `[[[ Hello! World~ Invoke Dynamic ]]]`。 可以看出来，上面的 `makeConcatWithConstants` 其实就是以 `[[[ \u0001\u0002 \u0001\u0002 \u0001 ]]]` 作为了一个模板。 第一次调用的时候可以理解成它把模板填充了一半，把两处的 `\u0002` 分别用 `!` 和 `~` 这两个参数替换了， 变成 `[[[ \u0001, \u0001~ \u0001 ]]]`。这个时候它返回的 `CallSite` 包含了一个 `MethodHandle`，可以进一步调用。 而再次调用这个 `MethodHandle` 时，它就把剩下的三处 `\u0001` 给用参数给填充上了，生成了我们看到的输出字符串。

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1022%20java9%E5%8A%A8%E6%80%81%E5%AD%97%E7%AC%A6%E4%B8%B2%E6%8B%BC%E6%8E%A5.png" alt="image-20231022120413153" style="zoom:50%;" /><img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1022%20java9%E5%AD%97%E7%AC%A6%E4%B8%B2%E6%8B%BC%E6%8E%A52.png" alt="image-20231022120658659" style="zoom: 40%;" />

而实际上，字节码里用到的 `InvokeDynamic` 所做的也就是我们上面做的。`str1 + str2 + str3` 对应的模板是 `\u0001\u0001\u0001`。 JVM 在执行到这一条 `InvokeDynamic` 指令时会自动使用 `makeConcatWithConstants` 对三个字符串的拼接操作进行优化，生成一个对应的 `CallSite` 对象， 最后用优化过的 `CallSite` 对象对字符串进行拼接。此后此处所有的字符串拼接都会使用同一个 `CallSite` 对象。

> 这里编译器只需要生成一句 `InvokeDynamic` 指令（以及对应的 `makeConcatWithConstants` 的初始参数），不需要担心性能优化。 这样使得在编译器升级时我们不必为了性能而重新编译之前的代码。
>
> 想想看，如果编译器生成了 `new StringBuilder(a.length() + b.length())...` 的代码， 那么在我们开启字符串压缩并且确认自己要处理 CJK 字符的时候，我们岂不是需要用某种奇妙的编译器选项让上面的代码重新编译为 `new StringBuilder((a.length() + b.length()) * 2)...`？因为目前的实现里 `StringBuilder` 默认按压缩处理，预分配的空间是 `N` 个 `byte` 而不是 `N` 个 `char`。
>
> 使用 `InvokeDynamic` 可以让 JVM 避免 boxing 和 unboxing 的开销。
>
> 我们上面手动的 `MethodHandle` 是必须把 `int` 变为 `Integer` 再传递进参数的。但是 `InvokeDynamic` 作为 JVM 直接支持的字节码， 它可以直接接受 `int` 之类的类型作为参数。（是的，Java 9 后 `"Hello" + 123` 这种拼接其实也是由 `InvokeDynamic` 实现的。）

***总结***

在 Java 9 及以后的版本，请大胆使用 `a + b + c + ...`。

- Java 9 后这里的字符串拼接使用了 `InvokeDynamic` 指令，默认会准确计算长度，只进行一次内存分配。 同时，它生成最终字符串时会使用字符串的内部 API，比起 `StringBuilder` 来说少了一次复制。
- Java 9 字符串拼接有不同的策略，默认策略就是上面所说的策略。其它策略在 Java 15 之后被删除掉了。

有对应的工具方法的操作，如 `String::join`，那还是用对应的方法好一点。

- `String::join` 在 Java 17 之前使用 `StringJoiner`，Java 17 之后采用了类似上面所说的优化。

在开启字符串压缩的情况下，`StringBuilder` 拼接无法压缩的（如中文）字符串会引来一次额外的内存分配。（至少 Java 19 在 `builder.setLength(0)` 后还会这样。）

## 其他操作

### substring

- `public String substring(int beginIndex, int endIndex)`
  - 开始索引位置、截止位置，区间：[ )；
  - endIndex最大值是父字符串的长度；

- `public String substring(int beginIndex)`
  - 该子字符串始于指定索引处的字符；

### 字符串与数组的转化

-  将字符串转化为字符数组：s.toCharArray

```java
// toCharArray
String str = "Hello";
char[] charArray = str.toCharArray();

// 手写toCharArray
String str = "Hello";
char[] charArray = new char[str.length()]; // 创建字符数组，长度与字符串相同
for (int i = 0; i < str.length(); i++) {
    charArray[i] = str.charAt(i); // 逐个获取字符并存储到字符数组中
```

- 字符数组转字符串：构造函数 or String.valueOf

```java
char[] charArray = {'H', 'e', 'l', 'l', 'o'};
String str = "";
// 使用构造函数
str = new String(charArray);
// 使用valueOf
str = String.valueOf(charArray);
```

- 字符串转字符串数组

```java
String str = "apple,banana,carrot";
String[] strArray = str.split(",");
```

- 字符串转整数数组

```java
String str = "1 2 3 4 5";
String[] strArray = str.split(" ");
int[] intArray = new int[strArray.length];
for (int i = 0; i < strArray.length; i++) {
    intArray[i] = Integer.parseInt(strArray[i]);
}

String s = "1 2 3 4 5";
int[] intArray = Arrays.stream(s.split(" ")).mapToInt(Integer::parseInt).toArray();
// 1. s.split(" ")：使用空格作为分隔符，将字符串 s 分割成多个子字符串，并返回一个字符串数组。
// 2. Arrays.stream(...)：将上一步得到的字符串数组转换为一个流（Stream）对象。
// 3. mapToInt(Integer::parseInt)：对流中的每个元素执行映射操作，将其转换为整数。这里使用了方法引用 Integer::parseInt，将字符串转换为对应的整数值。
// 4. toArray()：将流中的元素收集到一个数组中。
// 5. 最后，将结果数组赋值给数组，它的类型是 int[]，即整型数组
```

# StringBuffer

StringBuffer使用示例：反转字符串II

```java
public String reverseStr(String s, int k) {
    StringBuffer res = new StringBuffer();
    for (int i = 0; i < s.length(); i+=2*k) {
        StringBuffer temp = new StringBuffer();
        int l = i;
        int r = Math.min(s.length(), i+k);
        temp.append(s.substring(l,r));
        res.append(temp.reverse());
        if (r==s.length()){
            return res.toString();
        }
        int end = Math.min(s.length(), r + k);
        res.append(s.substring(r, end));
    }
    return res.toString();
}
```

