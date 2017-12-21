---
layout: post
title: "Android Studio 中通过 CMake 方式使用 JNI"
date: 2017-12-04
description: "初学NDK开发，记录知识要点和主要操作，防止重复造轮子"
tag: Android
---

在 AS 中使用 Eclipse 中的方式构建 jni 环境，这种方式配置起来稍有麻烦，在 AS 中还有另外一种方式可以使用：即 CMake 方式。

### 一、说点题外话：

2015年6月26日，Android产品经理在Android官网发表博客

1、2015年底停止对eclipse的adt更新支持，后续更新由eclipse团体提供

2、推荐大家使用Android官方集成开发环境 Android Studio

3、介绍eclipse到Android Studio 的项目转移

所以 Eclise 会逐步被取代，虽然很经典，但是既然出现能够替代他的产品，说明AS已经很优秀，足够好用！所以我们也要跟上步伐，上手 AS！之前的版本对 NDK 开发很多开发者还是喜欢使用 Eclipse，但是 Android Studio 已经更新到 3.0，对 NDK 支持又加大了力度，所以对于进行 NDK 开发的小伙伴来说是一个福音（ PS：木有受委托来推广AS。。。）

AS 中 NDK 开发的特点：
Android Studio 充分支持编辑 C/C++ 项目文件，可以在应用中快速构建 JNI 组件。IDE 为 C/C++ 提供了语法突出显示和重构功能，还提供了一个基于 LLDB 的调试程序，可以利用此调试程序同时调试 Java 和 C/C++ 代码。构建工具还可以在不进行任何修改的情况下执行CMake 和 ndk-build 脚本，然后将共享对象添加到 APK 中。

本篇文章是主要介绍一下在AS中通过 CMake 方式使用 JNI 的基本操作。

### 二、创建工程

#### 1.创建工程，选择 “ Include C++ support ”

[图片上传失败...(image-a0ba69-1512391552569)]


#### 2.选择 Toolchain Default，下面两个选项也勾选上

