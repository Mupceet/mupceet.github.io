---
title: 自定义 ButterKnife 框架
date: 2018-03-22 22:19:33
categories:
- 技术向
tags:
- Android
- Java
---

标题党一回，实际上这是一个关于注解、反射及简单使用的学习整理。当然，在常用框架：Dagger2、ButterKnife、Retrofit中也确实可以看到注解的身影，注解想必大家是很熟悉的，例如 Java 自带的注解 `@Override` 、 `@Depressed` 、`@SuppressWarnings`，在日常开发中，使用框架时经常使用到其自定义的注解，注解让框架的使用看起来更简单明了。

以下是一个简单的例子，天气应用中定位当前城市时使用 EasyPermissions 获取定位权限后，执行该方法：

```java
@AfterPermissionGranted(10010)
private void getLocationWithCheckPermission() {
  String[] perms = {Manifest.permission.ACCESS_FINE_LOCATION};
  if (EasyPermissions.hasPermissions(this, perms)) {
    getCurrentPosition();
  } else {
    EasyPermissions.requestPermissions(this,
        null, 10010, perms);
  }
}
```

EasyPermissions 中有着唯一的一个注解 `@AfterPermissionGranted`，它的使用场景也很简单，用它来修饰方法，该方法在声明的权限被用户所授予之后执行。就这样简单的声明之后，就可以达到一种特定的效果。

<!--more-->

可以看到注解是用字符 @ 开头，我们使用注解来修饰其后的代码元素，例如字段、方法、方法参数等，通过这种方式，编译器、其他工具可以增强或修改代码的行为。

这就是所谓声明式编程风格。在这种风格中，程序由三个组件组成：
1. 声明的关键字和语法本身
2. 系统/框架/库，它们负责解释、执行声明式的语句
3. 应用程序，使用声明式风格写程序

对应于 Java，这就是注解、反射及其在程序中的使用。其中第三部分在代码中的使用不用多说，只需要根据框架的 API 说明将注解使用在正确的可修饰的内容上即可。下面我们分别看看注解与反射。

## 注解

