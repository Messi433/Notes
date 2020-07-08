# Java基础

## 第四章-对象与类

多态：https://www.runoob.com/java/java-polymorphism.html

#### 

## 第九章-集合框架

### 弱项

- **HashSet、TreeSet**
- 迭代器
- 泛型
  - 自定义泛型
  - 泛型通配符
- 

### 9.1 集合概述

数组与集合的区别

### 9.2 集合框架

#### 9.2.1 Collection

单列集合 collection接口

​	list 重复、有序

​		ArrayList、LinkList

​	set

​		**HashSet、TreeSet**

Collection 常用API

双列集合 Map

#### 9.2.2 Iterator迭代器

### 9.3 泛型

9.3.1 泛型的引出

9.3.2 泛型的好处

9.3.3泛型的定义与使用

​		含有泛型的类

​		含有泛型的方法

​		含有泛型的接口

9.3.4 泛型通配符

​	通配符基本使用

​	通配符高级使用

### 9.4 综合案例1-斗地主

### 9.5 数据结构

​	红黑树

### 9.6 List接口

- 重复、有序

#### 9.6.1 List集合

1.List接口

2.List接口常用方法

#### 9.6.2 List的子类

1.LinkedList

2.ArrayList

### 9.7 Set接口

- 不重复、无序→去重

#### 9.7.1 HashSet

##### 1.HashSet认识

##### 2.HashSet结构

​	哈希表：查询速度快

​	哈希值 hashcode：逻辑地址，系统随机给

​	HashSet去重操作

​		添加值add()通过hashcode、equals()字符串比较，确定存储对象的桶位置(hash表位置)→**先比较hashcode->冲突再比较字符串equals()**，若字符串同则认定重复

​	HashMap实现原理：

​		不同对象相同hashcode处理方法：由于不同对象可能有相同hash值(数据结构中的hash冲突)、把相同hashcode的对象通过拉链法拉到一起，(1.8之前是链表+数组、1.8之后是链表+数组+红黑树(拉链长度超过8，则用红黑树))->从而确定对象的桶的唯一位置。

##### 3.hashset存储自定义元素

​		给HashSet中存放自定义类型元素时，需要重写对象中的hashCode和equals方法，建立自己的比较方式，才能保证HashSet集合中的对象唯一

```java
//实体类对象重写两个方法
@Override
public boolean equals(Object o) {
  if (this == o) return true;
  if (o == null || getClass() != o.getClass()) return false;
  Student student = (Student) o;
  return age == student.age &&
    Objects.equals(name, student.name);
}
@Override
public int hashCode() {
  return Objects.hash(name, age);
}
```

```java
/**
 * 自定义类型对象，若对象属性相同，必须重写hashCode() equals(),否则默认这两个元素对象不同，不会去重
 */
public class HashSetTest2 {
    public static void main(String[] args) {
//创建集合对象 该集合中存储 Student类型对象
        HashSet<Student> stuSet = new HashSet<Student>(); //存储
        Student stu = new Student("于谦", 43);
        stuSet.add(stu);
        stuSet.add(new Student("郭德纲", 44));
        stuSet.add(new Student("于谦", 43));
        stuSet.add(new Student("郭麒麟", 23));
        stuSet.add(stu);
        for (Student stu2 : stuSet) {
            System.out.println(stu2);
        }
    }
}
```

##### 4.LinkedHashSet(唯一且有序)

我们知道HashSet保证元素唯一，可是元素存放进去是没有顺序的，那么我们要**保证有序**，怎么办呢? 在HashSet下面有一个子类 `java.util.LinkedHashSet` ，它是链表和哈希表组合的一个数据存储结构。 演示代码如下:



##### 5.可变参数

在JDK1.5之后，如果我们定义一个方法**需要接受多个参数，并且多个参数类型一致**，我们可以对其简化成如下格

式:
 `修饰符 返回值类型 方法名(参数类型... 形参名){ }`

其实这个书写完全等价与
 `修饰符 返回值类型 方法名(参数类型[] 形参名){ }`

只是后面这种定义，在调用时**必须传递数组，而前者可以直接传递数据即可**。 JDK1.5以后。出现了简化操作。... 用在参数上，称之为可变参数。

### 9.8 Collections

集合工具类

### 9.9 Map集合

#### 9.9.1 概述

#### 9.9.2 Map常用子类

9.9.3