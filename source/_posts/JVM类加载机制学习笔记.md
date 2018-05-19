---
title: JVM类加载机制学习笔记
date: 2018-05-15 07:43:47
tags:
- java
- jvm
- classload
categories:
- java
---

**JVM类加载机制**

- 源代码被编译成class文件
- class文件通过jvm的类加载器进行加载
- class的执行包括解释执行和编译执行，两种执行方式都有很多优化
- JVM中线程资源同步的机制和线程之间交互的机制

与那些在编译时需要进行连接工作的语言不同，在Java语言里面，类型的加载、连接和初始化过程都是在程序运行期间完成的，这种策略虽然会令类加载时稍微增加一些性能开销，但是会为Java应用程序提供高度的灵活性，Java里天生可以动态扩展的语言特性就是依赖运行期动态加载和动态连接这个特点实现的。例如，如果编写一个面向接口的应用程序，可以等到运行时再指定其实际的实现类；用户可以通过Java预定义的和自定义类加载器，让一个本地的应用程序可以在运行时从网络或其他地方加载一个二进制流作为程序代码的一部分，这种组装应用程序的方式目前已广泛应用于Java程序之中。

**类加载过程**：

- 装载
- 链接：包括验证、准备、解析
- 初始化

---
# 加载
---

将字节码加载到JVM中，根据类的全路径和类加载器唯一标识一个被加载的类，创建代表这个类的Class的对象。

加载阶段完成后，虚拟机外部的二进制字节流就按照虚拟机所需的格式存储在方法区之中，方法区中的数据存储格式由虚拟机实现自行定义，虚拟机规范未规定此区域的具体数据结构。然后在内存中实例化一个java.lang.Class类的对象（并没有明确规定是在Java堆中，对于HotSpot虚拟机而言，Class对象比较特殊，它虽然是对象，但是存放在方法区里面），这个对象将作为程序访问方法区中的这些类型数据的外部接口。

在加载阶段，虚拟机需要完成以下3件事情：

- 通过一个类的全限定名来获取定义此类的二进制字节流。
- 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。
- 在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据的访问入口。

相对于类加载过程的其他阶段，一个非数组类的加载阶段（准确地说，是加载阶段中获取类的二进制字节流的动作）是开发人员可控性最强的，因为加载阶段既可以使用系统提供的引导类加载器来完成，也可以由用户自定义的类加载器去完成，开发人员可以通过定义自己的类加载器去控制字节流的获取方式（即重写一个类加载器的loadClass（）方法）。

对于数组类而言，情况就有所不同，数组类本身不通过类加载器创建，它是由Java虚拟机直接创建的。但数组类与类加载器仍然有很密切的关系，因为数组类的元素类型（ElementType，指的是数组去掉所有维度的类型）最终是要靠类加载器去创建

---
# 连接
---

连接就是将已经读入到内存的类的二进制数据加载到虚拟机的运行时环境中去。将字节码中的内容，例如类的静态字段等，与内存进行连接，在内存中分配空间。

连接阶段首先会对二进制字节码的格式进行校验，校验过程中如果碰到了要引用的其它接口和实现，也会进行加载。

对于类中的非final类型的静态变量，进行连接的时候，只会在方法区中分配变量所使用的空间，不会真正赋值，在初始化的时候才会真正赋值。

## 连接中的验证

验证是连接阶段的第一步，这一阶段的目的是为了确保Class文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机自身的安全。Class文件并不一定要求用Java源码编译而来，可以使用任何途径产生，甚至包括用十六进制编辑器直接编写来产生Class文件。虚拟机如果不检查输入的字节流，对其完全信任的话，很可能会因为载入了有害的字节流而导致系统崩溃，所以验证是虚拟机对自身保护的一项重要工作.

## 连接中的准备

准备阶段是正式为类变量分配内存并设置类变量初始值的阶段，这些变量所使用的内存都将在方法区中进行分配。这个阶段中有两个容易产生混淆的概念需要强调一下，首先，这时候进行内存分配的仅包括类变量（被static修饰的变量），而不包括实例变量，实例变量将会在对象实例化时随着对象一起分配在Java堆中。其次，这里所说的初始值“通常情况”下是数据类型的零值。

## 连接中的解析

字节码中的符号变量替换成为直接引用。

- 符号引用与虚拟机实现的布局无关，引用的目标并不一定要已经加载到内存中。各种虚拟机实现的内存布局可以各不相同，但是它们能接受的符号引用必须是一致的，因为符号引用的字面量形式明确定义在Java虚拟机规范的Class文件格式中。
- 直接引用可以是指向目标的指针、相对偏移量或是一个能间接定位到目标的句柄。如果有了直接引用，那引用的目标必定已经在内存中存在。

