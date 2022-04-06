# JavaSE高级特性：反射

#### 1.反射是什么

Java反射是框架的灵魂，大量框架底层都用到了反射机制，例如Spring....

Java反射是在**运行状态时**，可以构造任何一个类的对象，获取到任意一个对象所属的类信息，以及这个类的成员变量或者方法，可以调用任意一个对象的属性或者方法。可以理解为具备了**动态加载对象**以及**对对象的基本信息进行剖析和使用**的能力的一种机制。

解释型语言：不需要编译，在运行的时候逐行翻译解释；修改代码时可以直接修改，可以快速部署，不过性能上会比编译型语言稍差；比如 JavaScript、Python ；

编译型语言：需要通过编译器将源代码编译成机器码才能执行；编译之后如果需要修改代码，在执行之前就需要重新编译。比如 C 语言；

Java 严格来说也是编译型语言，但又介于编译型和解释型之间；Java 不直接生成机器码而是生成中间码：编译期间，是将源码交给编译器生成 class 文件（字节码），这个过程中只做了翻译的工作，并没有把代码放入内存运行；当进入运行期，字节码才被 Java 虚拟机加载、解释成机器语言并运行。

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/%E5%8F%8D%E5%B0%841)

根据以上流程我们一个类的创建过程如下所示：

![](/Users/shepherdmy/Desktop/反射2.png)

Java 反射主要涉及两个类(接口) **Class**， **Member**，如果把这两个类搞清楚了，反射基本就 ok 了。

**Class** 可以说是反射能够实现的基础；注意这里说的 **Class**与 class 关键字不是同一种东西。class 关键字是在声明 java 类时使用的；而 **Class** 是 java JDK 提供的一个类,完整路径为 java.lang.Class，本质上与 Math, String 或者你自己定义各种类没什么区别，其官方文档定义解释如下：

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/%E5%8F%8D%E5%B0%843)

对于每一种类，Java 虚拟机都会初始化出一个 Class 类型的实例，每当我们编写并且编译一个新创建的类就会产生一个对应 Class 对象，并且这个 Class 对象会被保存在同名 .class 文件里。当我们 new 一个新对象或者引用静态成员变量时，Java 虚拟机(JVM)中的类加载器系统会将对应 Class 对象加载到 JVM 中，然后 JVM 再根据这个类型信息相关的Class 对象创建我们需要实例对象或者提供静态变量的引用值。

比如创建编译一个 Shapes 类，那么，JVM 就会创建一个 Person 对应 Class 类的 Class实例，**该 Class 实例保存了 Person 类相关的类型信息，包括属性，方法，构造方法等等**，通过这个 Class 实例可以在运行时访问 Person 对象的属性和方法等。另外通过 Class类还可以创建出一个新的 Person 对象。这就是反射能够实现的原因，可以说 Class 是反射操作的基础。

需要特别注意的是，每个 class（注意 class 是小写，代表普通类）类，无论创建多少个实例对象，在 JVM 中都对应同一个 Class 对象。

#### 2.反射的实现

1）获取Class对象的三种方式：

- Class.forName("全类名")：将字节码文件加载进内存，返回Class对象
- 类名.class：通过类名的属性
- class获取对象.getClass()：getClass()方法在Object类中定义着。

```java
        /**
         获取Class对象的方式：
         1.Class.forName("全类名")：将字节码文件加载进内存，返回Class对象
         2.类名.class：通过类名的属性class获取
         3.对象.getClass()：getClass()方法在Object类中定义着。
         */
        //1.Class.forName("全类名")
        Class cls1 = Class.forName("com.shepherd.reflect.Person");
        System.out.println(cls1);
        //2.类名.class
        Class cls2 = Person.class;
        System.out.println(cls2);
        //3.对象.getClass()
        Person p = new Person();
        Class cls3 = p.getClass();
        System.out.println(cls3);
        //==比较三个对象
        System.out.println(cls1==cls2);//true
        System.out.println(cls1==cls3);//true

        Class  c = Student.class;
        System.out.println(c);
        System.out.println(c==cls1);//false

```

说 Class 是反射能够实现的基础的另一个原因是：Java 反射包 java.lang.reflect 中的所有类都没有 public 构造方法，要想获得这些类实例，**只能通过 Class 类获取。所以说如果想使用反射，必须得获得 Class 对象**。

2）Class类的相关方法：Class 实例可以在运行时访问 相应 对象的属性和方法，构造方法等等

```java
  			Class c = Student.class;
        //获取public类型的属性，包括父类的
        Field[] fields = c.getFields();
        System.out.println(Arrays.asList(fields));

        //只获取当前类所有属性
        Field[] declaredFields = c.getDeclaredFields();
        System.out.println(Arrays.asList(declaredFields));

        //获取当前类的方法
        Method[] methods = c.getMethods();
        System.out.println(Arrays.asList(methods));

        //获取当前类的构造对象
        Constructor constructor = c.getConstructor();
        Student stu = (Student) constructor.newInstance();
        System.out.println(stu);
```

