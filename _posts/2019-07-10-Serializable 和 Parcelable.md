---
layout: post
title: "Serializable 和 Parcelable"
date: 2019-06-28
description: "序列化"
tag: 进程间通讯
---
### 1.序列化和反序列化

序列化（Serialization）是将对象的状态信息转化为可以存储或者传输形式的过程，一般将一个对象存储到一个储存媒介，例如档案或记忆体缓冲等，在网络传输过程中，可以是字节或者 XML 等格式；而字节或者 XML 格式,可以还原成完全相等的对象，这个相反的过程又称为反序列化。

### 2.序列化的作用

通常完成序列化是由于以下三方面的需要：

> - 永久性保存对象，保存对象的字节序列到本地文件中
>
> - 通过序列化对象在网络中传递对象
>
> - 通过序列化在进程间传递对象

在 Java 中，对象的序列化和反序列化被广泛的应用到 RMI（远程方法调用）及网络传输中。Java 中创建的对象，并且只要对象没有被回收我们都可以复用此对象。但是，我们创建出来的这些对象都存在于 JVM 中的堆中，只有 JVM 处于运行状态的时候，这些对象才可能存在。一旦 JVM 停止，这些对象也就随之消失，但是在实际过程中，可能需要将这些对象持久化下来，并且在需要的时候将对象重新读取出来，Java 序列化可以帮助我们实现该功能。

对象序列化机制（object serialization）是 java 语言内建的一种对象持久化方式，通过对象序列化，可以将对象的状态信息保存为字节数组，并且可以在有需要的时候将这个字节数组通过反序列化的方式转换成对象，对象的序列化可以很容易的在 JVM 中的活动对象和字节数组（流）之间进行转换。

Java 类通过实现 java.io.Serialization 接口来启用序列化功能，未实现此接口的类将无法将其任何状态或者信息进行序列化或者反序列化。可序列化类的所有子类型都是可以序列化的。序列化接口没有方法或者字段，仅用于标识可序列化的语义。

在 Android 中，除了通过实现 java.io.Serialization 接口来实现序列化，还可以通过实现 Parcelable 接口，来进行序列化和反序列化，Parcelable 方式不同于将对象进行序列化，它的实现原理是将一个完整的对象进行分解，而分解后的每一部分都是Intent所支持的数据类型，这样也就实现传递对象的功能了。通常 Parcelable 序列化作为 Intent 和 Binder 的数据传递。

另外上面提到序列会为了完成上面 3 个需要，在 Android 中无法将对象引用传给 Activity 或者 Fragment，所以需要将这些对象放到一个 Intent 或者 Bundle 里面，然后再传递。

### 3.Serialization 序列化

Serialization 序列化方式比较简单，只需要将实体类实现 Serialization 接口即可，但是有一点需要注意，一般情况下最好设置一下 serialVersionUID，serialVersionUID 不是必须的，它会影响到反序列化的过程。序列化过程中，会把当前类的 serialVersionUID 写入文件当中，当进行反序列化时，会去检测当前类的 serialVersionUID 和写入文件的 serialVersionUID 进行比较，如果相同，表明当前类没有发生改变，可以反序列化成功；如果不相同，说明当前类发生过改变，例如成员变量改变，最终导致序列化失败。

(1)当我们手动指定 serialVersionUID 时，反序列化前后这个 serialVersionUID 一般情况下都是一致的，为什么说是一般情况下？因为某些对类的【破坏性】改变，如修改了类名，这时候尽管 serialVersionUID 验证通过，但是最终还是反序列化失败的

(2)如果不指定 serialVersionUID，系统会默认生成一个 serialVersionUID，这个 serialVersionUID 是根据当前类计算的 hash 值，如果类没有改变，这个 serialVersionUID 反序列化前后是一致；如果当前类发生过改变，例如成员变量改变，在检验 serialVersionUID 时，会重新计算当前类的 hash，这时候的 serialVersionUID 也跟着发生改变，和从文件中读取的 serialVersionUID 不再一致，导致反序列化失败。

```java
public class Student implements Serializable {

private static final long serialVersionUID = -8297638771764014163L;
private String name;
private int age;
private transient String nickName;

public Student(String name, int age, String nickName) {
this.name = name;
this.age = age;
this.nickName = nickName;
}

public String getName() {
return name;
}

public void setName(String name) {
this.name = name;
}

public int getAge() {
return age;
}

public void setAge(int age) {
this.age = age;
}

public String getNickName() {
return nickName;
}

public void setNickName(String nickName) {
this.nickName = nickName;
}

@Override
public String toString() {
return "Student{" +
"name='" + name + '\'' +
", age=" + age +
", nickName='" + nickName + '\'' +
'}';
}
}
```

Student 是一个实现 Serializable 接口的类，可以进行序列化，我们先来看 java 中的序列化操作。

```java
public static void main(String[] args) throws IOException, ClassNotFoundException {
Student student = new Student("Ralf", 26, "Nick");
System.out.println(student);
ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("temp"));
oos.writeObject(student);

ObjectInputStream ois = new ObjectInputStream(new FileInputStream(new File("temp")));
Student studentRead = (Student) ois.readObject();
System.out.println(studentRead);
}
```

