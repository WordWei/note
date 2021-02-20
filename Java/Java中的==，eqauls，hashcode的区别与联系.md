- == ： 该操作符生成的是一个boolean结果，它计算的是操作数的值之间的关系
- equals ： Object 的 实例方法，比较两个对象的content是否相同
- hashCode ： Object 的 native方法 , 获取对象的哈希值，用于确定该对象在哈希表中的索引位置，它实际上是一个int型整数 

### 关系操作符 ==

 Java中有八种**基本数据类型**： 

- 浮点型：float(4 byte), double(8 byte)
- 整型：byte(1 byte), short(2 byte), int(4 byte) , long(8 byte)
- 字符型: char(2 byte)
- 布尔型: boolean(JVM规范没有明确规定其所占的空间大小，仅规定其只能够取字面值”true”和”false”)

对于这八种基本数据类型的变量，变量直接存储的是“值”。因此，在使用关系操作符 == 来进行比较时，比较的就是“值”本身。要注意的是，浮点型和整型都是有符号类型的（最高位仅用于表示正负，不参与计算【以 byte 为例，其范围为 -2^7 ~ 2^7 - 1，-0即-128】），而char是无符号类型的（所有位均参与计算，所以char类型取值范围为0~2^16-1）。 

**引用类型变量**

 Java中，引用类型的变量存储的并不是“值”本身，而是**与其关联的对象在内存中的地址。** 

```java
String str1;
```

 声明了一个引用类型的变量，此时它并没有和任何对象关联。 而通过 new 来产生一个对象，并将这个对象和str1进行绑定： 

```java
str1= new String("hello");
```

 str1 就指向了这个对象，此时引用变量str1中存储的是它指向的对象在内存中的存储地址，并不是“值”本身，也就是说并不是直接存储的字符串”hello”。这里面的引用和 C/C++ 中的指针很类似。 

> - 若操作数的类型是基本数据类型，则该关系操作符判断的是左右两边操作数的值是否相等
> - 若操作数的类型是引用数据类型，则该关系操作符判断的是左右两边操作数的内存地址是否相同。也就是说，若此时返回true,则该操作符作用的一定是同一个对象。

### equals

在Object类中，是这样定义equals()方法的

```java
public boolean equals(Object obj){
    return (this == obj);
}
```

 明显是比较指针(地址)...

 为什么会有equals是比较内容的这种说法呢？ 

 因为在String、Double等封装类中，已经重写(overriding)了Object类的equals()方法，于是有了另一种计算公式，是进行内容的比较。 

在String类中，是这样定义equals()方法的

```java
 public int hashCode() { 
    int h = hash; 
    if (h == 0) { 
        char val[] = value; 
        int len = count; 
        for (int i = 0; i < len; i++) { 
            h = 31*h + val[off++]; 
        } 
        hash = h; 
    } 
    return h; 
} 
```

### hashCode

在Object类中的定义为:  返回的对象的地址值。 

```java
public native int hashCode();
```

在String类中的定义为:   在String等封装类中对此方法进行了重写。方法调用得到一个计算公式得到的 int值. 

```java
public int hashCode() {
    int h = hash;
    if (h == 0 && value.length > 0) {
        char val[] = value;

        for (int i = 0; i < value.length; i++) {
            h = 31 * h + val[i];
        }
        hash = h;
    }
    return h;
}
```

### hashCode和equlas的关系

-  如果两个对象相同（即用equals比较返回true），那么它们的hashCode值一定要相同； 
-  如果两个对象的hashCode相同，它们并不一定相同(即用equals比较返回false) 
- 原因：从散列的角度考虑，不同的对象计算哈希码的时候，可能引起冲突，大家一定还记得数据结构中。

自我的理解：由于为了提高程序的效率才实现了hashcode方法，先进行hashcode的比较，如果不同，那没就不必在进行equals的比较了，这样就大大减少了equals比较的 次数，这对比需要比较的数量很大的效率提高是很明显的，一个很好的例子就是在集合中的使用；

我们都知道java中的List集合是有序的，因此是可以重复的，而set集合是无序的，因此是不能重复的，那么怎么能保证不能被放入重复的元素呢，但靠equals方法一样比较的 话，如果原来集合中以后又10000个元素了，那么放入10001个元素，难道要将前面的所有元素都进行比较，看看是否有重复，欧码噶的，这个效率可想而知，因此hashcode 就应遇而生了，java就采用了hash表，利用哈希算法（也叫散列算法），就是将对象数据根据该对象的特征使用特定的算法将其定义到一个地址上，那么在后面定义进来的数据 只要看对应的hashcode地址上是否有值，那么就用equals比较，如果没有则直接插入，只要就大大减少了equals的使用次数，执行效率就大大提高了。 继续上面的话题，为什么必须要重写hashcode方法，其实简单的说就是为了保证同一个对象，保证在equals相同的情况下hashcode值必定相同，如果重写了equals而未重写 hashcode方法，可能就会出现两个没有关系的对象equals相同的（因为equal都是根据对象的特征进行重写的），但hashcode确实不相同的。





> 本文转自：  https://www.fangzhipeng.com/javainterview/2019/02/23/equals-hashcode.html 