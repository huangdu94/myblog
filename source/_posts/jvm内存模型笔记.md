---
title: jvm内存模型笔记
date: 2019/12/14
updated: 2019/12/14
comments: true
categories: 
- [笔记, java, jvm内存模型]
tags: 
- java笔记
- jvm
- 内存模型
---
## jvm内存模型

### 〇、概述
+ JVM将其内存数据主要分为以下五个部分：
+ 一、程序计数器
+ 二、虚拟机栈
+ 三、本地方法栈
+ 四、Java堆
+ 五、方法区

### 一、程序计数器
+ 程序计数器(Program Counter Register)是一块很小内存空间。
+ 每一个线程有一个私有的程序计数器，用于记录下一条要运行的指令。  
+ 如果当前线程正在执行一个Java方法，则程序计数器记录正在执行的Java字节码地址。
+ 如果当前线程正在执行一个Native方法，则程序计数器为空。

### 二、Java虚拟机栈
+ Java虚拟机栈也是线程私有的内存空间，它和Java线程在同一时间创建。
+ 它保存方法的局部变量、部分结果，并参与方法的调用和返回。  
+ 有两种异常与栈空间有关：`StackOverflowError`和`OutOfMemoryError`。
+ 虚拟机栈在运行时使用一种叫做栈帧的数据结构保存上下文数据(其中存放了方法的局部变量表、操作数栈、动态连接方法和返回地址等信息)。
+ 使用jclasslib工具可以对class文件做较为深入的研究。

### 三、本地方法栈
+ 与Java虚拟机栈功能相似，Java虚拟机栈用于管理Java函数的调用，本地方法栈用于管理本地方法的调用。
+ 在SUN的Hot Spot虚拟机中，不区分本地方法栈和虚拟机栈。

### 四、Java堆
+ Java堆是Java运行时内存中最为重要的部分，被JVM中所有线程共享，几乎所有的对象(数组也算对象)都是在堆中分配空间的。
+ Java堆分为新生代和老年代两个部分，新生代用于存放刚刚产生的对象和年轻的对象，如果对象一直没有被回收，生存得足够长，老年对象就会被移入老年代。
+ 新生代又可进一步细分为eden(伊甸园)、survivor space0(s0或者from space)和survivor space1(s1或者to space)。
+ Full GC(`System.gc()`)后，新生代空间被清空，未被回收的对象全部被移入老年代。

### 五、方法区
+ 方法区也称为永久区，虽然叫做永久区，其中的对象也是可以被GC回收的。
+ 方法区也是被JVM中所有线程共享，主要保存的信息是类的元数据(包括类的类型信息、常量池、域信息、方法信息)。
+ Hot Spot虚拟机对常量池的回收策略是明确的，只要常量池中的常量没有被任何地方引用，就可以被回收。
+ Hot Spot虚拟机确认某一个类信息不会被使用，也会将其回收。回收的基本条件至少有：所有该类的实例被回收，且装载该类的`ClassLoader`被回收。

### 六、jvm参数
```java
/**
 * JVM参数
 * -client -server
 * -ea 开启assert断言机制
 * -verbose:gc 显示gc过程
 * -XX:+PrintGCDetails 显示gc详情
 *
 * -XX:+UseSpinning 开启自旋锁
 * -XX:PreBlockSpin 设置自旋锁的等待次数
 *
 * -XX:+DoEscapeAnalysis 逃逸分析
 * -XX:+EliminateLocks 锁消除
 * -XX:-DoEscapeAnalysis 关闭逃逸分析
 * -XX:-EliminateLocks 关闭锁消除
 *
 * 虚拟机栈大小设置
 * -Xmx 最大堆内存   例-Xmx20M
 * -Xms 最小堆内存
 * -Xmn 设置新生代的大小
 * (-XX:NewSize 新生代初始值 -XX:MaxNewSize 新生代最大值 例-XX:NewSize=10M)
 * -XX:PermSize 方法区初始大小 -XX:MaxPermSize 方法区最大值
 * -Xss 设置线程栈的大小
 *
 * -XX:SurvivorRatio 新生代中 eden/s0 或 eden/s1(s0 和 s1大小相等)
 * -XX:NewRatio 老年代/新生代
 *
 * -XX:MinHeapFreeRatio 设置堆空间最小空闲比例
 * -XX:MaxHeapFreeRatio 设置堆空间最大空闲比例
 * -XX:TargetSurvivorRatio 设置survivor区的可使用率(当survivor区的空间使用率达到这个数值时，会将对象送入老年代)
 *
 * @author duhuang@iflytek.com
 * @version 2019/12/14 11:43
 */
public class JvmDemo {
    public static void main(String[] args) {
        System.out.println("Hello World.");
    }
}
```