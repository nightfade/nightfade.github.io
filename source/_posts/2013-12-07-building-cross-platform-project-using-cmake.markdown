---
categories: 'Tools CMake'
comments: true
date: 2013-12-07 20:01:32 +0800
layout: post
status: public
title: 'Building Cross-platform Project Using CMake'
---

## Why CMake ##
第一次在使用CMake作为项目的构建工具还是在实验室写Structure from Motion系统的时候。一方面是因为要求程序可以在Windows和Linux下都能够编译，另外一个重要原因还是因为实在厌倦了不断的手工去改VS工程文件的各种选项。

在此之前，在有道的做OCR系统的时候也使用过SCons。当时主要考虑的还是因为SCons是使用Python作为脚本，不需要额外的再去学习一门语言。如果项目仅仅需要在Linux下进行编译，那么SCons是一个不错的选择。但是在跨平台方面，和CMake差距还是比较大的。而目前使用CMake的大型项目也有很多，足够证明CMake的可用性：KDE、MySQL、Second Life、OpenCV等。

列举一下CMake的众多优点：

1. 仅仅需要一套Build文本配置文件（CMakeLists.txt），就可以在所有平台进行项目构建。不再需要为每一个平台都维护一份项目文件。
2. 可以根据CMakeLists直接生成Native的项目文件（Visual Studio、XCode、CodeBlocks、Makefile、Eclipse、KDevelop等）。这就意味着我们在享有使用纯文本配置项目构建的便利同时还可以继续使用IDE来编写代码。对于iOS等必须通过XCode对程序进行签名才能完成构建的项目，更是巨大的便利。
4. 自动推断源代码文件之间的依赖关系，并且在大多数平台上支持并行编译。
5. 具有编译时的配置的能力，对于有众多可选组件的项目来说，可以很容易的让用户配置自己所需的组件。
7. 具有众多内置的功能，诸如自动搜索系统中已经安装的依赖库、头文件，自动检测平台特性等。
8. 可以在source tree以外进行项目的构建，始终保持source tree的干净。对于有代码洁癖的coder来说，这一点也同样至关重要。