实际上就是输入输出流的操作，输出结果：

```java
Student{name='Ralf', age=26, nickName='Nick'}
Student{name='Ralf', age=26, nickName='null'}
```
可以看到加了 transient 关键字的字段不会进行序列化，另外还有一点，static 变量也不会进行序列化，因为 static 变量属于类文件的，不属于对象。

Serialization 序列化在 Android 中使用示例：

将 student 对象从 MainActivity 传递到 SecondActivity

```java
Student student = new Student("Ralf", 26, "Nick");
Intent intent = new Intent(this, SecondActivity.class);
Bundle bundle = new Bundle();
bundle.putSerializable("student", student);
intent.putExtra("bundle", bundle);
startActivity(intent);
```
取数据

```java
Intent intent = getIntent();
Bundle bundle = intent.getBundleExtra("bundle");
if (bundle != null) {
Student student = (Student) bundle.getSerializable("student");
if (student != null) {
textView.setText(student.toString());
}
}
```

取出来的数据中也没有 transient String nickName 这个字段。

### 4.Parcelable 序列化

使用 Parcelable 序列化，代码写起来会有点麻烦，不过好在 AS 中只要设置好字段，自动实现接口时，会帮我们自动填充需要实现的代码，方便很多。

**实现Parcelable步骤**

> - 重写 writeToParcel() 方法，对象序列化为一个 Parcel 对象，即：将类的数据写入外部提供的 Parcel 中，打包需要传递的数据到 Parcel 容器保存，以便从 Parcel 容器获取数据
>
> - 重写 describeContents() 方法，内容接口描述，默认返回 0 就可以
>
> - 实例化静态内部对象 CREATOR 实现接口 Parcelable.Creator

```java
public class Teacher implements Parcelable {

private String name;
private int age;

public Teacher(String name, int age) {
this.name = name;
this.age = age;
}

protected Teacher(Parcel in) {
name = in.readString();
age = in.readInt();
}

public static final Creator<Teacher> CREATOR = new Creator<Teacher>() {
@Override
public Teacher createFromParcel(Parcel in) {
return new Teacher(in);
}

@Override
public Teacher[] newArray(int size) {
return new Teacher[size];
}
};

@Override
public int describeContents() {
return 0;
}

public int getAge() {
return age;
}

public void setAge(int age) {
this.age = age;
}

@Override
public void writeToParcel(Parcel dest, int flags) {
dest.writeString(name);
dest.writeInt(age);
}

public String getName() {
return name;
}

public void setName(String name) {
this.name = name;
}

@Override
public String toString() {
return "Teacher{" +
"name='" + name + '\'' +
", age=" + age +
'}';
}
}
```
其中 public static final 一个都不能少，内部对象 CREATOR 名称也不能改变，必须全部大写。需重写本接口中的两个方法：createFromParcel(Parcel in) 实现从 Parcel 容器中读取传递数据值，封装成 Parcelable 对象返回逻辑层，newArray(int size) 创建一个类型为T，长度为size的数组，仅一句话即可（return new T[size]），供外部类反序列化本类数组使用。
通过 writeToParcel 将对象映射成 Parcel 对象，再通过 createFromParcel 将 Parcel 对象映射成对象。也可以将 Parcel 看成是一个流，通过 writeToParcel 把对象写到流里面，在通过 createFromParcel 从流里读取对象。

Android 中使用示例：

```java
Teacher teacher = new Teacher("Wang", 28);
Intent intent = new Intent(this, ThirdActivity.class);
Bundle bundle = new Bundle();
bundle.putParcelable("teacher", teacher);
intent.putExtra("bundle", bundle);
startActivity(intent);
```
将 teacher 实例从 MainActivity 传递到 ThirdActivity 中。

```java
Intent intent = getIntent();
Bundle bundle = intent.getBundleExtra("bundle");
if (bundle != null) {
Teacher teacher = bundle.getParcelable("teacher");
if (teacher != null) {
textView.setText(teacher.toString());
}
}
```

### 5.序列化选择原则

> - 使用内存序列化时，Parcelable 比 Serializable 性能高，有测试据说能够达到 10 倍以上，所以推荐使用 Parcelable。
>
> - Parcelable 将对象序列化到存储设备或者将对象通过网络进行传输也可以完成，但是相对比较麻烦。尽管Serializable 在序列化的时候会产生大量的临时变量，容易引起频繁的 GC，同时需要大量的 IO 操作，效率没有 Parcelable，但是使用较为简单，仅仅实现 Serializable 接口即可。如果将对象序列化到存储设备或者将对象通过网络时，还是建议使用 Serializable 。
>
> - 需要在多个部件( Activity 或 Service)之间通过 Intent 传递一些数据，简单类型（如：数字、字符串）的可以直接放入 Intent。复杂类型必须实现 Parcelable 接口。


### 参考

[java对象的序列化（Serialization）和反序列化详解](https://blog.csdn.net/yaomingyang/article/details/79321939)

[序列化Serializable和Parcelable的理解和区别](https://www.jianshu.com/p/a60b609ec7e7)

**[代码地址](https://github.com/RalfNick/AndroidPractice/tree/master/SerializationTest)**