### 注解的创建
我们通过 EasyPermissions 的例子来说明、学习。

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface AfterPermissionGranted {

  int value();

}
```

这看起来有点像接口，只是注解在 `interface` 前加上了 @，另外，可以看到定义注解时使用了两个注解：`@Retention` 和 `@Target`，这两个注解叫做元注解，专门用于定义注解本身。

### `@Retention`
`@Retention` 表示注解信息保留到什么时候，取值只能有一个，类型为 RetentionPolicy，它是一个枚举，有三个取值：
* `SOURCE`：只在源代码中保留，编译器将代码编译为字节码文件后就会丢掉
* `CLASS`：保留到字节码文件中，但 Java 虚拟机将 class 文件加载到内存时不一定会在内存中保留
* `RUNTIME`：一直保留到运行时

如果没有声明 `@Retention`，默认为 `RetentionPolicy.CLASS`。
`@AfterPermissionGranted` 是在运行时进行使用的，所以 `@Retention` 是 `RetentionPolicy.RUNTIME`。而 `@Override` 和 `@SuppressWarnings` 都是给编译器用的，所以 `@Retention` 都是 `RetentionPolicy.SOURCE`。事实上，要想用反射去读取注解，这个值一定是要 RUNTIME 的。

### `@Target`
`@Target` 表示注解的目标，类型 ElementType 是一个枚举，可选值有：
* `TYPE`：表示类、接口（包括注解），或者枚举声明
* `FIELD`：字段，包括枚举常量
* `METHOD`：方法
* `PARAMETER`：方法中的参数
* `CONSTRUCTOR`：构造方法
* `LOCAL_VARIABLE`：本地变量
* `ANNOTATION_TYPE`：注解类型
* `PACKAGE`：包

如果没有声明@Target，默认为适用于所有类型。

`@AfterPermissionGranted` 的目标是方法，所以取值 `ElementType.METHOD`。

### 定义元素
`@AfterPermissionGranted` 的使用中可以看到，其后跟着一个 int 值，这个值是就是注解的元素。如果一个注解没有元素，那这个注解可称为 **标记注解**，起标记的作用。

注解的元素的定义实际上是通过在定义注解的时候添加一些方法：
* 方法名就是元素的名称
* 方法的返回值就是元素的类型。

其实在使用时，`@AfterPermissionGranted(10010)` 的完整形式为： `@AfterPermissionGranted(value = 10010)`，在元素仅有一个且为 value 时，可以省略“value =”。

注解的元素仅支持有限的类型：**基本类型、String、Class、枚举、注解、以及这些类型的数组**

注解看起来像接口，而它的元素定义看起来也像接口中的方法，不同的是定义元素时可以指定一个默认值，例如：`int value() default 0`，实际上，**元素一定要有值**，如果定义时没有默认值，那么在使用注解的时候一定要提供具体的值，同时，默认值与提供的值都 **不能为 null**。由于有此约束，为了表示元素值无效，需要定义默认值为空字符串或者一个负数，这已经是一种惯例。

注解能不能被继承呢？答案是不可以。大家可以自行查阅学习一个与类继承有关的元注解 `@Inherited`，这里不展开说明。

通过以上的学习，我们知道如何创建自定义注解，创建之后就可以在程序中使用：修饰指定的目标，提供需要的元素。要注意的是，同一个注解在同一对象上不可以多次使用，当然这种错误 IDE 会提醒你的。

到此，我们完成了 **第一步：注解的声明与使用**。

但注解的单独使用并不会影响到程序的运行，只是定义和使用注解的话，它和写在代码中的注释一样没太大的作用。要完成完整的功能，就需要对注解进行处理，即需要另一模块来查看代码字段、方法等是否有特殊的注解，找到想要找的注解之后再对相关代码进行解释、执行以达到增加、修改代码的行为的目的。

这就需要学习反射机制。

## 反射
对于反射机制，大家应该有一个概念，它是一种运行时的机制，在运行时去获取类的相关信息，对其进行一系列的操作：创建、修改、访问、调用等。

由于其生效于运行时，编译器无法在前期检查错误，所以使用反射时要十分注意。另外，反射的性能因为其在访问字段、调用方法前要先查找所以性能有所降低。

如果直接看 EasyPermissons 的反射的例子，会比较简单不系统，所以先了解一下必要的知识后再看例子能更好地理解。

首先，反射的入口为 Class 对象，因此需要先了解 Class 对象。

### Class 对象
每一个类都有一个 Class 对象，要在运行时利用反射来操作类的信息，首先要先获取这个 Class 对象。有以下方法可以获取对应的 Class 对象：

1. obj.getClass()
2. <类名>.class
3. Class.forName() 该方法可以获取 Class 对象，同时如果该类还没有加载，就加载它。

补充说明：

* 接口、枚举同样可以使用 2 方式获取对应 Class 对象；
* 基本类型没有 getClass() 方法，也不支持 Class.forName() 的方法进行加载，但其 Class 对象存在，为其包装类型： int.class 为 `Class <Integer>`
* 数组类型的 Class 对象每一维度都对应一个不同的
* Class.forName() 可能抛出异常 `ClassNotFoundException`

有了 Class 对象之后，通过一系列方法就可以获取相关的信息，并且可以进行操作字段、调用方法。

>由于简单入门学习，这里不会涉及全部相关内容。

想一想一个类的结构，例如：

```java
package com.mupceet.test;

public class Student extends CommonStudent {

  public int mAge;
  private String mName;

  public Student(int age, String name) {
    mAge = age;
    mName = name;
  }

  public int getAge() {
    return mAge;
  }

  public void setAge(int age) {
    mAge = age;
  }

  public String getName() {
    return mName;
  }

  public void setName(String name) {
    mName = name;
  }
}

```

从上到下我们逐一查看我们关心的一个类包含的信息：
* 类的信息：名称、类的类型（接口、注解、枚举等）、声明信息（修饰符、继承、实现等）
* 构造方法：Contractor
* 字段信息：Field
* 方法信息：Method

### 类的信息

#### 名称信息、类型信息、声明信息

```java
// 返回 Java 内部使用的真正的名字
public String getName()

// 不带包信息
public String getSimpleName()

// 返回更为友好的名字
public String getCanonicalName()