![图一](http://upload-images.jianshu.io/upload_images/4744186-bd4fcf621635dec1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 3.目录结构（我们需要配置的文件）：

>* （1）CmakLists.txt

>* （2）app下的build.gradle文件

[图片上传失败...(image-1f8f48-1512391552570)]

#### 4. build.gradle

在我们的项目中的app中的build.gradle的
defaultConfig｛｝后面添加：

```cpp
  externalNativeBuild {
      cmake {
          cppFlags "-std=c++11"
      }
  }
  ```
或者：

```cpp
  externalNativeBuild {
      cmake {
          cppFlags ""
      }
  }
  ```
CMake 在编译 C/C++ 代码的时候，回根据上面的两种不同设置来使用C++的两种不同标准，C++11 或者默认标准
android {}后面添加：

  ```cpp
    externalNativeBuild {
        cmake {
            path "CMakeLists.txt"
        }
    }
```
指定 CMake的具体路径为当前项目的根路径,名字为 CMakeLists.txt。
除此之外，还可以增加其他配置属性，这里仅给出两个栗子。

（1）	制定ABI

```cpp
  ndk{
      moduleName "jnitest"//指定生成so库的名字，注意：也可以在CMakeLists中指定
      ldLibs "log", "z", "m"//添加依赖的Log库
      abiFilters "armeabi", "armeabi-v7a", "x86"指定特定的ABI架构
  }

```
（2）指定生成so库的位置

```cpp
//生成so到指定路径下
  sourceSets{
      main{
          jni.srcDirs = []
          jniLibs.srcDirs = ['libs']
      }
  }
```


#### 5. CMakeLists.txt

(1)首先说明一下，CMakeLists.txt中的注释是#。

```cpp
# For more information about using CMake with Android Studio, read the
# documentation: https://d.android.com/studio/projects/add-native-code.html

# Sets the minimum version of CMake required to build the native library.
#设置构建本地库文件所需要的CMake最低版本

cmake_minimum_required(VERSION 3.4.1)

# Creates and names a library, sets it as either STATIC
# or SHARED, and provides the relative paths to its source code.
# You can define multiple libraries, and CMake builds them for you.
# Gradle automatically packages shared libraries with your APK.
set(distribution_DIR ${CMAKE_SOURCE_DIR}/../distribution)
#指定so库存存放的位置
set(CPP_DIR src/main/cpp)
#指定源文件的位置


add_library( # Sets the name of the library.
             #设置library的名字，也是加载时的库的名字，生成的so是libnative-lib.so
             native-lib

             # Sets the library as a shared library.
             #设置library是一个共享库
             SHARED

             # Provides a relative path to your source file(s).
             #指定需要编译的源代码的相对路径
            ${CPP_DIR}/native-lib.cpp )

# Searches for a specified prebuilt library and stores the path as a
# variable. Because CMake includes system libraries in the search path by
# default, you only need to specify the name of the public NDK library
# you want to add. CMake verifies that the library exists before
# completing its build.
#搜索指定预编译库，并将路径存储到一个变量中。
#CMake默认情况下包含搜索路径下的系统库文件，仅需要指定NDK的库文件的名字即可。
#CMake在编译时会确认该库文件是否存在。

find_library( # Sets the name of the path variable.
              #指定库文件路径的变量
              log-lib

              # Specifies the name of the NDK library that
              # you want CMake to locate.
              #指定CMake需要加载的NDK库文件名字
              log )

# Specifies libraries CMake should link to your target library. You
# can link multiple libraries, such as libraries you define in this
# build script, prebuilt third-party libraries, or system libraries.
#指定需要链接到自己生成库文件上的库。
#你可以链接多个库文件，如在这个脚本中自己定义的库，预编译的三方库，或者是系统的库文件

target_link_libraries( # Specifies the target library.
                       #指定链接到的目标库文件
                       native-lib

                       # Links the target library to the log library
                       # included in the NDK.
                       #链接目标库文件到包含在NDK中的Log库上
                       ${log-lib} )
```
 

(2)添加自定义cpp文件或者c文件

这里仅给出添加 cpp 文件例子
定义MyMath.cpp文件

头文件

```cpp

#ifndef JNITESTCMAKE_MYMATH_CPP_H
#define JNITESTCMAKE_MYMATH_CPP_H

int add(int numA,int numB);

#endif //JNITESTCMAKE_MYMATH_CPP_H
```

源文件

```cpp

#include "MyMath.h"

int add(int numA,int numB) {

    return numA + numB;
}

```

很简单，就是计算两个数的和,然后需要修改 CMakeLists 文件，将 MyMath.cpp 添加进去（头文件不需要添加）

```cpp

add_library( # Sets the name of the library.
             #设置library的名字，也是加载时的库的名字，生成的so是libnative-lib.so
             native-lib

             # Sets the library as a shared library.
             #设置library是一个共享库
             SHARED

             # Provides a relative path to your source file(s).
             #指定需要编译的源代码的相对路径
             ${CPP_DIR}/native-lib.cpp ${CPP_DIR}/MyMath.cpp)

```

##### 这里着重讲一下add_library的用法:
```cpp
 add_library(<name> [STATIC | SHARED | MODULE]
            [EXCLUDE_FROM_ALL]
            source1 [source2 ...])
```
>* 第一个参数是 so 库的名字，官方指出名称在工程中是唯一的。
>* STATIC代表静态类型，即编译后为 .a文件;
SHARED 表示动态共享库，即编译后为 .so文件;
MODULE 是一个插件，在运行时完成动态加载

>* 第三个参数，是我们要指定的源文件，这里的类型类似于 java 中的可变类型，可以指定多个文件，如又添加 MyMath.cpp，${CPP_DIR}，表示取出我们自定义路径变量中值 src/main/cpp

完成以上配置后，再设置下布局，添加一个 Button，一个 TextView，cpp 文件中主要完成两个 int 相加工作，通过按钮点击，获取计算结果，并显示在 TextView 中，
下面给出主要代码部分,相信你能够看得懂！

native-lib.cpp文件
```cpp
#include <jni.h>
#include "MyMath.h"

extern "C"
JNIEXPORT jint

JNICALL
Java_com_example_ralf_jnitest_1cmake_MainActivity_resultFromJNI(
        JNIEnv *env,
        jobject /* this */) {

   return add(3,2);
}
```

```java

class MainActivity extends AppCompatActivity{

    static {
        System.loadLibrary("native-lib");
    }

    private native int resultFromJNI();

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        final TextView addTextView = findViewById(R.id.add_text);
        Button button = findViewById(R.id.button);

        button.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                addTextView.setText("相加结果为：" + resultFromJNI());
            }
        });
    }

}

```

----
好了，通过在 CMakeLists文件，我们就可以添加自定义的一些cpp文件，完成想要的功能。比如，一些重要算法的实现，并不像在java层实现，因为很容易被别人反编译，所以需要编译成so库，这样，就大大提高了APK的安全性能！

嗯，就到这里，本篇主要初步介绍了一下在 AS 中通过 CMake 方式编译so库，建立 NDK开发的基本环境

[代码地址](https://github.com/RalfNick/JniPractice/tree/master/NDK03)


剩下的尾巴：AS 中 so 库放在 jniLibs 中会被打包进 APK 中，但是 jniLibs 的设置会存在问题，先这样设置

```cpp

sourceSets {
        main {
            jniLibs.srcDirs = ['../distribution/libs']
            jni.srcDirs = [] //disable automatic ndk-build call
        }
    }
```
出现下面的问题

（1）More than one file was found with OS independent path 'lib/x86/libusb.so'
More than one file was found with OS independent path 'lib/x86/libusb.so'

##### 有知道的小伙伴，麻烦请指点，万分感激！