## Building Project Using CMake ##
以下内容大多参考《Mastering CMake》，以尽可能短的篇幅说明如何编写CMake脚本文件。CMake的安装和执行就不再赘述，可参看官方说明[Installing CMake](http://www.cmake.org/cmake/help/install.html)以及[Running CMake](http://www.cmake.org/cmake/help/runningcmake.html)。

### Basic CMake Usage and Syntax ###
类似于传统的Makefile，CMake使用项目目录下的一个或多个CMakeLists文件来控制构建过程。CMakeLists文件包含了控制构建过程的一系列命令，这些命令具有以下形式：

```cmake
command (args...)
```

command是命令的名称，args是空格分隔的参数列表。CMake对命令名字大小写不敏感。

一个虽然简短但是包含了诸如平台相关源码、搜索依赖库以及链接的CMakeLists样例如下：

```cmake

cmake_minimum_required (2.8)

project(HELLO)

set (HELLO_SRCS Hello.c File2.c File3.c)

if (WIN32)
    set (HELLO_SRCS ${HELLO_SRCS} WinSupport.c)
else ()
    set (HELLO_SRCS ${HELLO_SRCS} UnixSupport.c)
endif ()

# look for the Tcl library
find_library (TCL_LIBRARY
    NAMES tcl tcl84 tcl83 tcl82 tcl80
    PATHS /usr/lib /usr/local/lib
)

if (TCL_LIBRARY)
    target_link_library (Hello ${TCL_LIBRARY})
endif()
```

CMake中各单位的基本关系：

1. 若干源码文件构成一个`target`，一个`target`通常是一个可执行文件或库。
2. 一个directory表示source tree中包含一个CMakeLists文件的目录，并且有一个或多个target与之关联。
3. 每个目录有一个local generator负责生成这个目录下的Makefile或者工程文件。

#### 基本的COMMAND ####

0\. 在CMakeLists.txt首部增加对CMake的最低版本要求：

```cmake
cmake_minimun_required (VERSION 2.6)
````

1\. 定义工程名：

```cmake
project (MyProject [CXX] [C] [JAVA]) # 默认支持以上所有语言
```

定义工程名通常是项目的顶级目录下的CMakeLists.txt的第一条命令。对于每一个`project`命令，CMake会创建一个对应的顶层IDE工程文件，其包含了该CMakeLists种定义的所有Target以及该文件中所有使用`add_subdirectory`指定的子目录中定义的Target。

2\. 加入子目录：

```cmake
add_subdirectory(SOURCE_DIR [BINARY_DIR] [EXCLUDE_FROM_ALL])
```

生成子目录下的模块或程序并放在build的相应子目录下，如果指定了EXCLUDE_FROM_ALL选项，那么该目录下生成的工程将不会出现在顶层的Makefile或工程文件里。这对于将很多sample作为子项目的工程来说是很有意义的。
	
3\. 设置变量：

```cmake
set (VARIABLE_NAME value1 [value2 ...])
```

这里定义的变量是一个由空格分隔的参数列表。set的变量会在当前文件以及add_subdirectory所有子目录下的CMakeLists中生效。但是任何子域下set的变量不会影响父域。
	
4\. 输出提示信息：

```cmake
message ([SEND_ERROR | STATUS | FATAL_ERROR] "message")
```

5\. 定义可执行对象target：

```cmake
add_executable (MyExe ${SRC_FILES})
```

6\. 定义静态库、动态库target：

```cmake
add_library (MyLib [SHARED | STATIC | MODULE] "${SRC_FILES}")
```
	
7\. 增加INCLUDE目录：

```cmake
include_directories ("${PROJECT_BINARY_DIR}")
```

8\. 增加link的库目录：

```cmake
link_directories ("${PROJECT_BINARY_DIR}")
```

9\. 为target增加链接对象:

```cmake
target_link_libraries (MyExe "${LIB_NAMES}")
```

10\. 增加项目依赖：

```cmake
add_dependencies (target-name depend-target1 depend-target2)
```

定义target依赖其他target，保证在编译本target之前，其他target已经被构建。

#### 内置变量 ####

CMAKE提供了一些内置变量，通过读取或设置这些变量的值可以控制编译：

1\. 指定二进制目标的保存位置：

```cmake
# 指定编译的可执行文件输出到项目build目录下的bin文件夹
set (EXECUTABLE_OUTPUT_PATH ${PROJECT_BINARY_DIR}/bin)

# 指定编译的库文件输出到项目build目录下的lib文件夹
set (LIBRARY_OUTPUT_PATH ${PROJECT_BINARY_DIR}/lib)
```

无论是否在ADD_SUBDIRECTORY中制定了编译输出目录，都可以指定最终二进制的位置。

2\. CMake中系统信息的内置变量：

```cmake
CMAKE_SYSTEM		# 系统名称，比如Linux-2.6.22

CMAKE_SYSTEM_NAME	# 不包含版本的系统名，比如Linux

APPLE				# Mac OS X上返回TRUE

UNIX				# 所有的类UNIX平台上为TRUE，包括OS X和cygwin

WIN32				# 所有win32平台上为TRUE，包括cygwin

CYGWIN				# cygwin下返回TRUE

MSVC				# 使用 Microsoft Visual C 时返回TRUE
```

3\. 其他常用变量

```cmake
CMAKE_BINARY_DIR & PROJECT_BINARY_DIR	# 执行cmake的目录。

CMAKE_SOURCE_DIR $ PROJECT_SOURCE_DIR	# 工程顶层目录

CMAKE_CURRENT_SOURCE_DIR				# 当前处理的CMakeLists.txt的所在路径

CMAKE_CURRENT_BINARY_DIR				# 当前处理的CMakeLists.txt的输出路径
```

#### 流程控制 ####

CMake提供了三种流程控制命令：

1\. 条件指令（if）

```cmake
if (FOO)
	# do something
else ()
	# do something
endif ()
```

```cmake
if (MSVC80)
	# do something
elseif (MSVC90)
	# do something
elseif (APPLE)
	# do something
endif ()
```

其中`if`可支持的形式有：
```cmake
if (variable)

if (NOT variable)

if (variable1 AND variable2)

if (variable1 OR variable2)

if (EXISTS file-name)

if (EXISTS directory-name)

if (IS_DIRECTORY name)

if (IS_ABSOLUTE name)

if (name1 IS_NEWER_THAN name2)

EQUAL, LESS, GREATER # 数值比较
STRLESS, STREQUAL, STRGEATER # ZIFUCHUAN BIJIAO 
```


2\. 循环结构（foreach & while）

```cmake
foreach (tfile
		file1
		file2
		file3)
	# do something with tfile
endforeach (tfile)
```


```cmake
while (${var} LESS 3600)
	# do something
endwhile ()
```

3\. 过程定义 (macro & function)

```cmake
function (FunctionName param)
	# do something with ${param}
endfunction()

FunctionName(123)
```

macro和function的使用方法一样，但是macro不涉及变量作用于的变化。

```cmake
macro (assert TEST COMMENT)
	# do something
endmacro (assert)

FunctionName(123)
```

### 使用Modules ###

Module就是放在一个单独文件中的一段CMake命令。可以通过`include`命令来使用：

```cmake
include (FindTCL)

target_link_library (FOO ${TCL_LIBRARY})
```

CMake内置了大量的Module，可以很方便的定位已安装的库或头文件或查看系统属性。具体的Module项目以及用法可以查看cmake安装目录下的Module目录。

Find<XX>.cmake模块通常的规则是：

	<XX>_INCLUDE_DIRS: 头文件目录
	
	<XX>_LIBRARIES: LINK目录
	
	<XX>_DEFINITIONS: 编译该库需要使用的预处理define
	
	<XX>_FOUND: 是否找到


### 通过`find_path`，`find_library`命令自己指定目录进行搜索 ###

```cmake
find_library (TIFF_LIBRARY
			  NAMES tiff tiff2
			  PATHS /usr/local/lib /usr/lib)


find_path (TIFF_INCLUDES tiff.h
		   /usr/local/include
		   /usr/include)

include_directories (${TIFF_INCLUDES})

add_executable (mytiff mytiff.c)

target_link_libraries (mytiff ${TIFF_LIBRARY})
```


### 为编译器传递编译选项 ###

1\. 使用`add_definitions`为编译器增加预处理define：

```cmake
option (DEBUG_BUILD "Build with extra debug message")

if (DEBUG_BUILD)
	add_definitions (-DDEBUG_BUILD)
endif ()
```

2\. 为单个的目录，target，或源代码指定编译选项：

```cmake
set_property (
	DIRECTORY	# 当前目录及子目录
	PROPERTY COMPILE_DEFINITIONS A AV=1
)

set_property (
	TARGET mylib
	PROPERTY COMPILE_DEFINITONS B BV=2
)

set_property (
	SOURCE src1.c
	PROPERTY COMPILE_DEFINITIONS C CV=3
)
```


### 文件操作 ###

```cmake
file (WRITE filename "message to write")

file (APPEND filename "message to write")

file (READ filename variable)

file (GLOB variable [RELATIVE PATH] [GLOBBING EXPRESSIONS])

file (GLOB_RECURSE variable [RELATIVE PATH] [GLOBBING EXPRESSIONS])

file (REMOVE [DIRECTORY])

file (REMOVE_RECURSE [DIRECTORY])

file (MAKE_DIRECTORY [DIRECTORY])
```

### 其他 ###

至此，CMAKE的基本用法基本阐述完毕。然而，这次重新捡起CMake的主要目的还是在于希望用CMake进行cocos2d-x的跨平台构建。这就涉及到如何使用CMake进行交叉编译。这部分内容我会在下一篇博文中进行阐述。