// 返回包信息
public Package getPackage()
```

它们之间的区别可以看以下表格。

| | getName | getSimpleName | getCanonicalName | getPackage |
|--|--|--|--|--|
| int.class | int | int | int | null |
| int[].class | [l | int[] | int[] | null |
| int[][].class | [[l | int[][] | int[][] | null |
| String.class | java.lang.String | String | java.lang.String | java.lang |
| String[].class | [Ljava.lang.String; | String[] | java.lang.String[] | null |
| java.util.HashMap.class | java.util.HashMap | HashMap | java.util.HashMap | java.util |
| java.util.Map.Entry.class | java.util.Map$Entry | Entry | java.util.Map.Entry | java.util | 

对于一个给定的 Class 对象，它到底是什么类型呢？可以通过以下方法进行检查：

```java
//是否是数组
public native boolean isArray();  
//是否是基本类型
public native boolean isPrimitive();
//是否是接口
public native boolean isInterface();
//是否是枚举
public boolean isEnum() 
//是否是注解
public boolean isAnnotation()
//是否是匿名内部类
public boolean isAnonymousClass()
//是否是成员类
public boolean isMemberClass() 
//是否是本地类
public boolean isLocalClass() 
```

Class 还可以获取类的声明信息，如修饰符、父类、实现的接口、注解等，如下所示：

```java
//获取修饰符，返回值可通过 Modifier 类进行解读
public native int getModifiers()
//获取父类，如果为 Object，父类为 null
public native Class<? super T> getSuperclass()
//对于类，为自己声明实现的所有接口，对于接口，为直接扩展的接口，不包括通过父类间接继承来的
public native Class<?>[] getInterfaces(); 
//自己声明的注解
public Annotation[] getDeclaredAnnotations()
//所有的注解，包括继承得到的
public Annotation[] getAnnotations()
//获取或检查指定类型的注解，包括继承得到的
public <A extends Annotation> A getAnnotation(Class<A> annotationClass)
public boolean isAnnotationPresent(Class<? extends Annotation> annotationClass)
```

#### 构造方法与创建对象

Class 有一个方法，可以用它来创建对象：

```java
public T newInstance() throws InstantiationException, IllegalAccessException
```

它会调用类的默认构造方法(即无参 public 构造方法)，如果类没有该构造方法，会抛出异常 `InstantiationException`。很多利用反射的库和框架都 **默认假定类有无参 public 构造方法**，所以使用这些库和框架时要记得提供。

newInstance 只能使用默认构造方法，Class 还有一些方法，可以获取所有的构造方法：

```java
//获取所有的 public 构造方法，返回值可能为长度为 0 的空数组
public Constructor<?>[] getConstructors() 
//获取所有的构造方法，包括非 public 的
public Constructor<?>[] getDeclaredConstructors() 
//获取指定参数类型的 public 构造方法，没找到抛出异常 NoSuchMethodException
public Constructor<T> getConstructor(Class<?>... parameterTypes)
//获取指定参数类型的构造方法，包括非 public 的，没找到抛出异常 NoSuchMethodException
public Constructor<T> getDeclaredConstructor(Class<?>... parameterTypes) 
```

类 Constructor 表示构造方法，通过它可以创建对象，方法为：

```java
public T newInstance(Object ... initargs) throws InstantiationException, IllegalAccessException, IllegalArgumentException, InvocationTargetException
```

除了创建对象，Constructor 还有很多方法，可以获取关于构造方法的很多信息，比如：

```java
//获取参数的类型信息
public Class<?>[] getParameterTypes()
//构造方法的修饰符，返回值可通过 Modifier 类进行解读
public int getModifiers()
//构造方法的注解信息
public Annotation[] getDeclaredAnnotations() 
public <T extends Annotation> T getAnnotation(Class<T> annotationClass)
//构造方法中参数的注解信息
public Annotation[][] getParameterAnnotations() 
```

### Field
类中定义的静态和实例变量都被称为字段，用类 Field 表示。Class 有四个获取字段信息的方法：

```java
//返回所有的 public 字段，包括其父类的，如果没有字段，返回空数组
public Field[] getFields() 

//返回本类声明的所有字段，包括非 public 的，但不包括父类的
public Field[] getDeclaredFields() 

//返回本类或父类中指定名称的 public 字段，找不到抛出异常 NoSuchFieldException
public Field getField(String name)

//返回本类中声明的指定名称的字段，找不到抛出异常 NoSuchFieldException
public Field getDeclaredField(String name)
```

Field 也有很多方法，可以获取字段的信息，也可以通过 Field 访问和操作指定对象中该字段的值，基本方法有：

```java
//获取字段的名称
public String getName() 

//判断当前程序是否有该字段的访问权限
public boolean isAccessible()

//flag 设为 true 表示忽略 Java 的访问检查机制，以允许读写非 public 的字段
public void setAccessible(boolean flag)

