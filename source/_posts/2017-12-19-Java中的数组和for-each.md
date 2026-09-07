---
title: Java中的数组和for-each
date: 2017-12-19 01:41:39
tags: Java
description: 帮室友debug时发现自己并没有理解的一个知识点
---

# Problem

室友的代码

```java
class Data {
    int weight;
}
Data[] dataArray = new Data[10];
for (Data data : dataArray) {
    data.weight = 10;
}
```

其中的错误涉及了数组和for-each遍历

# Create an Array

```java
int[] array = new int[3];
```

所做的事情仅仅是创建3个int所需的空间，这部分空间内数据清零，然后将首个空间的地址赋给array

对原生类型(primitive type)数组来说，可以理解为数组内的默认值为0/false

但是对引用类型(reference type)数组来说，就要注意，数组内每个元素（数组内的元素均为引用）的值都是null，编译器并不会帮你调用默认构造函数

# For-each

### Primitive Type

对原生类型的数组(array)或容器(collection)使用for-each进行遍历，编译器会复制出一个temp值，无法改变原数组或容器内元素的值

```java
// primitive type in for-each
int[] array1 = new int[3];
	for (int value : array1) {
	value = 1;
}
for (int value : array1) {
	System.out.print(value + " ");
}
// output: 0 0 0
```

### Change of Reference Type

与原生类型相反，对引用类型的数组或容器使用for-each进行遍历，可以改变原数组或容器内元素的值

```java
class Nico {
    int i;
    Nico (int i) { 
        this.i = i;
    }
}
```
```java
// change of reference type in for-each
Nico[] array2 = new Nico[3];
for (int i = 0; i < array2.length; ++i) {
	array2[i] = new Nico(i);
}
for (Nico value : array2) {
	value.i = 25252;
}
for (Nico value : array2) {
	System.out.print(value.i + " ");
}
// output: 25252 25252 25252
```

### Assignment or Initialization of Reference Type

当然，也不能只记着“引用类型会改变数组或容器原来的值”

你应该注意到了，我在前一个例子中对数组赋值时使用的是含index的普通for循环，因为在for-each循环中对元素进行赋值/初始化操作同样无法完成

```java
// assignment of reference type in for-each
Nico[] array3 = new Nico[3];
for (Nico value : array3) {
	value = new Nico(25252);
}
for (Nico value : array3) {
	System.out.print(value + " ");
}
// output: null null null
```

### Analysis

假设引用类型的for-each循环的本质和原生类型的相同，也是编译器声明了一个temp变量，进行验证

```java
class Nico {
    int i;
    Nico (int i) { 
        this.i = i;
        System.out.println("Nico()");
    }
}
```

```java
System.out.println("New Array:");
Nico[] array4 = new Nico[3];
System.out.println("For Loop with Index:");
for (int i = 0; i < array4.length; ++i) {
	array4[i] = new Nico(i);
}
System.out.println("For-Each Loop:");
int i = 0;
for (Nico value : array4) {
	System.out.println(array4[i] + " " + value);
	i++;
}
// output:
// New Array:
// For Loop with Index:
// Nico()
// Nico()
// Nico()
// For-Each Loop:
// Nico@15db9742 Nico@15db9742
// Nico@6d06d69c Nico@6d06d69c
// Nico@7852e922 Nico@7852e922
```

可以看到

1. 新建数组时构造函数并没有被调用
2. `value` 和 `array4[i]` 是对同一个对象的不同引用

所以，如果你想用for-each循环进行赋值或初始化的话，结果是 `value` 指向了新的值，但是 `array4[i]` 还指向旧的对象

```java
Nico[] array5 = new Nico[1];
for (Nico value : array5) {
    System.out.println(array5[0] + " " + value);
    // array[0] -> null
    // value -> null

    value = new Nico(25252);

    System.out.println(array5[0] + " " + value);
    // array[0] -> null
    // value -> Nico@xxxxxxxx
}
// output:
// null null
// Nico()
// null Nico@15db9742
```

