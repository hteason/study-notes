[TOC]

# JVM

## 第2章：Java内存区域与内存溢出异常

## 运行时数据区

 * 共享区
    * 堆(heap)
    * 方法区(method area)

* 线程私有区
  * 虚拟机栈(VM stack)
  * 本地方法栈(native)
  * 程序计数器(PC)



## 线程共享区

### 堆（Heap）

1. 用于存储实例对象，所有的对象实例和数组必须在堆上分配
2. 是垃圾回收器（GC）的主要管理区域，也称“GC堆”

#### 新生代

##### 	Eden空间

##### 	From Survivor空间

##### 	TO Survivor空间

#### 老年代



#### OOM

不断实例化对象，且对象一直被引用不被清除：

~~~java
import java.util.*;
/*
虚拟机参数设置:-Xms:20M -Xmx:20M -XX:+HeapDumpOnOutOfMemoryError
*/
public class HeadOOM{
    static class Inner{}
    
    public static void main(String[] arga){
        //使List一直引用该对象，保证GC Roots有可达路径，避免GC清除对象，使实例对象到达堆最大容量而内存溢出
        List list = new Arrays<Inner>();
        while(true){
            lias.add(new Inner());
        }
    }
} 
~~~





### 方法区（Method Area）

1. 用于存储类信息（类版本、类名、方法）、常量和静态变量等

#### OOM

~~~java
/*
使用cglib无限创建代理类，即动态生成class文件
VM args

-XX:MetaspaceSize=10M
-XX:MaxMetaspaceSize=10M

*/

import java.lang.reflect.Method;

import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

public class MethodAreaOOM {

	public static void main(String[] args) {
		while (true) {
			Enhancer enhancer = new Enhancer();
			enhancer.setUseCache(false);//不使用缓存，防止GC清除
			enhancer.setSuperclass(MySuper.class);
			enhancer.setCallback(new MethodInterceptor() {

				@Override
				public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
					System.out.println("begin...");
					proxy.invokeSuper(obj, args);
					System.out.println("end...");
					return "ok";
				}
			});
			MySuper s = (MySuper) enhancer.create();
			s.doWork();
		}
	}
}

class MySuper {
	void doWork() {
		System.out.println("work...");
	}
}


//Exception in thread "main" java.lang.OutOfMemoryError: Metaspace
~~~







## 线程共享区

### 虚拟机栈

#### 	栈帧

1. 每个方法被执行时都会同时创建出一个栈帧。

2. 栈帧包括局部变量表（存放编译期可知的各种数据类型和对象引用类型）、操作数栈、动态链接、方法出口等信息。

   

   #### 可能出现的2种异常：

​		**(1) StackOverflowError**：线程请求的栈深度大于虚拟机所允许的最大深度。

​		**(2) OutOfMemoryError**：如果虚拟机可以动态扩展，当扩展时无法申请到足够的内存时抛出该异常。

​	

#### SOF

~~~java
/*
无限递归或者调用方法栈超出最大深度
VM args:-Xss128k
*/
public class JavaVMStackSOF{
    int stackHeight = 0;
    void createStack(){
        stackHeight++;
        createStack();
    }
    public static void main(String[] args){
        JavaVMStackSOF sof = new JavaVMStackSOF();
        try{
	        sof.createStack();            
        }cath(Throwable t){
            System.out.println(sof.stackHeight);
            throw e;
        }
    }
}
~~~



### 本地方法栈

1. 为虚拟机栈用到的Native方法服务
2. 和虚拟机栈一样会抛出StackOverflowError和OutOfMemoryError异常



#### OOM

~~~java
/*
无限创建线程，创建线程会调用native的fork函数

注：该段代码会出现系统假死情况!!务必做好其他应用的保存操作

VM args：-Xss2M
*/
public class JavaVMStackOOM {

	private void dontstop() {
		while(true){
			
		}
	}
	
	public void stackLeakByThread() {
		while(true) {
			new Thread(()->dontstop()).start();
		}
	}
	public static void main(String[] args) {
		JavaVMStackOOM oom = new JavaVMStackOOM();
		oom.stackLeakByThread();
	}
}
~~~





### 程序计数器

1. 记录线程执行到了哪一条指令，即所执行字节码的行号指示器，字节码解释器工作时就是把通过改变计数器的值来选取下一条需要执行的字节码指令
2. 如果执行的是Native方法，计数器为空
3. 分支、循环、跳转、异常处理、线程恢复等基础功能都要依赖计数器完成
4. 唯一一个不会出现OutOfMemoryError异常的区域





## 第3章：垃圾收集器(Garbage Collection)与内存分配

### 4种垃圾收集算法

#### 	标记-清除算法（Mark-Sweep）

​	**原理**：将被标记的对象直接删除。

​	**缺点**：容易产生内存空间碎片。



#### 复制算法（Copying）

​	**原理**：将Eden区和Survivor区还存活的对象复制到另一块Survivor区，如果另一块Survivor区没有足够的空间存放存活下来的对象，则移入Old区（**分配担保机制**）。

​	**缺点**：对象存活率高时需要执行较高的复制操作，效率会变低。



####标记-整理算法（Mark-Compact）

​	**原理**：不直接对标记的对象进行清除，而是让所有的对象都移向一端，然后清除掉**端边界外**的内存。



#### 分代收集算法（Generational Collection）

​	**原理**：根据对象的存活周期的不同将Java堆划分为**新生代（Young generation）和老年代(Tenured generation)**，根据各个年代的特点采用最适当的收集算法：

1. 新生代：每次垃圾收集都会有大批的对象死去，只有少数存活，选用**复制算法**；
2. 老年代：存活率普遍较高，没有额外的空间给它担保，则使用**标记-整理MC算法**或**标记-清除MS算法**。



### 垃圾收集器

#### 	新生代收集器

##### 	Serial

​	串行回收，单线程，执行时会挂起其他所有线程（Stop The World）

##### 	ParNew

​	Serial的多线程版本

##### 	Parallel Scavenge

​	并行回收，不会Stop The World，吞吐量优先



#### 老年代收集器

##### 	Serial Old

​	老年代串行回收

​	优点：

​	缺点：

##### 	Parallel Old

​	老年代并行回收，配合Parallel Scavenge实现吞吐量优先的收集器组合

​	优点：

​	缺点：

##### 	CMS

​	基于标记-清除（MS）算法，一种获取最短停顿时间为目标的收集器

- 初始标记（CMS Initial mark）

  - 标记GC Roots可达的老年代对象
  - 遍历新生代对象，标记可达的老年代对象

- 并发标记(CMS concurrent mark)

- 重新标记(CMS remark)

- 并发清除(CMS concurrent sweep)

  ​	优点：

  ​	缺点：

#### Java堆收集器

##### 	G1（Garbage First）

​	全能收集器，跨越了新生代和老年代，把两个区域划分为一个个相同大小的块，优先（First）收集垃圾最多的块。

​	优点：

​	缺点：