//获取指定对象 obj 中该字段的值
public Object get(Object obj)

//将指定对象 obj 中该字段的值设为 value
public void set(Object obj, Object value)

//返回字段的修饰符
public int getModifiers()

//返回字段的类型
public Class<?> getType()
//以基本类型操作字段
public void setBoolean(Object obj, boolean z)
public boolean getBoolean(Object obj)
public void setDouble(Object obj, double d)
public double getDouble(Object obj)

//查询字段的注解信息
public <T extends Annotation> T getAnnotation(Class<T> annotationClass)
public Annotation[] getDeclaredAnnotations()
```

在 get/set 方法中，对于静态变量，obj 被忽略，可以为 null，如果字段值为基本类型，get/set 会自动在基本类型与对应的包装类型间进行转换，对于 private 字段，直接调用 get/set 会抛出非法访问异常 IllegalAccessException，应该先调用 setAccessible(true) 以关闭 Java 的检查机制。

### Method
类中定义的静态和实例方法都被称为方法，用类 Method 表示，Class 有四个获取方法信息的方法：

```java
//返回所有的 public 方法，包括其父类的，如果没有方法，返回空数组
public Method[] getMethods()

//返回本类声明的所有方法，包括非 public 的，但不包括父类的
public Method[] getDeclaredMethods() 

//返回本类或父类中指定名称和参数类型的 public 方法，找不到抛出异常 NoSuchMethodException
public Method getMethod(String name, Class<?>... parameterTypes)

//返回本类中声明的指定名称和参数类型的方法，找不到抛出异常 NoSuchMethodException
public Method getDeclaredMethod(String name, Class<?>... parameterTypes)
```

Method 也有很多方法，可以获取方法的信息，也可以通过 Method 调用对象的方法，基本方法有：

```java
//获取方法的名称
public String getName() 

//flag 设为 true 表示忽略 Java 的访问检查机制，以允许调用非 public 的方法
public void setAccessible(boolean flag)

//在指定对象 obj 上调用 Method 代表的方法，传递的参数列表为 args
public Object invoke(Object obj, Object... args) throws IllegalAccessException, IllegalArgumentException, InvocationTargetException

//获取方法的修饰符，返回值可通过 Modifier 类进行解读
public int getModifiers()

//获取方法的参数类型
public Class<?>[] getParameterTypes()

//获取方法的返回值类型
public Class<?> getReturnType()

//获取方法声明抛出的异常类型
public Class<?>[] getExceptionTypes()

//获取注解信息
public Annotation[] getDeclaredAnnotations()
public <T extends Annotation> T getAnnotation(Class<T> annotationClass)