编译阶段生成的字节码中的包含的引用并不是真实的地址，所以编译生成的字节码文件中的变量还都是符号变量，没有对应真实的内存中的地址。但是经过加载、验证、准备阶段之后，变量会在内存中分配空间，也就是说有了真实的内存中的地址，所以在解析阶段可以对符号变量进行替换。

**注意**：

加载阶段与连接阶段的部分内容（如一部分字节码文件格式验证动作）是交叉进行的，加载阶段尚未完成，连接阶段可能已经开始，但这些夹在加载阶段之中进行的动作，仍然属于连接阶段的内容，这两个阶段的开始时间仍然保持着固定的先后顺序。

---
# 初始化
---

类初始化阶段是类加载过程的最后一步，前面的类加载过程中，除了在加载阶段用户应用程序可以通过自定义类加载器参与之外，其余动作完全由虚拟机主导和控制。到了初始化阶段，才真正开始执行类中定义的Java程序代码（或者说是字节码）。

在准备阶段，变量已经赋过一次系统要求的初始值，而在初始化阶段，则根据程序员通过程序制定的主观计划去初始化类变量和其他资源，或者可以从另外一个角度来表达：初始化阶段是执行类构造器＜clinit＞（）方法的过程。

＜clinit＞（）方法是由编译器自动收集类中的所有类变量的赋值动作和静态语句块（static{}块）中的语句合并产生的，编译器收集的顺序是由语句在源文件中出现的顺序所决定的，静态语句块中只能访问到定义在静态语句块之前的变量，定义在它之后的变量，在前面的静态语句块可以赋值，但是不能访问。

＜clinit＞（）方法与类的构造函数（或者说实例构造器＜init＞（）方法）不同，它不需要显式地调用父类构造器，虚拟机会保证在子类的＜clinit＞（）方法执行之前，父类的＜clinit＞（）方法已经执行完毕。因此在虚拟机中第一个被执行的＜clinit＞（）方法的类肯定是java.lang.Object。

整个类的加载过程包括：加载、连接（验证、准备、解析）、初始化，这几个阶段会按照这个顺序开始（但是不一定按照这个顺序完成），虽然加载是类加载过程的第一步，但是JVM没有明确规定什么时候执行加载，JVM给出的规范是规定什么时候必须要进行初始化，当然初始化之前必须先要执行加载、验证、准备阶段。JVM规定在以下五种情况下必须对类进行初始化，而且这五种情况是有且只有：

- 遇到new、 getstatic、 putstatic或invokestatic这4条字节码指令时，如果类没有进行过初始化，则需要先触发其初始化。生成这4条指令的最常见的Java代码场景是：使用new关键字实例化对象的时候、 读取或设置一个类的静态字段（被final修饰、 已在编译期把结果放入常量池的静态字段除外）的时候，以及调用一个类的静态方法的时候。
- 使用java.lang.reflect包的方法对类进行反射调用的时候，如果类没有进行过初始化，则需要先触发其初始化。
- 当初始化一个类的时候，如果发现其父类还没有进行过初始化，则需要先触发其父类的初始化。
- 当虚拟机启动时，用户需要指定一个要执行的主类（包含main（）方法的那个类），虚拟机会先初始化这个主类。
- 当使用JDK 1.7的动态语言支持时，如果一个java.lang.invoke.MethodHandle实例最后的解析结果为REF_getStatic、REF_putStatic、REF_invokeStatic的方法句柄，并且这个方法句柄所对应的类没有进行过初始化，则需要先触发其初始化。

虚拟机会保证一个类的＜clinit＞（）方法在多线程环境中被正确地加锁、同步，如果多个线程同时去初始化一个类，那么只会有一个线程去执行这个类的＜clinit＞（）方法，其他线程都需要阻塞等待，直到活动线程执行＜clinit＞（）方法完毕。 如果在一个类的＜clinit＞（）方法中有耗时很长的操作，就可能造成多个进程阻塞，在实际应用中这种阻塞往往是很隐蔽的。

**知识点**：

