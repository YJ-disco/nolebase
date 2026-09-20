# Java多态原理 - JVM的静态分派和动态分派
## 0. 前言

多态是面向对象编程模型中一个核心概念，它可以帮助我们写出更具有弹性的代码。相信每个Java开发者都对多态的使用非常熟练，不过可能大多人对于”多态”这一概念的理解仅是浮于表面，对它内部的调用过程以及实现原理缺乏更深一步的认识。本文将对多态的实现原理抽丝剥茧，带大家深入理解多态。

## 1.多态的种类

多态(Polymorphism)这个术语在不同的上下文中会有不同的含义。在类型学说(Type Theory)中，多态分为好几个种类，其中最常见的有3种类型，分别为`Ad hoc polymorphism`、`Subtyping`,`Parametric polymorphism` ，而Java实现了这3种类型的多态；在OOP(面向对象编程)中，我们常说的多态指的是类型学说中的Subtyping。

为了方便理解多态调用原理，本文着重介绍前两者。后者因为涉及到类型擦除和单态化(monomorphized)，碍于篇幅，不进行详细介绍。如果读者想系统了解请点击维基百科原词条[Polymorphism (computer science)](https://en.wikipedia.org/wiki/Polymorphism_(computer_science))。

- _[Ad hoc polymorphism](https://en.wikipedia.org/wiki/Ad_hoc_polymorphism)_
    
    在Java中，方法重载(Method Overloading)属于Ad hoc polymorphism。在这种模式下，我们使用同一个方法名字和返回值，不同的方法参数和类型来区分不同的方法。比如String类中，多个valueOf使用不同的参数类型区分。
    
    ```
    //code 1-1
    public static String valueOf(boolean b) {
      return b ? "true" : "false";
    }
      
    public static String valueOf(char c) {
      char data[] = {c};
      return new String(data, true);
    }
    ```
    
- _[Parametric polymorphism](https://en.wikipedia.org/wiki/Parametric_polymorphism)_
    
    我们可以把Parametric polymorphism理解为泛型编程
    
    ```
    //code 1-2
    ArrayList<String> list = new ArrayList<>();
    list.add("generic programming");
    ```
    
- _[Subtyping](https://en.wikipedia.org/wiki/Subtyping)_
    
    Subtyping即是我们最熟悉的一种多态，它描述的是subtype和supertype之间的一种可替换关系，比如下面代码
    
    ```
    //code 1-3
    Charsequence s = new String("bigcat");
    ```
    
    String实现了Charsequence，在这里我们说String是Charsequence的subtype(String is a subtype of Charsequence)。
    
    Subtyping类似于集合中的”包含关系(⊆)”。在上面的例子中，可以理解为String是Charsequence的子集。因为它继承自Charsequence，因此使用Charsequence来表达String满足类型安全。
    

## 2.方法分派(Method Dispatch)

多态带来了许多好处的同时也引入了一些问题，比如在进行方法调用时，我们如何确定调用哪一个版本的方法呢? 如同上一个小节描述的Overloading，String内部存在多个valueOf方法;同样，Charsequence和String都存在签名(Method Signature)完全一样的方法(比如`length`、`charAt`等)，我们需要选中其中一个仅且一个版本的方法进行调用。 这个选择某个版本方法的过程就称为方法分派。按照类型划分，又有静态分派和动态分派两种，下面将分别介绍。

### 2.1 静态分派(Static Dispatch)

静态分派是指**选择方法的过程发生在编译期**，它主要实现了Ad hoc polymorphism和Parametric polymorphism(即overloading和generic)。考虑下面代码

```
//code 2-1
public class StaticDispatch {

    public void speak(Animal animal) {
        System.out.println("undefined...");
    }

    public void speak(Cat cat) {
        System.out.println("miao...");
    }

    public void speak(Dog dog) {
        System.out.println("wang...");
    }

    public static void main(String[] args) {
        Animal cat = new Cat();
        Animal dog = new Dog();
        StaticDispatch speaker = new StaticDispatch();
        speaker.speak(cat);
        speaker.speak(dog);
        speaker.speak((Cat)cat);
        speaker.speak((Dog)dog);
    }
}
```

上面代码最终输出的结果为

```
# undefined...
# undefined...
# miao...
# wang...
```

对于熟悉Java的方法重载的朋友可能很轻易就猜到这个输出结果。在上面的代码中，cat和dog的类型定义为Animal，因为Subtyping，这是完全合法的。但编译器在**编译期**无法得知cat和dog的实际类型(实际指向内存哪个对象)，从上面的结果能看出编译器在**编译期**只能根据参数的静态类型去选择某个方法的实现。

为了加深对静态分派的理解，我们查看StaticDispatch的字节码，通过命令`javap -v org.moonto.java.StaticDispatch`查看字节码的详细信息。

```
//code 2-2
//注: 为了方便阅读删减了部分不影响阅读的内容

public class org.moonto.java.StaticDispatch
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #16.#41        // java/lang/Object."<init>":()V
   #2 = Methodref          #42.#43        // org/moonto/java/Animal.speak:()V
   #3 = Fieldref           #44.#45        // java/lang/System.out:Ljava/io/PrintStream;
   #4 = String             #46            // miao...
   #5 = Methodref          #47.#48        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #6 = String             #49            // wang...
   #7 = Class              #50            // org/moonto/java/Cat
   #8 = Methodref          #7.#41         // org/moonto/java/Cat."<init>":()V
   #9 = Class              #51            // org/moonto/java/Dog
  #10 = Methodref          #9.#41         // org/moonto/java/Dog."<init>":()V
  #11 = Class              #52            // org/moonto/java/StaticDispatch
  #12 = Methodref          #11.#41        // org/moonto/java/StaticDispatch."<init>":()V
  #13 = Methodref          #11.#53        // org/moonto/java/StaticDispatch.speak:(Lorg/moonto/java/Animal;)V
  #14 = Methodref          #11.#54        // org/moonto/java/StaticDispatch.speak:(Lorg/moonto/java/Cat;)V
  #15 = Methodref          #11.#55        // org/moonto/java/StaticDispatch.speak:(Lorg/moonto/java/Dog;)V
{
  public org.moonto.java.StaticDispatch();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 3: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lorg/moonto/java/StaticDispatch;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=4, args_size=1
         0: new           #7                  // class org/moonto/java/Cat
         3: dup
         4: invokespecial #8                  // Method org/moonto/java/Cat."<init>":()V
         7: astore_1
         8: new           #9                  // class org/moonto/java/Dog
        11: dup
        12: invokespecial #10                 // Method org/moonto/java/Dog."<init>":()V
        15: astore_2
        16: new           #11                 // class org/moonto/java/StaticDispatch
        19: dup
        20: invokespecial #12                 // Method "<init>":()V
        23: astore_3
        24: aload_3
        25: aload_1
        26: invokevirtual #13                 // Method speak:(Lorg/moonto/java/Animal;)V
        29: aload_3
        30: aload_2
        31: invokevirtual #13                 // Method speak:(Lorg/moonto/java/Animal;)V
        34: aload_3
        35: aload_1
        36: checkcast     #7                  // class org/moonto/java/Cat
        39: invokevirtual #14                 // Method speak:(Lorg/moonto/java/Cat;)V
        42: aload_3
        43: aload_2
        44: checkcast     #9                  // class org/moonto/java/Dog
        47: invokevirtual #15                 // Method speak:(Lorg/moonto/java/Dog;)V
        50: return
      LineNumberTable:
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      51     0  args   [Ljava/lang/String;
            8      43     1   cat   Lorg/moonto/java/Animal;
           16      35     2   dog   Lorg/moonto/java/Animal;
           24      27     3 speaker   Lorg/moonto/java/StaticDispatch;
}

```

观察`code 2-2`的字节码，留意main方法编号为26和31的指令，这两条指令都是`invokevirtual #13`，其中`#13`指的是常量池中index为13的常量，即speak(Animal)的方法签名: `speak:(Lorg/moonto/java/Animal;)V`。

同样，再观察编号39和47的指令，分别为`invokevirtual #14`和`invokevirtual #15`，而常量池中14和15代表的是`speak:(Lorg/moonto/java/Cat;)V`、`speak:(Lorg/moonto/java/Dog;)V`。

`code 2-2`的字节码能看出调用重载方法在编译期就已经选择好了某个版本的方法。因此在上面的例子中，cat和dog的实际类型并不会影响方法的选择，编译器只能根据它定义的类型进行方法选择。

### 2.2 动态分派

和静态分派相反，动态分派即**方法选择的过程发生在运行时(Runtime)**，因为有些信息在编译时无法被确定(引用在编译器的角度看纯粹是一串符号，需要等待运行时才会被解析为实际引用)。考虑下面的代码:

```
//code 2-3
public class Animal {
  public void speak() {
    System.out.println("undefined");
  }
  
  public void run() {
    
  }
}

public class Dog extends Animal{
    @Override
    public void speak() {
        System.out.println("wang..");
    }
}

public class Cat extends Animal{
    @Override
    public void speak() {
        System.out.println("miao..");
    }
}

public class DynamicDispatch {

  public static void main(String[] args) {
    DynamicDispatch dispatcher = new DynamicDispatch();
    dispatcher.doDispatch(new Cat());
    dispatcher.doDispatch(new Dog());
  }

  public void doDispatch(Animal animal) {
    animal.speak();
  }
}
```

上面代码的执行结果最终输出

```
#miao..
#wang..
```

对于任何一个有OOP基础的读者，都能猜到这个输出结果。按照我们对静态分派的理解，如果此时根据静态类型来决定调用方法，那么显然应该输出”undefined”。但上面的例子对方法进行调用时，显然是根据引用的**实际类型**选择方法，我们把这种分派逻辑称为动态分派。

为了深入理解这个原则，同样看一下DynamicDispatch的字节码

```
//code 2-4
public void doDispatch(org.moonto.java.Animal);
  descriptor: (Lorg/moonto/java/Animal;)V
  flags: ACC_PUBLIC
  Code:
    stack=1, locals=2, args_size=2
       0: aload_1
       1: invokevirtual #9                  // Method org/moonto/java/Animal.speak:()V
       4: return
    LineNumberTable:
      line 11: 0
      line 12: 4
    LocalVariableTable:
      Start  Length  Slot  Name   Signature
        0       5     0  this   Lorg/moonto/java/DynamicDispatch;
        0       5     1 animal   Lorg/moonto/java/Animal;
```

上面的字节码中调用方法的指令为`invokevirtual #9`，后面的注释也标注了调用的方法是Animal的方法签名，那么为什么最终会选择了Cat和Dog的speak方法呢？这是因为分派逻辑由`invokevirtual`指令定义。

Java虚拟机规范中规定了`invokevirtual`指令的逻辑，如下:

> Let C be the class of _objectref_. The actual method to be invoked is selected by the following lookup procedure:
> 
> - If C contains a declaration for an instance method `m` that overrides ([§5.4.5](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-5.html#jvms-5.4.5)) the resolved method, then `m` is the method to be invoked, and the lookup procedure terminates.
> - Otherwise, if C has a superclass, this same lookup procedure is performed recursively using the direct superclass of C; the method to be invoked is the result of the recursive invocation of this lookup procedure.
> - Otherwise, an `AbstractMethodError` is raised.

为了方便理解，这里就不逐字翻译上面的内容(翻译水平差)。它大概要表达的意思如下:

假设C是引用objectref所属的类，实际要调用的方法遵循以下的查找过程:

1. 如果C中存在一个方法`m`,它重写(Override)了被解析的方法，那么直接调用此方法，结束查找过程。
2. 如果第一步中找不到此方法，且C存在父类(superclass)，那么对它的父类重复第一步的查找过程。
3. 如果都找不到，则抛出AbstractMethodError。

`invokevirtual`指令定义可以看出我们平时熟悉的多态(Subtyping)调用原理是由这条指令本身的逻辑提供的。

需要注意的是，虽然JVM的规范(Specification)这样规定`invokevirtual`指令的查找过程，但实际上JVM的实现(Implementation)只需要实现规范中要求的功能即可，并不一定完全按照这种死板的方式查找。

### 2.3 单分派和多分派

上面的分派逻辑中，有两个因素会影响方法分派的结果，即根据**方法的接收者(method receiver，被调用对象本身)的类型**和**参数类型**进行分派。

**只根据方法接收者的实际类型进行分派的称为单分派; 根据方法接收者，参数等多个组合进行分派的称为多分派**。在Java中，方法调用都是属于单分派(注2)。因为JVM只会根据方法接收者的实际类型进行分派，而被调用的方法(签名)已经在编译期就被确定，参数的实际类型在运行时再也无法影响方法的选择过程。

**注1**: 在OOP中，我们把方法调用称为给对象发送消息，被调用对象本身是消息接受者，因此称为方法接收者。

**注2**: 关于Java的静态分派是否属于多分派可能存在争议，在《深入理解Java虚拟机》这本书中，作者认为静态分派属于多分派。大概是因为Java静态分派在选择方法的过程确实会根据静态类型进行选择。个人觉得这种解释有待商榷。对于读者而言，理解这个概念，以及为什么会有这种争议更加重要。

## 3. JVM运行时内存区域

上一小节介绍了多态调用的一部分原理，不过我们对方法以什么样的方式存储在内存中仍是非常抽象，要想彻底理解方法分派原理，必须要弄懂方法在内存中的布局。在正式开始之前，我们先复习一下Java虚拟机运行时的内存区域。

### 3.1 内存区域

Java虚拟机把内存划分出几个不同的区域，分别为method area、heap、java stacks、pc registers、native method stacks，它们分别负责存放不同类型的数据。

![jvm's runtime memory area](https://user-images.githubusercontent.com/3600657/175875845-c9a742f8-b393-45f9-ad6b-0c921f121288.png)

如上图所示，对于理解JVM的朋友会非常熟悉。在本文中，我们将重点放在method area、heap、java stacks这3个区域上。

- Stacks (Java Virtual Machine Stacks)
    
    每一个JVM的线程都拥有一块独立的内存叫Stacks，它伴随着这个线程的生命周期。对于这块内存，主要用于存储栈帧(Stack Frame)。在本文后续的介绍中，将会看到进行方法调用之前，需要先把对象的引用(objectref)push到栈帧的操作栈(oprand stack)中。
    
- Heap
    
    Heap是一块被所有线程共享的内存区域，所有对象(即class instances)都在这块区域分配内存。在后续的介绍中，我们将会看到普通对象在这块区域中的内存布局。
    
- Method Area
    
    Method Area也是一块被所有线程共享的区域，主要用来存储类的信息。比如类结构中的常量池(constant pool)，方法数据(method data)等。
    

### 3.2 类的结构和类加载

Java的源代码会被编译成一个class文件，它包含我们定义的常量、方法等等信息，结构如下

```
ClassFile {
    u4             magic;
    u2             minor_version;
    u2             major_version;
    u2             constant_pool_count;
    cp_info        constant_pool[constant_pool_count-1];
    u2             access_flags;
    u2             this_class;
    u2             super_class;
    u2             interfaces_count;
    u2             interfaces[interfaces_count];
    u2             fields_count;
    field_info     fields[fields_count];
    u2             methods_count;
    method_info    methods[methods_count];
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```

当类被ClassLoader加载时，类的信息就会按照某种格式存放到内存中的方法区

![jvm's classloading](https://user-images.githubusercontent.com/3600657/175875859-4d4a86ad-2017-4e82-8054-464259b322b4.png)

![image-20220626131609198](https://user-images.githubusercontent.com/3600657/175875863-b87cad03-1369-4685-a5bb-251ca6a2a9fa.png)

## 4. 方法的内存布局

Java虚拟机规范并没有强制虚拟机的实现者应该要如何实现方法的内存布局，它把具体实现细节交给实现虚拟机的开发者。虚拟机的实现只需要保证`invokevirtual`这条指令的定义的语义即可。

### 4.1 虚方法(Virtual method)

`invokevirtual`这条指令实际上调用的是虚方法，在正式开始介绍方法布局之前，有必要先了解一下虚方法。

虚方法又叫虚函数(Virtual function)，学习过C++的读者应该很清楚这个概念。简单来说，虚方法就是那些可以被继承(inheritable)、重写(Overridable)的方法。因为这些特性，这些方法在编译期无法确定调用的版本，只能在运行时进行动态分派。

Java中的非static、final、private的方法就属于虚方法，它们无法在编译期就被确定方法的版本。反过来说，被static、final、private修饰的方法，因为不可被继承和重写，在类加载时就可以确定方法的版本，无需进行动态分派。比如private方法调用时使用的是`invokespecial`指令而不是`invokevirtual`。

更多关于虚方法的介绍，[点击进入](https://en.wikipedia.org/wiki/Virtual_function)维基百科页面。

### 4.2 虚方法表(Virtual method table)

Virtual method table别名很多，也叫virtual function table，vtable等。为了方便，下面统一使用vtable。

回想`invokevirtual`这条指令的定义，它存在递归查找过程。动态分派是极为频繁的操作，如果每次分派都递归查找方法显然效率非常低下。vtable在内存中为每个类都维护了一个方法表(method table)，可以把该表看作是一个装着指针的数组，指针指向了具体的方法数据。

虚方法表中记录了类自己拥有的方法以及它从superclass中继承过来的方法。

![virtual method table](https://user-images.githubusercontent.com/3600657/175875872-9307132b-cba8-4b0b-8db2-021635fee1af.png)

对于从superclass中继承的方法，如果subclass对它进行了重写(Orverride)，那么方法表中的指针会指向subclass的方法数据; 如果subclass没有重写，指针指向superclass的方法数据。

![virtual method table](https://user-images.githubusercontent.com/3600657/175875890-46a24b7d-4e8a-4257-af8a-8f3b1d03e246.png)

上图是Dog的方法表，深灰色部分是从Object继承过来的方法，Dog类没有重写这部分方法，因此方法表中存放的只指向Object方法数据的指针; 浅灰色的run也是从Animal继承过来的方法，这部分也没有重写，所以存放的是指向Animal的指针; 白色部分也是从Animal继承的方法，因为Dog内部重写了该方法，所以这里是指向Dog方法数据的指针。

值得注意的是，虚方法表只会记录虚方法，非虚方法因为在类加载阶段就可以把符号引用转化为直接引用，因此并不需要记录在虚方法表中。换而言之，非虚方法无需动态分派。

通过方法表可以优化`invokevirtual`指令中的递归查询从而提高查询性能，同时也能间接实现`invokevirtual`指令的要求。

## 5. 对象的结构

理解了方法在内存的布局，最后一步就是理解一个对象是如何查找到方法表中的数据。当Java虚拟机遇到new指令时，就会触发类加载，完成后给对象分配内存，对象的格式如下

```
#The layout of regular objects in memory

+---------------+	  +---------------+
| Object Header |  --->>  |   mark word   | 
+---------------+         +---------------+       
| Instance Data |	  | klass pointer | --->> ptr to special structure
+---------------+         +---------------+
|    Padding    |	
+---------------+ 

```

对象在内存中主要分为3部分结构，分别是Header、Data、Padding。

- Object Header
    
    - mark word
        
        mark word对于熟悉Java多线程机制和垃圾回收的朋友会比较熟悉，这里存放了对象的GC状态、锁、hashcode等等信息。
        
    - klass pointer
        
        klass pointer是一个指向另一个描述了当前对象方法的布局的指针。简单来说，就是一个指向了vtable的一个指针。
        
- Instance Data
    
    这部分数据是对象的属性，包括对象从父类中继承过来的属性
    
- Padding
    
    这部分主要用作内存对齐，因为Instance Data部分数据大小不确定，因此需要添加额外的字节做内存对齐。
    

下面是一个更完整的展示对象结构的图

![](https://user-images.githubusercontent.com/3600657/175878256-2690842b-63ad-47b5-b5d6-8374306399a1.jpg)

仔细回顾`code 2-4`中的字节码，在执行`invokevirtual`指令之前，还执行了aload_1指令，这条指令的的作用是从栈帧中的Local Variable中加载slot为1的变量，并push到操作栈(oprand stack)。

`code 2-4`中slot为1的变量正是animal的符号引用`Lorg/moonto/java/Animal`，执行invokevirtual指令后，JVM从操作栈中读取这个符号引用，并解析为内存中的直接引用。因为对象的Header中有一个指向class的指针，因此就可以按照`invokevirtual`的分派逻辑进行方法调用。

```
//duplicate code 2-4
public void doDispatch(org.moonto.java.Animal);
  descriptor: (Lorg/moonto/java/Animal;)V
  flags: ACC_PUBLIC
  Code:
    stack=1, locals=2, args_size=2
       0: aload_1
       1: invokevirtual #9                  // Method org/moonto/java/Animal.speak:()V
       4: return
    LocalVariableTable:
      Start  Length  Slot  Name   Signature
        0       5     0  this   Lorg/moonto/java/DynamicDispatch;
        0       5     1 animal   Lorg/moonto/java/Animal;
```

至此，方法调用的完成流程已经全部介绍完毕。

## 6. 小结

本文详细介绍了Java的方法分派过程，其中包括静态分派和动态分派以及方法在内存中的布局。静态分派发生在编译期，由编译器进行方法分派。动态分派发生在运行时，由虚拟机中的`invokevitrual`指令决定分派逻辑。

一般较为常见的支持动态分派的机制是Virtual method table，Java的Hotspot虚拟机，C++都在使用这种机制实现动态分派。

本文虽然介绍的是Java的方法分派过程，但所有面向对象语言都会面临这个问题，所有相关的语言都会有相关的机制实现方法分派，读者有兴趣可以参考引用的资料进一步了解相关知识，达到触类旁通的效果。

## 7. 引用

《Inside the Java Virtual Machine》  
《深入理解Java虚拟机: JVM高级特性与最佳实践》  
[https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html](https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.12.1)  
[https://en.wikipedia.org/wiki/Subtyping](https://en.wikipedia.org/wiki/Subtyping)  
[https://en.wikipedia.org/wiki/Dynamic_dispatch](https://en.wikipedia.org/wiki/Dynamic_dispatch)  
[https://en.wikipedia.org/wiki/Virtual_method_table](https://en.wikipedia.org/wiki/Virtual_method_table)  
[https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-6.html](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-6.html#jvms-6.5.invokevirtual)