//获取方法参数的注解信息
public Annotation[][] getParameterAnnotations() 
```

对 invoke 方法，如果 Method 为静态方法，obj 被忽略，可以为 null，args 可以为 null，也可以为一个空的数组，方法调用的返回值被包装为 Object 返回，如果实际方法调用抛出异常，异常被包装为 InvocationTargetException 重新抛出，可以通过 getCause 方法得到原异常。

> Note：getDeclared*** 获取的是仅限于本类的所有的不受访问限制的，而 get*** 获取的是包括父类的但仅限于 public 修饰符的

### 使用反射需要注意的地方

> 来自：[Java反射以及在Android中的特殊应用](https://juejin.im/post/5a2c1c5bf265da431956334c)

使用反射非常方便，而且在一些特定的场合下可以实现特别的需求，但是使用反射也是需要注意一下几点的:

* 反射最好是使用 public 修饰符的，其他修饰符有一定的兼容性风险，比如这个版本有，另外的版本可能没有
* 大家都知道的 Android 开源代码引起的兼容性的问题,这是 Android 系统开源的最大的问题，特别是那些第三方的 ROM，要慎用。
* 如果大量使用反射，在代码上需要优化封装，不然不好管理，写代码不仅仅是实现功能，还有维护性和可读性方法也需要加强。


### EasyPermissons 反射处理

通过以上注解与反射的学习，我们知道注解提升了 Java 语言的表达能力，有效地实现了应用功能和底层功能的分离，框架/库的程序员可以专注于底层实现，借助反射实现通用功能，提供注解给应用程序员使用，应用程序员可以专注于应用功能，通过简单的声明式注解与框架/库进行协作。

我们再看　EasyPermissons 关于注解的处理就很简单了。

整理一下思路：

1. 查找方法，找到具有 @AfterPermissonGranted 注解的方法
2. 判断该方法的注解的元素是否与 RequestCode 相同
3. 判断该方法的参数是否符合要求
4. 暴力反射调用该方法

```java
private static void runAnnotatedMethods(@NonNull Object object, int requestCode) {
  Class clazz = object.getClass();

  while (clazz != null) {
    for (Method method : clazz.getDeclaredMethods()) {
      AfterPermissionGranted ann = method.getAnnotation(AfterPermissionGranted.class);
      if (ann != null) {
        // Check for annotated methods with matching request code.
        if (ann.value() == requestCode) {
          // Method must be void so that we can invoke it
          if (method.getParameterTypes().length > 0) {
            throw new RuntimeException(
                "Cannot execute method " + method.getName() + " because it is non-void method and/or has input parameters.");
          }

          try {
            // Make method accessible if private
            if (!method.isAccessible()) {
              method.setAccessible(true);
            }
            method.invoke(object);
          } catch (IllegalAccessException e) {
            Log.e(TAG, "runDefaultMethod:IllegalAccessException", e);
          } catch (InvocationTargetException e) {
            Log.e(TAG, "runDefaultMethod:InvocationTargetException", e);
          }
        }
      }
    }
  }
}
```


## 结合实践：自定义 ButterKnife (伪)

结合上面所说的声明式编程的三步曲：
1. 声明
  ```Java
  @Retention(RetentionPolicy.RUNTIME)
  @Target(ElementType.FIELD)
  public @interface BindView {
      @IdRes int value();
  }
  ```

2. 解释
  ```Java
  private static void bindView(Activity context) {
      Class<? extends Activity> aClass = context.getClass();

      // 遍历所有的变量字段
      Field[] declaredFields = aClass.getDeclaredFields();

      for (Field field :
              declaredFields) {
          // 如果不可访问，设置为可访问
          if (!field.isAccessible()) {
              field.setAccessible(true);
          }

          // 如果字段有 BindView 注解，则读取其元素值进行操作
          BindView bindView = field.getAnnotation(BindView.class);
          if (bindView != null) {
              int idRes = bindView.value();
              View viewById = context.findViewById(idRes);

              try {
                  field.set(context, viewById);
              } catch (IllegalAccessException e) {
                  e.printStackTrace();
              }
          }
      }
  }
  ```
3. 使用
  ```Java
  @BindView(R.id.tv_content)
  TextView mTextView;
  @BindView(R.id.btn_change_content)
  Button mButton;

  @Override
  protected void onCreate(Bundle savedInstanceState) {
      super.onCreate(savedInstanceState);
      setContentView(R.layout.activity_butter_knife);

      ButterKnife.bind(this);
  }
  ```

当然，以上只是简单的示例，真正的 [ButterKnife](https://github.com/JakeWharton/butterknife) 不是这样子实现的，它是使用了注解处理器（APT：相关类`javax.annotation.processing`），在编译时对注解进行处理生成辅助代码及其他文件，从而避免降低运行时性能。编译时生成代码，可以在项目的build/generated/source/apt/目录下找到生成的代码，也可以对这些代码进行调试。现在主流的框架很多都使用这种方式来生成代码，比如 Dagger2、Glide、GreenDao、Room等。

## 其他

Android 提供了额外的注解，合理利用这些注解可以提高代码可读性和可维护性，Coding 时就可以避免错误，比如常用的 @Nullable、@Nonnull等，要查看更多相关注解及其使用请点击以下链接：

* [探究 Android 中的注解](https://droidyue.com/blog/2016/08/14/android-annnotation/) : 介绍 Android 中的注解,以及 ButterKnife 和 Otto 这些基于注解的库的一些工作原理

* [Android 中注解的使用](https://juejin.im/post/59bf5e1c518825397176d126) : Android Support Library 提供的元注解介绍及使用，参考了上一文章。

* [Java 反射以及在 Android 中的特殊应用](https://juejin.im/post/5a2c1c5bf265da431956334c) : 文章介绍了反射在 Android 框架层的应用：作为四大组件的 Activity 其实也是一个普通对象，也是由反射创建。

* 本文的大部分内容摘自老马的《[Java 编程的逻辑](https://item.jd.com/12299018.html)》进行整理，这本书对 Java 知识点的讲解清晰透彻，对入门的人深入理解 Java 很有帮助，推荐购买阅读。