- 对于静态字段，只有直接定义这个字段的类才会被初始化，因此通过其子类来引用父类中定义的静态字段，只会触发父类的初始化而不会触发子类的初始化。
- `SuperClass[]sca=new SuperClass[10]`不会触发SuperClass的初始化，只会初始化一个名为`[Lorg.fenixsoft.classloading.SuperClass`的类，它是一个由虚拟机自动生成的、直接继承于java.lang.Object的子类，创建动作由字节码指令newarray触发。
- `public static final String HELLOWORLD="hello world"`调用类中的`static final`变量不会触发类的初始化，在编译阶段通过常量传播优化，已经将此常量的值“hello world”存储到了NotInitialization类的常量池中，以后NotInitialization对常量ConstClass.HELLOWORLD的引用实际都被转化为NotInitialization类对自身常量池的引用了。也就是说，实际上NotInitialization的Class文件之中并没有ConstClass类的符号引用入口，这两个类在编译成Class之后就不存在任何联系了。
- 接口的初始化：当一个类在初始化时，要求其父类全部都已经初始化过了，但是一个接口在初始化时，并不要求其父接口全部都完成了初始化，只有在真正使用到父接口的时候（如引用接口中定义的常量）才会初始化。

类的加载过程将类的所有信息都加载到了JVM的内存中，并且对其中类变量进行了初始化，所以经过加载、连接、初始化之后，一个类便可以正常的使用：

- 调用类中的静态变量和静态方法
- 创建类的对象实例

我们在代码中进行new操作创建对象的时候，首先进行的是类的加载，加载之后使用该类创建一个实例对象存放在JVM的堆内存中。当然不仅仅是创建对象的时候会进行类的加载，上面已经提到了5中JVM规定必须要进行类的初始化的场景，如果仅仅是使用类的静态属性或者静态方法，那么只会在JVM的方法区中创建类对应的Class对象，并不会在堆内存中创建类的对象实例，所以说类加载与对象创建不能混为一谈，类加载是对象创建的前提。

---
# 类加载器
---

类的加载阶段中的“通过一个类的全限定名来获取描述此类的二进制字节流”这个动作放在JVM虚拟机外部去实现，以便让应用程序自己去决定如何获取所需要的类，实现这个动作的代码模块叫“类加载器”。也正是因为这个类加载器，应用程序可以通过反射机制，实现运行时加载类，创建对象。

相同的类加载器加载相同的二进制字节流会得到相同的Class对象，但是不同的类加载器加载相同的二进制字节流，却得到不同的Class对象。

JVM的类加载采用的双亲委托机制，从Java虚拟机的角度来讲，只存在两种不同的类加载器：一种是启动类加载器（Bootstrap ClassLoader），这个类加载器使用C++语言实现，是虚拟机自身的一部分；另一种就是所有其他的类加载器，这些类加载器都由Java语言实现，独立于虚拟机外部，并且全都继承自抽象类java.lang.ClassLoader。

绝大多数java程序都会使用3种系统自带的类加载器：

- 启动类加载器（Bootstrap ClassLoader）
- 扩展类加载器（Extension ClassLoader）
- 应用程序类加载器（Application ClassLoader）

这三种类加载器，以及用户自定义的类加载器之间存在着层级关系，称为类加载器的双亲委托模型。双亲委派模型要求除了顶层的启动类加载器外，其余的类加载器都应当有自己的父类加载器。这里类加载器之间的父子关系一般不会以继承（Inheritance）的关系来实现，而是都使用组合（Composition）关系来复用父加载器的代码。

双亲委派模型的工作过程是：如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的加载请求最终都应该传送到顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个加载请求（它的搜索范围中没有找到所需的类）时，子加载器才会尝试自己去加载。

使用双亲委派模型来组织类加载器之间的关系，有一个显而易见的好处就是Java类随着它的类加载器一起具备了一种带有优先级的层次关系。例如类java.lang.Object，它存放在rt.jar之中，无论哪一个类加载器要加载这个类，最终都是委派给处于模型最顶端的启动类加载器进行加载，因此Object类在程序的各种类加载器环境中都是同一个类。相反，如果没有使用双亲委派模型，由各个类加载器自行去加载的话，如果用户自己编写了一个称为java.lang.Object的类，并放在程序的ClassPath中，那系统中将会出现多个不同的Object类，Java类型体系中最基础的行为也就无法保证，应用程序也将会变得一片混乱。

## 双亲委派模型的源代码实现

```
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // First, check if the class has already been loaded
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            //如果父加载器存在，通过父加载器进行递归加载
            try {
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                c = findClass(name);

                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```

`AppClassLoader`中有属性`parent`，对应的父加载器为`ExtClassLoader`，父加载器通过`parent`属性进行组合，不是通过继承的方式。