**这里需要注意：xxx.getFields()方法默认情况下，会返回本类、父类、父接口的公有属性，而xxx.getDeclaredFields()返回本类的所有属性，包括私有的属性。同理，反射API中其他getXXX和getDeclaredXXX的用法类似。**

3）对于 Member 接口可能会有人不清楚是干什么的，但如果提到实现它的三个实现类，估计用过反射的人都能知道。我们知道类成员主要包括构造函数，变量和方法，Java 中的操作基本都和这三者相关，而 Member 的这三个实现类就分别对应他们。

```  
java.lang.reflect.Field ：对应类变量
java.lang.reflect.Method ：对应类方法
java.lang.reflect.Constructor ：对应类构造函数
```

- **Field：类属性变量**

  通过 Field 你可以访问给定对象的类变量，包括获取变量的类型、修饰符、注解、变量名、变量的值或者重新设置变量值，即使变量是 private 的。

  - **获取Field**：Class 提供了4种方法获得给定类的 Field:

  - - getDeclaredField(String name) ：获取指定的变量（只要是声明的变量都能获得，包括 private）

  - - getField(String name) ：获取指定的变量（只能获得 public 的）
    - getDeclaredFields()  ：获取所有声明的变量（包括 private）
    - getFields()：获取所有的 public 变量

  - **获取属性变量类型、修饰符、注解**

    ```java
            Class c = Student.class;
    				Field field = c.getField("sn");
            System.out.println(field);
            //获取名称
            System.out.println(field.getName());
            //获取类型
            System.out.println(field.getType());
            //获取修饰符
            System.out.println(Modifier.toString(field.getModifiers()));
            //获取注解
            System.out.println(Arrays.asList(field.getAnnotations()));
    ```

  - **获取、修改属性值**

    ```java
    				//获取public属性值
            Long sn  = (long) field.get(stu);
            System.out.println(sn);
    
            //获取private属性值，这时候需要setAccessible(true)，因为私有属性是不能被其他类访问的
            Field declaredField = c.getDeclaredField("schoolName");
            declaredField.setAccessible(true);
            String schoolName = (String)declaredField.get(stu);
            System.out.println(schoolName);
    
            //修改的对象属性值
            field.set(stu, 1l);
            declaredField.set(stu, "浙大");
            System.out.println(stu);
    ```

    这里需要注意：如果是访问private修饰的属性会报错，因为私有属性的值是不能其他类访问，没有权限。强大的反射可以帮我们打破这种约束。反射包里为我们提供了一个强大的类。

    > java.lang.reflect.AccessibleObject

    AccessibleObject 为我们提供了一个方法 setAccessible(boolean flag)，该方法的作用就是可以取消 Java 语言访问权限检查。所以任何继承 AccessibleObject 的类的对象都可以使用该方法取消 Java 语言访问权限检查。（final 类型变量也可以通过这种办法访问）。Field 正好继承 AccessibleObject ，所以只要在访问私有变量前调用 filed.setAccessible(true) 就可以了。代码看上面示例即可

    **注意 Method 和 Constructor 也都是继承 AccessibleObject，所以如果遇到私有方法和私有构造函数无法访问，记得处理方法一样**

  