`ExtClassLoader`的父加载器为`Bootstrap ClassLoader`，但是并不是通过java代码来实现的，也就是说`ExtClassLoader`虽然有父加载器，但是代码中的`parent`为空。因为`Bootstrap ClassLoader`不是通过java实现的，是通过C++实现的。

所以当递归到`ExtClassLoader`的时候，如果没有成功加载类，递归停止，调用`findBootstrapClassOrNull(name);`方法进行`Bootstrap`类加载器进行加载。

如果`Bootstrap`没有成功加载类，那么通过最开始进行加载的类加载器进行加载：

```
if (c == null) {
    // If still not found, then invoke findClass in order
    // to find the class.
    long t1 = System.nanoTime();
    c = findClass(name);
    //....
}
```

---
# MethodHandle
---

MethodHandle的使用方法和效果上与Reflection都有众多相似之处。不过，它们也有以下这些区别：

- Reflection和MethodHandle机制本质上都是在模拟方法调用，但是Reflection是在模拟Java代码层次的方法调用，而MethodHandle是在模拟字节码层次的方法调用。在MethodHandles.Lookup上的三个方法findStatic()、findVirtual()、findSpecial()正是为了对应于invokestatic、invokevirtual & invokeinterface和invokespecial这几条字节码指令的执行权限校验行为，而这些底层细节在使用Reflection API时是不需要关心的。
- Reflection中的java.lang.reflect.Method对象远比MethodHandle机制中的java.lang.invoke.MethodHandle对象所包含的信息来得多。前者是方法在Java一端的全面映像，包含了方法的签名、描述符以及方法属性表中各种属性的Java端表示方式，还包含有执行权限等的运行期信息。而后者仅仅包含着与执行该方法相关的信息。用开发人员通俗的话来讲，Reflection是重量级，而MethodHandle是轻量级。
- 由于MethodHandle是对字节码的方法指令调用的模拟，那理论上虚拟机在这方面做的各种优化（如方法内联），在MethodHandle上也应当可以采用类似思路去支持（但目前实现还不完善）。而通过反射去调用方法则不行。

和反射相比好处是：

- 调用 invoke() 已经被JVM优化，类似直接调用一样。
- 性能好得多，类似标准的方法调用。
- 当我们创建MethodHandle 对象时，实现方法执行权限检测(如MethodHandles.Lookup上的三个方法findStatic()、findVirtual()、findSpecial())，而不是调用invoke() 时。

MethodHandle与Reflection除了上面列举的区别外，最关键的一点还在于去掉前面讨论施加的前提“仅站在Java语言的角度看”之后：Reflection API的设计目标是只为Java语言服务的，而MethodHandle则设计为可服务于所有Java虚拟机之上的语言，其中也包括了Java语言而已。

java7在JSR 292中增加了对动态类型语言的支持，使java也可以像C语言那样将方法作为参数传递，其实现在lava.lang.invoke包中。MethodHandle作用类似于反射中的Method类，但它比Method类要更加灵活和轻量级。通过MethodHandle进行方法调用一般需要以下几步：

- 创建MethodType对象，指定方法的签名；
- 在MethodHandles.Lookup中查找类型为MethodType的MethodHandle；
- 传入方法参数并调用MethodHandle.invoke或者MethodHandle.invokeExact方法。

```
import java.lang.invoke.MethodHandle;
import java.lang.invoke.MethodHandles;
import java.lang.invoke.MethodType;
import java.util.Arrays;
import java.util.List;
public class MethodHandleTest {
    public static void main(String[] args) throws Throwable {
        MethodHandles.Lookup lookup = MethodHandles.lookup();
        MethodHandle mh = lookup.findStatic(MethodHandleTest.class, "doubleVal", MethodType.methodType(int.class, int.class));
        List<Integer> dataList = Arrays.asList(1, 2, 3, 4, 5);
        MethodHandleTest.transform(dataList, mh);// 方法做为参数
        for (Integer data : dataList) {
            System.out.println(data);//2,4,6,8,10
        }
    }
    public static List<Integer> transform(List<Integer> dataList, MethodHandle handle) throws Throwable {
        for (int i = 0; i < dataList.size(); i++) {
            dataList.set(i, (Integer) handle.invoke(dataList.get(i)));//invoke
        }
        return dataList;
    }
    public static int doubleVal(int val) {
        return val * 2;
    }
}
```

### 参考

- 《Java虚拟机（第二版）》
- [MethodHandle与反射Method区别，invokedynamic指令](https://blog.csdn.net/yushuifirst/article/details/48028859)
- [java7 MethodHandle学习笔记](https://blog.csdn.net/aesop_wubo/article/details/48858931)