- **Method**

  - **获取Method**：Class 依然提供了4种方法获取 Method

    - getDeclaredMethod(String name, Class<?>... parameterTypes)：根据方法名获得指定的方法， 参数 name 为方法名，参数 parameterTypes 为方法的参数类型，如 getDeclaredMethod(“eat”, String.class)
    - getMethod(String name, Class<?>... parameterTypes)：根据方法名获取指定的 public 方法，其它同上
    - getDeclaredMethods()：获取所有声明的方法
    - getMethods()：获取所有的 public 方法

     **注意：获取带参数方法时，如果参数类型错误会报 NoSuchMethodException，对于参数是泛型的情况，泛型须当成Object处理（Object.class）**

  - **获取方法返回类型**

    - getReturnType()  获取目标方法返回类型对应的 Class 对象
    - getGenericReturnType()  获取目标方法返回类型对应的 Type 对象

      **两者的区别：getReturnType()返回类型为 Class，getGenericReturnType() 返回类型为 Type; Class 实现 Type**

    ```tex
    1.返回值为普通简单类型如 Object, int, String 等，getGenericReturnType() 返回值和 getReturnType() 一样
    
    例如 public String function1()，那么各自返回值为：
    
    getReturnType() : class java.lang.String
    
    getGenericReturnType() : class java.lang.String
    
    2.返回值为泛型
    
    例如 public T function2()，那么各自返回值为：
    
    getReturnType() : class java.lang.Object
    
    getGenericReturnType() : T
    
    3.返回值为参数化类型
    
    例如public Class<String> function3()，那么各自返回值为：
    
    getReturnType() : class java.lang.Class
    
    getGenericReturnType() : java.lang.Class<java.lang.String>
    ```

    **其实反射中所有形如 getGenericXXX()的方法规则都与上面所述类似**

  - **获取方法参数类型**

    getParameterTypes() 获取目标方法各参数类型对应的 Class 对象
    getGenericParameterTypes() 获取目标方法各参数类型对应的 Type 对象
    返回值为数组，它俩区别同上 “方法返回类型的区别” 。

  - **获取方法声明抛出的异常的类型**

    getExceptionTypes() 获取目标方法抛出的异常类型对应的 Class 对象
    getGenericExceptionTypes()  获取目标方法抛出的异常类型对应的 Type 对象
    返回值为数组，区别同上

  - **获取方法参数名称**

    .class 文件中默认不存储方法参数名称，如果想要获取方法参数名称，需要在编译的时候加上 -parameters 参数。(构造方法的参数获取方法同样)

  - 通过反射调用方法

    反射通过Method的invoke()方法来调用目标方法。第一个参数为需要调用的目标类对象，如果方法为static的，则该参数为null。后面的参数都为目标方法的参数值，顺序与目标方法声明中的参数顺序一致。

  ```java
  
  				Class c = Student.class;
          Method method = c.getMethod("getAddress", String.class, String.class, String.class);
  
          //获取方法的修饰符
          System.out.println(Modifier.toString(method.getModifiers()));
  
          //获取方法的放回类型
          System.out.println(method.getReturnType());
  
          //获取方法的参数类型
          System.out.println(Arrays.asList(method.getParameterTypes()));
  
          //获取参数抛出的异常
          System.out.println(Arrays.asList(method.getExceptionTypes()));
  
  
          Method doHomework = c.getDeclaredMethod("doHomework", String.class, Integer.class);
          doHomework.setAccessible(true);
          System.out.println(doHomework.getReturnType());
          System.out.println(doHomework.getName());
          doHomework.invoke(stu, "数学", 2);
  
  ```

  **注意：如果方法是private的，可以使用 method.setAccessible(true) 方法绕过权限检查。被调用的方法本身所抛出的异常在反射中都会以 InvocationTargetException 抛出。换句话说，反射调用过程中如果异常 InvocationTargetException 抛出，说明反射调用本身是成功的，因为这个异常是目标方法本身所抛出的异常**

- **Constructor**

  - 获取构造方法：和 Method 一样，Class 也为 Constructor 提供了4种方法获取

    - - getDeclaredConstructor(Class<?>... parameterTypes)：获取指定构造函数，参数 parameterTypes 为构造方法的参数类型
      - getConstructor(Class<?>... parameterTypes)：获取指定 public 构造函数，参数 parameterTypes 为构造方法的参数类型
      - getDeclaredConstructors()：获取所有声明的构造方法
      - getConstructors()：获取所有的 public 构造方法

    构造方法的名称、限定符、参数、声明的异常等获取方法都与 Method 类似，请参照Method。

  - 创建对象

    通过反射有两种方法可以创建对象：

    - java.lang.reflect.Constructor.newInstance()
    - Class.newInstance()

    一般来讲，我们优先使用第一种方法；那么这两种方法有何异同呢？

    1. Class.newInstance()仅可用来调用无参的构造方法；Constructor.newInstance()可以调用任意参数的构造方法。
    2. Class.newInstance()会将构造方法中抛出的异常不作处理原样抛出; Constructor.newInstance()会将构造方法中抛出的异常都包装成 InvocationTargetException 抛出。
    3. Class.newInstance()需要拥有构造方法的访问权限; Constructor.newInstance()可以通过 setAccessible(true) 方法绕过访问权限访问 private 构造方法。

#### 3.反射的缺点

- 性能开销

  反射涉及类型动态解析，所以 JVM 无法对这些代码进行优化。因此，反射操作的效率要比那些非反射操作低得多。我们应该避免在经常被执行的代码或对性能要求很高的程序中使用反射。

- 安全限制

  使用反射技术要求程序必须在一个没有安全限制的环境中运行。如果一个程序必须在有安全限制的环境中运行，如 Applet，那么这就是个问题了。

- 内部曝光

  由于反射允许代码执行一些在正常情况下不被允许的操作（比如访问私有的属性和方法），所以使用反射可能会导致意料之外的副作用－－代码有功能上的错误，降低可移植性。反射代码破坏了抽象性，因此当平台发生改变的时候，代码的行为就有可能也随着变化。

  以上示例完整代码demo：https://github.com/ShepherdZFJ/spring_code_learn/tree/main/framework_core



